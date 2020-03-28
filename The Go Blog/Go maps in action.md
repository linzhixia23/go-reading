原文：https://blog.golang.org/go-maps-in-action 

Go maps in action 

#### 介绍 

计算机科学中，最常用的一个数据结构就是哈希表。
很多哈希表的实现有不同的属性，但是他们提供了快速的查询，增加和删除。
Go提供了一个内置的map类型，他们实现了哈希表。

#### 声明和初始化 

Go的map类型像这样的：

```go
map[KeyType]ValueType
```

KeyType可以是任何可以比较的类型，而ValueType可以是任何类型，甚至能是另一个map。

这里的m是一个string到int的map 

```go
var m map[string]int
```

map是引用类型，就跟指针和切片一样，因此，上面m的值其实是nil；它并没有指向一个初始化了的map。
一个nil的map在读的时候，就跟空的map一样；但是如果你尝试写入的话，就会导致panic；所以不要这么做。
要初始化map，应该用内置的make函数：

```go
m = make(map[string]int)
```

make函数分配空间，初始化一个hash map结构，并返回一个指向它的map值。
它的数据结构由runtime的实现细节来决定，而跟语言本身无关。
这篇文章我们主要关注map的使用，而不是它的实现。

#### map的使用

go提供了一种常见的使用map的方式。 
这个声明把key值"routue"映射到值66。

```go
m["route"] = 66
```

这个声明检索key值"route"的映射值，并把它分配给变量i：

```go
i := m["route"]
```

如果请求的key不存在，我们会得到value的类型的零值。
在这个case中，value的类型是int，因而它的零值是0：

```go
j := m["root"]
// j == 0
```

内置函数len返回map的元素个数：

```go
n := len(m)
```

内置函数delete从map中删除一个entry：

```go
delete(m, "route")
```

delete函数不会返回任何值，因此如果key不存在的话，它不会做任何事情。

tow-value的分配可以验证key是否存在：

```go
i, ok := m["route"]
```

在这个语句里，第一个值（i）会返回key值"route"的value。
如果key不存在的话，i是value的类型的零值。
第二个值（ok）是一个bool变量，如果key存在，它会返回true，反之则返回false。

如果只是测试key值，而不需要返回，则可以用下划线：

```go
_, ok := m["route"]
```

如果要遍历map，则可以使用range关键字：

```go
for key, value := range m {
    fmt.Println("Key:", key, "Value:", value)
}
```

要用一些数据初始化map，则使用map字面量：

```go
commits := map[string]int{
    "rsc": 3711,
    "r":   2138,
    "gri": 1908,
    "adg": 912,
}
```

相同的语法可以用户初始化一个空的map，在语法上，它跟make函数是一样的：

```go
m = map[string]int{}
```

### 利用零值

当key不存在时返回value的零值这个特征很有用。

例如，一个value是bool值的map可以当作一个set数据结果来使用。
下面这个例子遍历一个链表。它用map来检测list中是否存在环。

```go
type Node struct {
        Next  *Node
        Value interface{}
    }
    var first *Node

    visited := make(map[*Node]bool)
    for n := first; n != nil; n = n.Next {
        if visited[n] {
            fmt.Println("cycle detected")
            break
        }
        visited[n] = true
        fmt.Println(n.Value)
    }
```

如果节点n被访问过，则visited\[n\]为true。
这就不需要用two-value的格式来，因为默认零值帮我们做了这些。

另一个游泳的例子是map of slice。
append一个nil的slice相当于分配一个新的slice，因此它append一个值是one-liner的；它不需要检验key是否存在。
在下面的例子中，切片peaple赋值为Person值。
每个Person有一个Name和Like切片。
这个例子创建一个map来关键like的用户列表。

```go
 type Person struct {
        Name  string
        Likes []string
    }
    var people []*Person

    likes := make(map[string][]*Person)
    for _, p := range people {
        for _, l := range p.Likes {
            likes[l] = append(likes[l], p)
        }
    }
```

要打印喜欢cheese的用户，如下：

```go
for _, p := range likes["cheese"] {
        fmt.Println(p.Name, "likes cheese.")
    }
```

要打印喜欢bacon的用户数，如下：

```go
fmt.Println(len(likes["bacon"]), "people like bacon.")
```

因为range和len都把nil的slice看作长度为0的slice，因而即便没有人喜欢cheese或bacon，代码也不会有错。


### key的类型 

正如前面提到的，map的key可以是任何可以比较的类型。
(go语言规范)[https://golang.org/ref/spec#Comparison_operators]对此做了精确的定义，简而言之，可比较的类型包括：布尔型、数值型、字符串、指针、channel等。
它不包括slice、map和函数；这些类型不能通过==进行比较，因而它们不能用作map的key。

显然，string、int以及其他基本类型可以用作map的key，但是struct是例外。
struct可以以多个维度来做为key。
例如，这个map可以用于不同国家点击的网页总数。

```go
hits := make(map[string]map[string]int)
```

这是一个string到map的map。每一个外部的map指向它内部的map。内部的map是两个字符的国家码。
这个表达检索了澳大利亚登录文档页面的次数：

```go
n := hits["/doc/"]["au"]
```

不过，这种方式在新加入数据时并不方便；对于每个外部的map，你需要判断内部的map是否存在，并决定是否需要创建：

```go
func add(m map[string]map[string]int, path, country string) {
    mm, ok := m[path]
    if !ok {
        mm = make(map[string]int)
        m[path] = mm
    }
    mm[country]++
}
add(hits, "/doc/", "au")
```

另一方面，可以用一个以struct为key的map来解决这个复杂的问题：

```go
type Key struct {
    Path, Country string
}
hits := make(map[Key]int)
```

当一个越南的用户登录时，可以增加其技术：

```go
hits[Key{"/", "vn"}]++
```

同样地，可以用以下方式直接看有多少瑞士用户访问了spec页面：

```go
n := hits[Key{"/ref/spec", "ch"}]
```

(map不是并发安全的)[https://golang.org/doc/faq#atomic_maps]：它并不保证在你同时读和写map时，会发生什么。
如果你需要从并发执行的goroutine同时读和写map，其访问必须要通过一些同步机制。
常用的方法是[sync.RWMutex](https://golang.org/pkg/sync/#RWMutex)。

以下声明了一个counter变量，它是一个匿名结构，包含了一个map和内置的sync.RWMutex。

```go
var counter = struct{
    sync.RWMutex
    m map[string]int
}{m: make(map[string]int)}
```

需要读的时候，使用读锁：

```go
counter.RLock()
n := counter.m["some_key"]
counter.RUnlock()
fmt.Println("some_key:", n)
```

需要写的时候，使用写锁：

```go
counter.Lock()
counter.m["some_key"]++
counter.Unlock()
```

### 迭代顺序

当使用range来迭代map的时候，它的顺序并不确定，也不保证每次的顺序都一样。
如果你要保证稳定的迭代顺序，你必须维护一个独立的数据结构。
这个例子使用了一个独立的有序的slice结构来顺序地输出map[\int\]的key。

```go
import "sort"

var m map[int]string
var keys []int
for k := range m {
    keys = append(keys, k)
}
sort.Ints(keys)
for _, k := range keys {
    fmt.Println("Key:", k, "Value:", m[k])
}
```



