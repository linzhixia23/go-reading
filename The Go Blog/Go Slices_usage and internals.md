原文： https://blog.golang.org/go-slices-usage-and-internals

<br>

#### 介绍
Go的slice类型提供了一种方便且高效的方式来使用连续的数据类型。
在其他语言里，slice类似于array，但有一些不寻常的属性。
本文就来看看slice是什么，以及它如何工作。

#### Arrays 

slice类型是基于array类型做的抽象，所以了解slice之前先了解array。

array类型明确定义了长度和元素类型。例如，类型[4]int表示一个长度为4的数组。
array的大小是固定的；它的长度是它的类型的一部分（[4]int和[5]int是不一样、且不兼容的类型）。
array可以用通常的方式索引，因而表达式s[n]表示访问从0开始的第n个元素。

```go
var a [4]int
a[0] = 1
i := a[0]
// i == 1
```

array不需要显式地初始化；它们会被赋为0值。

```go
// a[2] == 0, the zero value of the int type
```

在内存里，[4]int是4个整形顺序地平铺：

![image](https://github.com/linzhixia23/go-reading/blob/master/The%20Go%20Blog/pic/go-slices-usage-and-internals_slice-array_laid_out.png)

Go的array是值（value）。一个array变量表示整个array，它并不是指向array第一个元素的指针（这点跟C是不一样的）。
这意味这，当你赋值或者传递一个指针值，你会复制它的内容。（为了避免复制，你可以传递一个指向array的指针，这时它就是一个指针，而不是array）。
可以这么理解array，它是一种用索引而不是名字来访问的结构：一个固定大小的组合值。

一个array可以被定义为：

```go
b := [2]string{"Penn", "Teller"}
```

或者你可以让编译器帮你计算元素个数

```go
b := [...]string{"Penn", "Teller"}
```

上面两个case中，b的类型都是[2]string。

#### Slice 

array有它们的位置，但它们有一些不灵活，所以你在go代码里很少看到它们。
slice则随处可见。它们是基于array的，但更加方便，而且功能强大。

slice的定义为[]T，而T是slice的严肃类型。
跟array类型不同，slice类型并不会指定长度。

slice的定义跟array差不多，只是你不需要指定元素的个数：

```go
letters := []string{"a", "b", "c", "d"}
```

slice也可以用内置函数make来创建，它的签名如下：

```go
func make([]T, len, cap) []T
```

T表示slice要创建的元素类型。make函数需要类型、长度和可选的容量（capacity）
调用的时候，make分配一个array，并返回一个slice引用该array。

```go
var s []byte
s = make([]byte, 5, 5)
// s == []byte{0, 0, 0, 0, 0}
```

当容量的参数缺省时，它会默认为等于指定的length。以下是一个更简洁的版本：

```go
s := make([]byte, 5)
```

slice的长度和容量大小，可以使用内置的函数len和cap。

```go
len(s) == 5
cap(s) == 5
```

下面两节会讨论长度和容量的关系。

slice的0值是nil，也就是，len和cap对一个nil的slice都会返回0。

slice可以通过"切片"一个现有的slice或者array来构建。切片是通过指定一个half-open的范围来指定两个下标。
例如，b[1:4]表示选择元素1到3。

```go
b := []byte{'g', 'o', 'l', 'a', 'n', 'g'}
// b[1:4] == []byte{'o', 'l', 'a'}, sharing the same storage as b
```

slice的开始和结束的下标是可选的；默认值是0和slice的长度。

```go
// b[:2] == []byte{'g', 'o'}
// b[2:] == []byte{'l', 'a', 'n', 'g'}
// b[:] == b
```

同样可以用给定的array来构建slice。

```go
x := [3]string{"Лайка", "Белка", "Стрелка"}
s := x[:] // a slice referencing the storage of x
```

#### slice内核

sliece是数组段的描述符。
slice包含了指向array的指针，segment的长度和它的容量。

![image](https://github.com/linzhixia23/go-reading/blob/master/The%20Go%20Blog/pic/go-slices-usage-and-internals_slice-struct.png)

通过make([]byte,5)来创建的变量s，则是这样的：

![image](https://github.com/linzhixia23/go-reading/blob/master/The%20Go%20Blog/pic/go-slices-usage-and-internals_slice-struct_slice_make.png)

length表示slice的元素个数。capacity是底层array的元素个数。
length和capacity的区别会在下一节阐释。

如果我们对s切片，可以看到slice的数据接口和它们底层array的关系：

```go
s = s[2:4]
```

![image](https://github.com/linzhixia23/go-reading/blob/master/The%20Go%20Blog/pic/go-slices-usage-and-internals_slice-struct_slice_make_2.png)

切片并不会复制slice的数据。它创建一个新的slice值，并把指针指向原始的array。
这使得slice操作跟使用array下标一样方便。
然而，修改re-slice的元素则会修改原始的slice。

```go
d := []byte{'r', 'o', 'a', 'd'}
e := d[2:] 
// e == []byte{'a', 'd'}
e[1] = 'm'
// e == []byte{'a', 'm'}
// d == []byte{'r', 'o', 'a', 'm'}
```

前面的例子，我们把s切片为比它的容量小的长度。我们可以切片来增大它的容量：

```go
s = s[:cap(s)]
```

![image](https://github.com/linzhixia23/go-reading/blob/master/The%20Go%20Blog/pic/go-slices-usage-and-internals_slice-struct_slice_make_3.png)

slice不能增长得超过它的容量。如果尝试这么做的话，会导致运行时错误，正如我们的下标索引超过了slice或者array的界限。

#### 增长的slice（copy和append函数）

要增加slice的容量，必须要创建一个新的更大的slice，并把原始slice的内容复制过去。
这就是其他语言里的动态数组的实现。下面的例子中，会创建一个新的slice，t来扩大s的容量，并把s的数据复制到t，然后把t的slice值

```go
t := make([]byte, len(s), (cap(s)+1)*2) // +1 in case cap(s) == 0
for i := range s {
        t[i] = s[i]
}
s = t
```

for循环的操作用内置函数copy会更简单。正如它的名字一样，copy会把源数据的slice复制到目标的slice。
它会返回复制的元素个数。

```go
func copy(dst, src []T) int
```

copy函数支持复制两个不同长度的slice（它只会复制最少的元素个数的slice）。
此外，copy可以处理源和目标共享相同底层array的slice，它很很好处理overlapping的问题。

通过copy，我们可以把上述代码简化为：

```go
t := make([]byte, len(s), (cap(s)+1)*2)
copy(t, s)
s = t
```

一个常用的操作是在slice的后面加数据。这个函数会在byte类型的slice后面加入byte元素，如果有必要的话，它会扩大slice，并返回更新的slice值。

```go
func AppendByte(slice []byte, data ...byte) []byte {
    m := len(slice)
    n := m + len(data)
    if n > cap(slice) { // if necessary, reallocate
        // allocate double what's needed, for future growth.
        newSlice := make([]byte, (n+1)*2)
        copy(newSlice, slice)
        slice = newSlice
    }
    slice = slice[0:n]
    copy(slice[m:n], data)
    return slice
}
```

可以这样使用AppendByte：

```go
p := []byte{2, 3, 5}
p = AppendByte(p, 7, 11, 13)
// p == []byte{2, 3, 5, 7, 11, 13}
```

像AppendByte这样的函数是很有用的，它处理了slice增长的各种情况。
根据程序的特征，可以任意分配更大或更小的chunk，甚至可以刚刚好。

但大多数情况并不需要这样的完全控制，所以go提供了一个内置的append函数，可用于大多数情况。
它的签名是：

```go
func append(s []T, x ...T) []T
```

append函数把x放到s的后面，并根据需要增加slice的容量。

```go
a := make([]int, 1)
// a == []int{0}
a = append(a, 1, 2, 3)
// a == []int{0, 1, 2, 3}
```

要append一个slice到另一个上，可以使用...

```go
a := []string{"John", "Paul"}
b := []string{"George", "Ringo", "Pete"}
a = append(a, b...) // equivalent to "append(a, b[0], b[1], b[2])"
// a == []string{"John", "Paul", "George", "Ringo", "Pete"}
```

由于slice的0值（nil）表现为一个长度为0的slice，你可以声明一个slice变量，把它append到一个循环中：

```go
// Filter returns a new slice holding only
// the elements of s that satisfy fn()
func Filter(s []int, fn func(int) bool) []int {
    var p []int // == nil
    for _, v := range s {
        if fn(v) {
            p = append(p, v)
        }
    }
    return p
}
```

#### "gocha"

正如前面提到的，重新切片slice并不会复制底层的array。
array会一直在内容，直到它没有被引用（refer）。
有时会导致程序在内容保留了大量数据，而实际需要的只有一点。

例如，FindDigits这个函数load一个文件到内容，并搜索第一组连续的整数，并把它的值做为新的slice返回。

```go
var digitRegexp = regexp.MustCompile("[0-9]+")

func FindDigits(filename string) []byte {
    b, _ := ioutil.ReadFile(filename)
    return digitRegexp.Find(b)
}
```

这个代码里，返回的[]byte会指向一个包含整个文件的array。
因为slice引用了原始的array，只要slice保留，那么gc就不会释放array；
虽然只用到一点byte，却把整个文件放到了内存。

要避免这个问题，可以在返回之前，只复制相关的数据。

```go
func CopyDigits(filename string) []byte {
    b, _ := ioutil.ReadFile(filename)
    b = digitRegexp.Find(b)
    c := make([]byte, len(b))
    copy(c, b)
    return c
}
```

更简洁的做法是用append来实现。这就留给读者练习吧。



