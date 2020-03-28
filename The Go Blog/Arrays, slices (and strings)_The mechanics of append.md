原文：https://blog.golang.org/slices 

<br>

#### 介绍

过程式编程语言的最常见的特征是array的概念。
array看起来很简单，但如果要把它加入到一门语言中，则需要回答一些问题，例如：

* 固定size还是可变的size？

* size是否是类型的一部分？ 

* 多维数据看起来是什么样的？ 

* 空的array是否有意义？ 

这些问题的回答，影响到array是否只是语言的一个特征，还是它设计的核心部分。

在go语言早期的设计中，它花了一年多的时间，直到设计觉得差不多了，才来决定这些问题的答案。
其中关键的步骤是引入了slice，它是基于固定大小的array来构建的，但它提供了灵活的、可扩展的结构。
直到今天，go的新程序员对slice的工作方式还似懂非懂，可能是因为其他语言的经验干扰了他们的思路。

在这篇博客中，我们尝试讲清楚这些困惑。
我们通过解释内置函数append如果工作，以及它为什么这么工作，来讲清楚这个问题。

#### Array 

array是go语言的重要基石，但正如建筑的基础一样，它们通常隐藏于其他可见的组件中。
在我们讨论slice的有趣而强大的设计思想时，我们需要简单了解array。

array在go程序里很少看到，因为它把大小做为了它类型的一部分，这样限制了它的表达力。

像这样的声明，定义了变量buffer，它有256个byte。

```go
var buffer [256]byte
```

类型buffer包括了它的大小，以及[256]byte。一个512字节的byte则是另一种byte。

数据和array的联系就只是：一串元素。我们的buffer坐在内存里是这样的：

```go
buffer: byte byte byte ... 256 times ... byte byte byte
```

也就是说，变量只有256个byte，没有其它的。我们可以用熟悉的下标语法来访问它，例如：buffer[0], buffer[1]。
如果超过了下标的范围，程序会崩溃。

有一个内置函数len可以返回array或者slice的元素个数。

array有它们的应用地方，例如可以表达矩阵实例，不过在go语言中，它们的作用是为slice提供存储空间。

#### Slice：slice头部

要想用好slice，需要理解它们到底做了什么，以及怎么做的。

slice是一个数据结构，它描述了array的一段连续的存储空间，以及slice自身的变量。
slice不是array。它只是描述了一个array。


从前一节里面，给定buffer变量，我们可以通过切片一个array的100-150下标来创建slice。

```go
var slice []byte = buffer[100:150]
```

在代码段里，变量slice类型是[]byte，它从array进行初始化（即buffer），切片元素100-150。
更常用的语法是省略type。

```go
var slice = buffer[100:150]
```

在函数里，我们可以用短声明表达

```go
slice := buffer[100:150]
```

这里的slice变量到底是什么呢？
虽然不是准确的定义，但目前位置可以认为slice是一个有2个元素的数据结构：长度值和指向array的指针。
你可以认为它们底层是这样创建的：

```go
type sliceHeader struct {
    Length        int
    ZerothElement *byte
}

slice := sliceHeader{
    Length:        50,
    ZerothElement: &buffer[100],
}
```

当然，这只是一个举例。
虽然这个代码段里的sliceHeader结构对程序员不可见，元素的指针也取决于元素的类型，但它提供了这个机制的大致思路。

目前为止，我们对array使用了切片操作，我们其实还可以对slice用切片操作，就像这样：

```go
slice2 := slice[5:10]
```

正如前面提到的，这个操作创建了一个新的slice，它是原来slice的5-9，这意味着，是原始array的105-109位置。
因而，slice2变量的sliceHeader结构如下：

```go
slice2 := sliceHeader{
    Length:        5,
    ZerothElement: &buffer[105],
}
```

注意，这个header仍然是指向底层的array，也就是存储buffer变量的array。

我们可以切片一个slice，并存储返回结果。执行下面操作之后，slice就跟slice2一样了。

```go
slice = slice[5:10]
```

你可以看到，重新切片很常见，例如剪枝一个slice。以下操作去掉了slice的第一个和最后一个元素：

```go
slice = slice[1:len(slice)-1]
```


你可能会经常听到go程序员讨论slice header，因为它是真实存储在slice变量中的。
例如，你调用一个函数，它把slice做为参数，例如bytes.IndexRune，传递给函数的，其实就是header。
在这个调用里，

```go
slashPos := bytes.IndexRune(slice, '/')
```

slice的传递给IndexRune函数的实参，实际上就是"slice header"。

有更多的数据的slice header里，我们接下来会讨论。在这里，先看看slice header的存在对你使用slice意味着什么。

#### 传递slice到函数

一个需要理解的重要概念是，尽管slice包含了一个指针，但它实际上是一个值。
在它底层，它实际上是一个结构，结构的值是一个指针和长度。它并不是一个指向结构的指针。

这很重要。

在我们调用IndexRune时，它实际上是传递的slice header的拷贝。
这样会导致很重要的结果。

例如这个简单的函数：

```go
func AddOneToEachElement(slice []byte) {
    for i := range slice {
        slice[i]++
    }
}
```

它做的跟它名字说的一样，编译所有元素，并加1。

运行一下：

```go
func main() {
    slice := buffer[10:20]
    for i := 0; i < len(slice); i++ {
        slice[i] = byte(i)
    }
    fmt.Println("before", slice)
    AddOneToEachElement(slice)
    fmt.Println("after", slice)
}
```

尽管slice header是通过值传递的，但header包括了指向array的指针，因此，原始的slice和header的copy其实描述的是同样的array。
因此，当函数返回时，修改的元素会被原来的slice变量看见。

函数的实参是一个copy，正如这个例子所见：

```go
func SubtractOneFromLength(slice []byte) []byte {
    slice = slice[0 : len(slice)-1]
    return slice
}

func main() {
    fmt.Println("Before: len(slice) =", len(slice))
    newSlice := SubtractOneFromLength(slice)
    fmt.Println("After:  len(slice) =", len(slice))
    fmt.Println("After:  len(newSlice) =", len(newSlice))
}
```

这里，我们可以看到，slice参数的content可以被一个函数修改，但它的header不能。
slice变量里存储的length并没有因为调用函数而被修改，因为函数调用传递的是slice header的拷贝，而不是原始的slice header。
因此，我们如果需要写一个函数来修改header，我们必须把它做为一个结果参数返回，正如我们这里做的这样。
slice变量不会改变，但是返回的值有新的length，而它存储在newSlice。

#### 指向slice的指针：receiver方法

另一个在函数里修改slice header的方式是，传递它的指针。
它可以是之前例子的变体：

```go
func PtrSubtractOneFromLength(slicePtr *[]byte) {
    slice := *slicePtr
    *slicePtr = slice[0 : len(slice)-1]
}

func main() {
    fmt.Println("Before: len(slice) =", len(slice))
    PtrSubtractOneFromLength(&slice)
    fmt.Println("After:  len(slice) =", len(slice))
}
```

这个代码看起来很笨拙，尤其是引入多一层的定向，但这是你看到的指向slice指针的常见例子。
标准是使用指针receiver来修改一个slice。

假设，我们要截断slice的最后一个斜杠后的内容。我们可以这样写：

```go
type path []byte

func (p *path) TruncateAtFinalSlash() {
    i := bytes.LastIndex(*p, []byte("/"))
    if i >= 0 {
        *p = (*p)[0:i]
    }
}

func main() {
    pathName := path("/usr/bin/tso") // Conversion from string to path.
    pathName.TruncateAtFinalSlash()
    fmt.Printf("%s\n", pathName)
}
```

另一方面，如果我们想把path里都变成大写字母，则方法可以是一个值，因为值的receiver仍然指向相同的底层指针。

#### 容量

看下面这个函数，它把实参的slice扩增一个元素：

```go
func Extend(slice []int, element int) []int {
    n := len(slice)
    slice = slice[0 : n+1]
    slice[n] = element
    return slice
}
```

(为什么它需要返回修改后的slice)。现在运行一下：

```go
func main() {
    var iBuffer [10]int
    slice := iBuffer[0:0]
    for i := 0; i < 20; i++ {
        slice = Extend(slice, i)
        fmt.Println(slice)
    }
}
```

slice似乎并没有如预期增长。

下面需要引入slice header的第三个部分：capacity。
除了数组指针和长度，slice header还存储了它的容量。

```go
type sliceHeader struct {
    Length        int
    Capacity      int
    ZerothElement *byte
}
```

capacity属性记录了底层数组实际的空间；它也是length能达到的最大值。
如果slice达到了它的容量，会导致array数组越界，进而引发panic。

在我们的例子中，slice通过

```go
slice := iBuffer[0:0]
```

创建之后，它是这样的：

```go
slice := sliceHeader{
    Length:        0,
    Capacity:      10,
    ZerothElement: &iBuffer[0],
}
```

capacity等于底层array的长度。如果你想查询slice的容量，可以使用内置函数：

```go
if cap(slice) == len(slice) {
    fmt.Println("slice is full!")
}
```

#### Make

如果我们把slice增长超过它的容量呢？不行！根据定义，容量是增长的极限。
但是你可以分配一个新的array，复制数据，把新的slice修改成你想要的部分。

先从分配开始。我们可以用内置函数new来分配一个更大的array，再切片结果，但其实用内置函数make会更容易。
它分配了一个新的array，并创建了一个slice header去描述它。
make函数使用3个参数，slice的类型，初始长度和容量。
这个调用创建了一个长度为10的slice，而且有5个额外的空间。

```go
slice := make([]int, 10, 15)
fmt.Printf("len: %d, cap: %d\n", len(slice), cap(slice))
```

这段代码的容量是slice的两倍，但length是一样的：

```go
	slice := make([]int, 10, 15)
    fmt.Printf("len: %d, cap: %d\n", len(slice), cap(slice))
    newSlice := make([]int, len(slice), 2*cap(slice))
    for i := range slice {
        newSlice[i] = slice[i]
    }
    slice = newSlice
    fmt.Printf("len: %d, cap: %d\n", len(slice), cap(slice))
```

当创建slice的时候，通常它的length和capacity是一样的。
内置函数为此提供了便利，因而你不需要把它们设置成相同的值。
在这个case中，gopher的长度和容量是一样的：

```go
gophers := make([]Gopher, 10)
```

#### copy 

前面一节中，当我们把slice的容量翻倍时，我们写一个循环来搬运数据。
go有一个内置函数copy，可以把这变得简单。
它的参数是两个slice，会把右边的数据拷贝到左边。
下面是使用copy的例子：

```go
	newSlice := make([]int, len(slice), 2*cap(slice))
    copy(newSlice, slice)
```

copy函数很聪明。它会关注两个参数的长度，copy能复制的。换句话说，它复制的是两个slice的length的最小值。
这样可以省去一些繁琐的工作。同时，copy返回一个整数，说明它复制的元素个数，尽管这不总是值得检验。

copy函数总是能得到正确的结果，即便是源和目标存在overlap。
这也意味着，它可以在一个slice里面移动元素的位置。
以下是如何使用copy插入一个值在slice的中间。

```go
// Insert inserts the value into the slice at the specified index,
// which must be in range.
// The slice must have room for the new element.
func Insert(slice []int, index, value int) []int {
    // Grow the slice by one element.
    slice = slice[0 : len(slice)+1]
    // Use copy to move the upper part of the slice out of the way and open a hole.
    copy(slice[index+1:], slice[index:])
    // Store the new value.
    slice[index] = value
    // Return the result.
    return slice
}
```

这个函数里，有两个值得注意的地方。首先，它返回更新后的slice，因为它的长度改变了。
其次，它使用了捷径（shorthand）。

```go
slice[i:]
```

这等价于

```go
slice[i:len(slice)]
```

还有一个我们没有用到的trick是，我们可以缺省第一个值；它默认是0。因此，

```go
slice[:]
```

等于slice本身。

现在，我们来运行Insert函数。

```go
	slice := make([]int, 10, 20) // Note capacity > length: room to add element.
    for i := range slice {
        slice[i] = i
    }
    fmt.Println(slice)
    slice = Insert(slice, 5, 99)
    fmt.Println(slice)
```

#### Append: 一个例子

在前几节里，我们写了一个函数，把slice增加一个元素。
它是有bug的，因为如果slice的容量太小，则函数会crash。
现在，我们来修复它，写一个健壮的程序。

```go
func Extend(slice []int, element int) []int {
    n := len(slice)
    if n == cap(slice) {
        // Slice is full; must grow.
        // We double its size and add 1, so if the size is zero we still grow.
        newSlice := make([]int, len(slice), 2*len(slice)+1)
        copy(newSlice, slice)
        slice = newSlice
    }
    slice = slice[0 : n+1]
    slice[n] = element
    return slice
}
```

在这个case中，返回slice是很重要的，因为当重新分配时，结果的slice描述了一个完全不同的slice。
以下代码阐述这个问题：

```go
    slice := make([]int, 0, 5)
    for i := 0; i < 10; i++ {
        slice = Extend(slice, i)
        fmt.Printf("len=%d cap=%d slice=%v\n", len(slice), cap(slice), slice)
        fmt.Println("address of 0th element:", &slice[0])
    }
```

注意到当最初分配的5个空间满时，它重新分配了。
当新的array分配时，容量和第0个元素的地址都发生了改变。

把这个健壮的Extend函数做为指引，我们可以写一个更好的函数。
要做到这个，我们使用go的能力把一个list的参数变成slice。 
也就是，我们使用go的variadic function facility。

下面调用Append函数。第一个版本，我们只需要重复调用Extend，因而variadic function的机制很清晰。
Append函数的签名是：

```go
func Append(slice []int, items ...int) []int
```

这个函数的意思是说，Append需要一个参数，slice，以及0或更多个int参数。

```go
// Append appends the items to the slice.
// First version: just loop calling Extend.
func Append(slice []int, items ...int) []int {
    for _, item := range items {
        slice = Extend(slice, item)
    }
    return slice
}
```

注意，for循环迭代了items参数的元素，也即是类型[]int。
同样注意到，只要了空占位符_来丢弃循环的下标，因为我们不需要用到它。

试一试这个：

```go
    slice := []int{0, 1, 2, 3, 4}
    fmt.Println(slice)
    slice = Append(slice, 5, 6, 7, 8)
    fmt.Println(slice)
```

另一个我们用到的技术是，我们使用了composite literal，它定义了slice里的元素：

```go
    slice := []int{0, 1, 2, 3, 4}
```

Append还有一个有意思的地方。我们不仅可以append元素，还可以append整个slice，只要使用...符号。

```go
	slice1 := []int{0, 1, 2, 3, 4}
    slice2 := []int{55, 66, 77}
    fmt.Println(slice1)
    slice1 = Append(slice1, slice2...) // The '...' is essential!
    fmt.Println(slice1)
```

当然，我们可以在Append里分配空间不超过一次，在Extend的内部实现：

```go
// Append appends the elements to the slice.
// Efficient version.
func Append(slice []int, elements ...int) []int {
    n := len(slice)
    total := len(slice) + len(elements)
    if total > cap(slice) {
        // Reallocate. Grow to 1.5 times the new size, so we can still grow.
        newSize := total*3/2 + 1
        newSlice := make([]int, total, newSize)
        copy(newSlice, slice)
        slice = newSlice
    }
    slice = slice[:total]
    copy(slice[n:], elements)
    return slice
}
```

注意，我们使用copy函数两次，第一次是把slice数据copy到新分配的内存中，第二次是把append的内容加到旧数据的后面。

运行一下；它的行为跟之前一样：

```go
    slice1 := []int{0, 1, 2, 3, 4}
    slice2 := []int{55, 66, 77}
    fmt.Println(slice1)
    slice1 = Append(slice1, slice2...) // The '...' is essential!
    fmt.Println(slice1)
```

#### Append:内置函数

现在，我们可以讨论内置函数append的设计动机了。
它跟我们的Append函数做一样的事情，效率也一样，只是它适用于任何数据类型。

GO语言的一个不好的地方是，它任何范型都必须由runtime提供。
或许它有一天会改变，但目前，为了使用slice更方便，go提供了内置范型函数append。
它跟我们的int slice版本做一样的事情，只是适用于任何slice类型。

记住，由于slice header总是会在调用append时发生改变，所以，你需要在调用后保存返回的slice。

关于更多的append的用法，可以看["Slice Tricks" Wiki page](https://github.com/golang/go/wiki/SliceTricks)。

#### nil

题外话，根据我们刚学到的知识，我们可以看一个nil的slice如何表达。
自然地，它的slice header是0值：

```go
sliceHeader{
    Length:        0,
    Capacity:      0,
    ZerothElement: nil,
}
```

关键是，指向元素的指针也是nil。

如果一个slice是这样创建的：

```go
array[0:0]
```

则它的长度是0（甚至容量也是0），但它的指针并不是nil，所以，它并不是一个nil的slice。

需要清楚的是，empty slice可以增长，但是nil slice没有array来存放value，所以它不能放哪怕一个元素。

#### Strings

下面来看go里面strings的例子。

strings事实上非常简单，它们是只读的byte的slice，还有一些额外的语法对语言做支持。

由于它们只读，所以不需要capacity（因为不会增加它），但大多数时候，你可以把它当作只读的byte的slice。

我们可以用下标来访问单个byte：

```go
slash := "/usr/ken"[0] // yields the byte value '/'.
```

我们可以把一个string切片成子字符串。

```go
usr := "/usr/ken"[0:4] // yields the string "/usr"
```

现在应该很清楚，当我们切片一个string时，会发生什么了。

我们可以把一个byte的slice转成slice

```go
str := string(slice)
```

当然，反过来也是可以的：

```go
slice := []byte(usr)
```

一个string的底层array是看不到的；只有通过string才能访问到它的内容。
这也就意味着，我们要么做这样的转换，要么使用copy。
当然，go帮你做了这些。做了转换之后，修改byte slice底层的array并不会影响对应的string。

像这样的slice-like的设计的一个重要的结果是，创建一个子字符串会很有效率。
所需要做的就是创建一个string header。
由于string是只读的，因而原来的string和slice后的string可以安全地共用相同的array。

注意：最早string的实现总是会分配空间，但是当slice加入到语言之后，它提供了一个高效的string处理方式。
一些benchmark可以看到巨大的速度提升。

[这篇文章](https://blog.golang.org/strings)更加深入地讨论了string的问题。

#### 结论

要理解slice是怎么工作的，最好是知道它们是怎样实现的。
通过slice header这个数据结构把slice变量联系起来，它的header描述了独立分配的array信息。
当我们传递slice时，会复制header的信息，而指向的array的指针是不变的。

当你理解它的工作方式，slice不仅很容易使用，而且高效且易于表达，尤其是在copy和append这两个内置函数的帮助下。

#### 延伸阅读

正如前面提到的，["Slice Tricks" Wiki](https://golang.org/wiki/SliceTricks)提供了很多用例。
[Go Slice这篇博客](https://blog.golang.org/go-slices-usage-and-internals)通过清晰的图表达了slice的内存布局。
Russ Cox的[Go Data Structure](https://research.swtch.com/godata)讨论了slice，以及go的一些内部数据结构。

有很多材料，但最好的学习方式是使用它们。 

