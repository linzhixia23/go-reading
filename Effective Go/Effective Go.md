原文：https://golang.org/doc/effective_go.html

<br>

###  一、介绍
Go是一门全新的语言。虽然它从现有的语言借鉴了一些理念，但它的语言特性，使得编写高效的go程序与其他语言完全不同。
直接把现有的C++或Java程序翻译成go，结果并不理想--毕竟Java程序不是go。
另一方面，如果从go的视角去思考，则能够写出更好的、不一样的代码。
换句话说，要想把go程序写好，需要深入理解它的特性和风格。
同时，还要了解go的既定规则，包括命名规范、格式、代码结构等，这样你的代码才更有可读性。

这篇文档介绍一些tips，帮助编写清晰地道的go代码。它是对之前文档的补充说明，因此建议先阅读这些文档。

##### 示例
go源码包不仅提供了核心库，同时还有一些例子示范如何使用go语言。
此外，很多库包含了可执行的示例。
如果你有对一些实现有疑问，你可以通过代码库的文档、代码和示例，获得解答，并了解结果背后的思想方法。


### 二、格式
格式问题经常引起争议，但几乎讨论不出结果。尽管码农能适应各种编码风格，但如果没有岂不是更好。倘若大家都使用同样的风格，那么在这上面花的时间就越少。
问题在于，如何在没有强制的风格指引的条件下，实现这个乌托邦。

在go里面，我们采取了不同的方式--通过让机器来解决大多数的格式问题。
*gofmt*程序把go代码按照标准风格缩进、对齐、保持一致，必要时还会优化注释（reformatting comments）。
如果你想知道如何代码布局，请运行*gofmt*；如果答案看起来不正确，可以自己重新组织你的程序（或者提交bug单到*gofmt*）。

举个例子，在定义结构体的时候，不需要花费时间对齐注释，*Gofmt*会为你做这些。像下面这个定义：

```go
type T struct {
    name string // name of the object
    value int // its value
}
```

*gofmt* 自动会按列对齐 

```go
type T struct {
    name    string // name of the object
    value   int    // its value
}
```

标准库里的所有代码都是用*gofmt*做格式化的。

还有一些小细节：
##### 缩进 
  我们使用tab缩进，*gofmt*也会默认使用它。因而，只在必要的时候才用空格。
  
##### 行长度
  Go没用行长度的限制，不要担心长度溢出。如果觉得一行太长，就换行，并用tab缩进。
  
##### 括号
  Go比C和Java的行数更少：控制结构（if，for，switch）在语法上不需要大括号。
  同时，操作符优先级更加简洁，"x<<8 + y<<16" 表示空格分割表达的意思，而不像其他语言。 
   

### 三、注释
Go支持C-风格的块注释/**/，以及C++风格的行注释//。行注释比较常见；块注释常见于包的注释，但它对于大段代码的注释也比较管用。

*godoc*程序是个web服务，能够从go的源代码中提取包的注释。注释通常出现在top-level的声明位置，与声明之间没有空行。
它会跟声明一起被提取出来，作为该条目的说明文档。这些注释的风格和类型，决定了*godoc*生成文档的质量。

每个包都应该有一个包注释（package comment），即一段在package clause前面的块注释。
对于有多个文件的包，包注释只要放在其中任意一个文件就可以。
包注释应该介绍包，以及包的相关信息。它会出现在*godoc*页面的最前面，并随后生成详尽的文档。

```go
/*
Package regexp implements a simple library for regular expressions.

The syntax of the regular expressions accepted is:

    regexp:
        concatenation { '|' concatenation }
    concatenation:
        { closure }
    closure:
        term [ '*' | '+' | '?' ]
    term:
        '^'
        '$'
        '.'
        character
        '[' [ '^' ] character-ranges ']'
        '(' regexp ')'
*/
package regexp
```

如果包比较简单，包注释也可以简略。
```
// Package path implements utility routines for
// manipulating slash-separated filename paths.
```

注释不需要额外的格式，例如星号凸显等。生成的文件甚至不会以等宽字体显示，因此不需要空格来对齐。
注释是不会被解析的纯文本，像HTML格式的*_this_*，则会按原样输出，所以最好不要这样使用。
*godoc*做的另一个调整，就是把缩进文本按等宽字体调整，更好地适应程序的snippet。*gofmt*包就用了包注释的方式。

*godoc*是否重新格式化注释取决于上下文，因而要保证它们看起来清晰明了：正确的拼写、标点和句式结构，折叠长行等。

根据上下文，godoc并不会重新格式化注释，所以要保证它们：用正确的拼写、句子结构、折叠长的行等等

在一个包里，任何top-level前面的注释，都会做为其注释文档。每个可以导出的命名，都应该有注释文档。

注释文档最好是完整的句子，这样才能适应自动化的展示。它最好是一句话的总结，并且以被申明的名称做为第一个词。

```
// Compile parses a regular expression and returns, if successful,
// a Regexp that can be used to match against text.
func Compile(str string) (*Regexp, error) {
```

如果每个注释文档都以名称开头，你可以用go tool的doc子命令，并对输出进行grep。
例如，你记不住名称"Compile"，但又想查找正则表达式的解析函数，你可以运行如下命令：

```
$ go doc -all regexp | grep -i parse
```

但如果这个包里的注释文档，都是以"This Function..."开头的，*grep*就无法帮你记住名字。 
正是因为注释文档以其名称开头，因而可以找到你要的词，正如下面这样：

```
$ go doc -all regexp | grep -i parse
    Compile parses a regular expression and returns, if successful, a Regexp
    MustCompile is like Compile but panics if the expression cannot be parsed.
    parsed. It simplifies safe initialization of global variables holding
$
```

Go的声明语法允许分组声明。单个文档的注释应该介绍相关的常量或变量。由于是整个声明，这种注释通常较为简单。

```go
// Error codes returned by failures to parse an expression.
var (
    ErrInternal      = errors.New("regexp: internal error")
    ErrUnmatchedLpar = errors.New("regexp: unmatched '('")
    ErrUnmatchedRpar = errors.New("regexp: unmatched ')'")
)
```

分组还可以用来指示各item之间的关系，例如一组由mutex进行保护的变量。

```go
var (
    countLock   sync.Mutex
    inputCount  uint32
    outputCount uint32
    errorCount  uint32
)
```

### 四、命名 

跟其他语言一样，命名在Go里是很重要的。它们有语法影响（semantic effect）：一个命名的首字母是否大写，影响到它在包外是否可见。
因此，需要花一点时间介绍Go的命名规范。

##### 包名 
当一个包被导入后，就可以用包名访问它的内容：

```go
import "bytes"
```

可以引用bytes.Buffer。所有人都用包名来引用，这就意味着包名需要更简短、易于记忆。
通常惯例，包名是一个小写的单词；而不应该使用下划线或者是驼峰。
简洁可以避免犯错，因为每个使用包的人，都会输入这个名字。
包名是import时默认的名称，它不强制要求在所有源码里是唯一的，在偶尔出现冲突的情况下，导入程序包可以选择一个不同的名字在本地使用。
任何情况下，都不会造成困惑，因为文件名就决定了要使用哪个包。

另一个惯例是，包名是基于其源码路径名称的；在路径 src/encoding/base64 下的包应该用 "encoding/base64" 来导入，其包名是 base64， 而不是 encoding_base64 或 encodingBase64。

导入包后，可以使用包名来引用其内容，因此可以利用这一点来避免歧义。（不要使用 import . 符号，它可以简化必须在包之外运行的测试，除此之外，应该尽量避免使用）。
例如，*bufio*包中 buffered reader类型叫做Reader，而不是BufReader，因为我们是按照bufio.Reader来使用它，这样更加简洁清晰。
另外，由于导入的包总是通过它的包名来确定，因此 bufio.Reader 并不会与 io.Reader 冲突。
类似的，用于初始化ring.Ring实例的函数（相当于go中的构造函数），它通常会命名为NewRing，但因为Ring是保留唯一的类型，而且包名叫ring，因此它可以只叫New。这样看到的就是ring.New。
利用包的结构，可以帮助你选一个好的名字。

另一个小例子是*once.Do*;once.Do表达很清晰，而没有必要写成once.DoOrWaitUntilDone。
长的名字并不会增加可读性。好的注释文档会比额外的长名称更有价值。

##### Getters 
Go没有提供getter和setter的自动化支持。提供getter和setter本身没有错，而且通常很有必要这么做，但把Get放到getter名字的前面，既不符合惯例，也没有必要。
如果你有一个成员变量叫owner（小写，未导出），那么其getter方法应该叫Owner（大写，导出），而不是GetOwner。
首字母大写即可以导出这种方式为区分成员变量和方法提供了便利。
一个setter方法，如果有必要的话，可以写成SetOwner。两个名字看起来都不错。

```go
owner := obj.Owner()
if owner != user {
    obj.SetOwner(user)
}
```

##### 接口名
通常，一个方法接口命名为方法名加上 -er 做为后缀，或者类似的名词，如：Reader, Writer, Formatter, CloseNotifier等。

有很多这样的命名，遵循它们会让事情变得简单。Read, Write, Close, Flush, String等都有典型的特征和含义。
为了避免困惑，不要用它们来命名你的方法，除非你有相同的含义。
反过来，如果你实现了一个类型的方法，与常见的方法有相同的含义，那就用相同的名字；将你的string转换方法命名为String，而不是ToString。 

##### 驼峰记法
最后，在go里通常是用驼峰，而不是下划线。 

### 五、分号 
跟C语言一样，Go的标准语法是用分号来结束语句的，但跟C语言不同的是，这些分号不需要出现在源码里。
它在词法分析器扫描的时候，会根据一些简单的规则插入分号。
 
规则是，如果一行的最后一个token是定义符（包括int、float64等），基本的文字（数字、字符串常量等），或者是以下符号之一：
 
```go 
 break continue fallthrough return ++ -- ) }
```
 
词法分析器总是会在这些token后插入分号。
这点可以概括为，如果新行前的token，它可能为语句的结尾，则插入分号。
（if the newline comes after a token that could end a statement, insert a semicolon）
 
分号也可以在闭合的括号前省略，因而像这样的语句：
 
```go
go func() { for { dst <- <-src } }()
```
 
则不需要分号。通常Go程序只有在for循环子句，用于分离初始器、条件器和增量操作。
如果你在一行写多个语句，也需要用分号分开。

分号插入规则的一个结果是，控制结构（如if、for、switch）的左括号不能放在下一行。
如果你这样做，分号会插在括号的前面，而这不是我们希望的。
所以，要写成这样

```go
if i < f() {
    g()
}
```

而不是这样

```go
if i < f()  // wrong!
{           // wrong!
    g()
}
```

### 六、控制结构 

Go的控制结构与C有相似之处，但在一些重要的地方不同。
它没有do或者while的循环，只有一个更通用的for；switch更加灵活；if和switch可以像for一样，接受初始化语句；
break和continue可以根据一个label去决定break或continue；
它还提供了一个新的结构控制类型，用于类型选择和多路通信复用，select。
此外，它在语法上也有细微的不同：它没有括号，但body则必须在大括号里。

##### if 

在go里，一个简单的if语句是像这样的：

```go
if x > 0 {
    return y
}
```

强制的括号鼓励把简单的if语句分多行来写，这样写是有好处的，尤其是在body里有return或break。

由于if和switch可以使用初始化语句，因此可以用来初始化局部变量：

```go
if err := file.Chmod(0664); err != nil {
    log.Print(err)
    return err
}
```

在Go的库中，如果if语句不会执行到下一行，比如body里以break, continue, goto, return 结束，则可以忽略不必要的else。

 ```go
 if, err := os.Open(name)
 if err != nil {
     return err
 }
 codeUsing(f)
 ```
 
以下是一段常见的代码，需要判断一系列错误的条件。如果控制流顺利运行，则说明错误的情况都被排除了，这样代码的可读性就比较好。
由于出现错误就直接返回了， 因此后面的代码也就不需要else了。

```go
 f, err := os.Open(name)
 if err != nil {
     return err
 }
 d, err := f.Stat()
 if err != nil {
     f.Close()
     return err
 }
 codeUsing(f, d)
 ```
 
##### 重定义和重赋值
题外话：前一节的最后一个用例展示了 := 如何使用。 
os.Open的声明是：

```go
f, err := os.Open(name) 
```

这个语句有两个变量，f和err。 几行之后，又调用 

```go
d, err := f.Stat()
```

看起来似乎是声明了d和err。注意，err其实出现在了两个语句。
这种重复是合法的：err在第一个语句中声明，在第二个语句中被再次赋值。
也就是说，f.stat调用使用了前面声明的的变量err，并赋了一个新的值。

在 := 声明中，一个变量v即便之前已经被声明了，它同样可以使用：

* 这个声明与之前声明的v处于同一个作用域中（如果v已经在外层作用域声明，则这次声明会重新创建一个新的变量§） 

* 初始化时相应的值会赋给v 

* 在这次声明中，至少要有一个变量是新声明的。 

这个特性是纯粹的实用主义，它使得err这个变量更加方便使用，例如在一个相当长的if-esle语句链中。

§ 值得一提的是，即便go的形参和返回值在语法上是在括号外，但其作用域仍然是在函数体内。

##### for 

在go里，for的用法跟C类似-但不是完全相同。它统一了for和while，不再有do-while。
它有3种形式，其中只有1种需要用到分号

```go
// Like a C for
for init; condition; post { }

// Like a C while
for condition { }

// Like a C for(;;)
for { }
```

简短的声明让我们更容易在for循环中声明下标变量。

```go
sum := 0
for i := 0; i < 10; i++ {
    sum += i
}
```

如果你在循环遍历array、slice、string、map，或者从channel读数据，使用range会更加方便

```go
for key, value := range oldMap {
    newMap[key] = value
}
```

如果你只需要range的第一个item（key或者是index），则可以省略第二个item：

```go
for key := range m {
    if key.expired() {
        delete(m, key)
    }
}
```

如果你只需要第二个item（value），则使用空白标识符，即下划线，来缺省第一个item：

```go
sum := 0
for _, value := range array {
    sum += value
}
```

空白标识符有很多用法，我们后面会介绍。 

对于字符串来说，range操作可以根据解析utf-8编码，把每个独立的uniode码分离出来。
每个错误的编码会消耗一个字节，并用一个父文（rune）U+FFFD代替。
rune是go的术语，用于称呼单个unicode码点（Unicode code point）。

代码
```go
for pos, char := range "日本\x80語" { // \x80 is an illegal UTF-8 encoding
    fmt.Printf("character %#U starts at byte position %d\n", char, pos)
}
```

会输出

```go
character U+65E5 '日' starts at byte position 0
character U+672C '本' starts at byte position 3
character U+FFFD '�' starts at byte position 6
character U+8A9E '語' starts at byte position 7
```

最后，go没有逗号操作，++和--是语句而不是表达式。因此，如果你需要在for中使用多个变量，应该使用并行赋值（parallel assignment）的方式，尽管这样做会影响使用++或--。

```go
// Reverse a
for i, j := 0, len(a)-1; i < j; i, j = i+1, j-1 {
    a[i], a[j] = a[j], a[i]
}
```

##### switch 
go的switch比C更加通用。表达式不需要是常量或者整数，case语句会自上到下进行求值，直到匹配为止。
如果switch没有表达式，它会匹配true。因此，按照惯例，尽量把if-else-if-else用swithc表达：

```go
func unhex(c byte) byte {
    switch {
    case '0' <= c && c <= '9':
        return c - '0'
    case 'a' <= c && c <= 'f':
        return c - 'a' + 10
    case 'A' <= c && c <= 'F':
        return c - 'A' + 10
    }
    return 0
}
```

switch不会自动地从一个case下溯到另一个case，但case可以使用逗号分隔的别表。
```go
func shouldEscape(c byte) bool {
    switch c {
    case ' ', '?', '&', '=', '#', '+', '%':
        return true
    }
    return false
}
``` 

尽管go可以用break来提前结束switch分支，但它在go中并不像C语言那样常见。
有事我们需要跳出包含它的循环，而不是switch，在go里面，可以设置一个lable在循环外，然后跳转到那个label。
以下代码展示了上述两个情况的用法：
```go
Loop:
	for n := 0; n < len(src); n += size {
		switch {
		case src[n] < sizeOne:
			if validateOnly {
				break
			}
			size = 1
			update(src[n])

		case src[n] < sizeTwo:
			if n+1 >= len(src) {
				err = errShortInput
				break Loop
			}
			if validateOnly {
				break
			}
			size = 2
			update(src[n] + src[n+1]<<shift)
		}
	}
``` 

当然，continue也能使用一个optional的标签，但它只能在循环中使用。

做为本节的结束，来看一段代码，它使用switch对两个slice做对比。
```go
// Compare returns an integer comparing the two byte slices,
// lexicographically.
// The result will be 0 if a == b, -1 if a < b, and +1 if a > b
func Compare(a, b []byte) int {
    for i := 0; i < len(a) && i < len(b); i++ {
        switch {
        case a[i] > b[i]:
            return 1
        case a[i] < b[i]:
            return -1
        }
    }
    switch {
    case len(a) > len(b):
        return 1
    case len(a) < len(b):
        return -1
    }
    return 0
}
```

##### 类型切换（type switch） 
switch可以用于发现一个interface变量的动态类型。类型选择通过类型断言的语法，在括号中使用关键字type。 
如果switch在表达式中声明了一个变量，那么该变量的每个子句中都有其对应的类型。
比较符合语言习惯的方式是在这些case里重用一个名字，其实它是在每个case里声明同名，但不同类型的变量。

```go
var t interface{}
t = functionOfSomeType()
switch t := t.(type) {
default:
    fmt.Printf("unexpected type %T\n", t)     // %T prints whatever type t has
case bool:
    fmt.Printf("boolean %t\n", t)             // t has type bool
case int:
    fmt.Printf("integer %d\n", t)             // t has type int
case *bool:
    fmt.Printf("pointer to boolean %t\n", *t) // t has type *bool
case *int:
    fmt.Printf("pointer to integer %d\n", *t) // t has type *int
}
```

### 七、函数

##### 多返回值 
Go的一个与众不同的特点是，它允许函数有多个返回值。这种形式可以用于改善C语言的一些笨拙的风格：把错误值返回（例如用-1表示EOF），并且修改通过地址传入的实参。

在C语言中，写入的错误是通过传入负的计数，以及隐藏在易变位置（ volatile location）的错误代码来给出的。
而在go里，则可以返回写入数据的计数，以及错误信息，"是的，你写入了一些数据，但不是所有的，因为设备已经写满了"。
在os包里，write方法的签名（signature）是：

```go
func (file *File) Write(b []byte) (n int, err error)

```

正如文档里写的，它会返回写入的字节数，以及非空的error（如果 n != len(b)的话）。
这是一种常见的风格；可以在异常处理那一节看到更多的示例。

我们可以通过一个类似的方法，来避免为模拟引用参数而传入指针。
以下是一个例子，它从字节slice的一个位置获取一个数字，返回这个数字和下一个位置。

```go
func nextInt(b []byte, i int) (int, int) {
	for ; i < len(b) && !isDigit(b[i]); i++ {
	}
	x := 0
	for ; i < len(b) && isDigit(b[i]); i++ {
		x = x*10 + int(b[i]) - '0'
	}
	return x, i
}
```

你可以用它扫描一个输入的slice，就像这样
```go
	for i := 0; i < len(b); {
        x, i = nextInt(b, i)
        fmt.Println(x)
    }
```

##### 命名的结果参数（Named result parameters）
Go函数的返回结果可以进行命名，并把它做为一个普通的变量来使用，就像传入的形参一样。
当返回参数（return parameter）被命名，会在函数开始时，根据它们的类型初始化为0；如果函数执行了return，而没有实参，当前的返回参数会被用作return的值。

命名不是强制的，但它们可以使代码变得简短而清晰：它们本身也是文档。
如果我们将nextInt的结果进行命名，那么哪个int是返回的int就很容易看出来了。 

```go
func nextInt(b []byte, pos int) (value, nextPos int) {
```
由于命名结果已经被初始化，而且关联到没有实参的return，因此它们可以很简单明了。
以下是io.ReadFull的一个示例：

```go
func ReadFull(r Reader, buf []byte) (n int, err error) {
    for len(buf) > 0 && err == nil {
        var nr int
        nr, err = r.Read(buf)
        n += nr
        buf = buf[nr:]
    }
    return
}
```

##### defer 
Go的defer语句调度一个函数调用，使其在函数返回前执行。
这是一种不常见，但又有效的方法，例如有些资源必须要回收，不管它是通过哪个运行路径返回的。
典型的例子是解互斥锁，以及关闭文件。 

```go
// Contents returns the file's contents as a string.
func Contents(filename string) (string, error) {
    f, err := os.Open(filename)
    if err != nil {
        return "", err
    }
    defer f.Close()  // f.Close will run when we're finished.

    var result []byte
    buf := make([]byte, 100)
    for {
        n, err := f.Read(buf[0:])
        result = append(result, buf[0:n]...) // append is discussed later.
        if err != nil {
            if err == io.EOF {
                break
            }
            return "", err  // f will be closed if we return here.
        }
    }
    return string(result), nil // f will be closed if we return here.
}
```

Defer诸如Close这样的函数有两个好处。
第一个，它使得你不会忘记关闭文件——这是一个很常见的错误，尤其是你之后编辑函数，去增加一个新的返回路径。
第二个，它可以把close写在open附近，这比把close写在函数最后更加清晰。

defer函数的实参在defer执行时就会被赋值，而不是它实际被执行的时候。
因此不需要担心变量在函数执行时被改变。
这样意味着，单个被推迟的调用，可以推迟多个函数的执行。以下是简单的例子：

```go
for i := 0; i < 5; i++ {
	defer fmt.Printf("%d ", i)
}
```

被推迟的函数会按照后进先出的顺序执行，所以以上代码的输出是 4 3 2 1 0。
一个更实际的例子是，用一个简单的方式tracing函数的执行。
我们可以实现一个简单的跟踪协程：

```go
func trace(s string)   { fmt.Println("entering:", s) }
func untrace(s string) { fmt.Println("leaving:", s) }

// Use them like this:
func a() {
    trace("a")
    defer untrace("a")
    // do something....
}
```

我们利用defer函数的实参是在defer执行的时候被求值的这个特点，把代码优化一下。
tracing协程可以为untracing函数设定实参。如以下例子：

```go
func trace(s string) string {
    fmt.Println("entering:", s)
    return s
}

func un(s string) {
    fmt.Println("leaving:", s)
}

func a() {
    defer un(trace("a"))
    fmt.Println("in a")
}

func b() {
    defer un(trace("b"))
    fmt.Println("in b")
    a()
}

func main() {
    b()
}
```

输出 

```go
entering: b
in b
entering: a
in a
leaving: a
leaving: b
```

对于熟悉其他语言block-leve资源管理的程序员，defer可能看起来很奇怪，但它最有趣、最强大的特点就是来自于它是基于函数的，而不是基于块的。
在 pania 和 recover 这两节，还会看到它更多的可能性。
 
### 八、数据 
 
##### new分配  
Go有两个分配空间的原语，即内建函数new和make。它们做不同的事情，运用于不同的类型，可能会让人困惑，但其中的规则很简单。
我们先来说new，它会分配内存，但跟其他语言的同名函数不同，它不会初始化内存，只会将内存置为0。
也就是说，new(T)分配0的空间给一个类型T的item，并返回它的地址，也就是*T的值。
用GO的术语说，它返回一个指针，指向新分配的类型为T的0值。

既然内存已经是0值，那么在设计数据结构时，每种类型的0值就不必初始化而直接使用。
这意味着，数据结构的使用者，只要new一个新的对象，就可以直接使用。
例如，bytes.Buffer的文档声明，"0值的Buffer就是一个空的准备就绪的缓冲区"。
同样的，sync.Mutex 没有一个显示的构造器或者init方法。而它的0值则定义为已解锁的mutex。

它的zero-value-is-useful特性是有传递性的。考虑以下类型声明：

```go
type SyncedBuffer struct {
    lock    sync.Mutex
    buffer  bytes.Buffer
}
```

SyncedBuffer 类型的值也是在声明时就分配好内存就绪了。
在后续的代码中，p和v可以正常的运行，而不需要额外的处理。
```go
p := new(SyncedBuffer)  // type *SyncedBuffer
var v SyncedBuffer      // type  SyncedBuffer
```

##### 构造器和复合字面量（composite literals）
有时0值还不够完美，这里就需要一个初始化的构造器。正如以下这个从os包里的代码：
```go
func NewFile(fd int, name string) *File {
    if fd < 0 {
        return nil
    }
    f := new(File)
    f.fd = fd
    f.name = name
    f.dirinfo = nil
    f.nepipe = 0
    return f
}
```

它有很多模板。我们可以用composite literal来简化，它是一个表达式，在每次求值时都会创建一个新的实例。

```go
func NewFile(fd int, name string) *File {
    if fd < 0 {
        return nil
    }
    f := File{fd, name, nil, 0}
    return &f
}
```

注意，返回一个局部变量的地址是没问题的，这点跟C语言不同。该局部变量对应的数据，在函数返回时依然有效。
实际上，使用composite literal的地址会在每次求值时，获得一个新的实例。因此，我们可以把上面最后两个代码合并。

```go
return &File{fd, name, nil, 0}
```

composite literal的field必须按顺序给出，并且要包含所有属性。
如果以field:value的方式显示地标记字段，则可以不按顺序，对于缺省的filed，则会补0值。
因此，我们可以用下面的形式：

```go
return &File{fd: fd, name: name}
```

在个别情况下，composite literal不包含任何field，则会创建该类型的0值。
此时，new(File)和&File{}是等价的。

Composite literals可以用于array、slice和map， field labels是索引还是关键字则视情况而定。
在下列的例子中，初始化时，不管Enone, Eio和Einval的值是什么，只要它们不同就可以。

```go
a := [...]string   {Enone: "no error", Eio: "Eio", Einval: "invalid argument"}
s := []string      {Enone: "no error", Eio: "Eio", Einval: "invalid argument"}
m := map[int]string{Enone: "no error", Eio: "Eio", Einval: "invalid argument"}
```

##### make分配
回到内存分配的问题。内置函数make(T, args)的用法目的不同于new(T)。
它只能创建slice、map和channel，它会返回一个初始化（非置0）的类型T（不是*T）。
出现这种差异的原因在于，这三种类型本质上是引用类型，它们在使用前必须初始化。
例如，slice是一个具有3项描述符的内容，包括指向数据的指针，数据长度和容量，在它们被初始化之前，slice都是nil。
对于slice、map和channel，make函数会初始化内部的数据结构，并准备好要使用的值。例如，

```go
make([]int, 10, 100)
```

会分配一个具备100个int的数组空间，然后会创建一个slice，它的长度是10，容量是100，并指向前10个元素。
创建slice时，容量是可以缺省的，更多的信息可以看slice章节。
相对比，new([]int)则返回一个新分配的，已置0的slice结构的指针，即指向nil的slice的指针。

下面例子显示了new和make的区别。

```go
var p *[]int = new([]int)       // allocates slice structure; *p == nil; rarely useful
var v  []int = make([]int, 100) // the slice v now refers to a new array of 100 ints

// Unnecessarily complex:
var p *[]int = new([]int)
*p = make([]int, 100, 100)

// Idiomatic:
v := make([]int, 100)
```
 
记住，make只应用于map、slice和channel，而且不会返回一个指针。
要想获得指针，需要用new分配，或者显示地获取变量的地址。

##### Array类型
在规划内存时，array是非常有用的，有时它也可以帮助避免分配，但最主要的是，它们是slice的基本构建。
做为铺垫，这里简单介绍array。 

Array在go和C有一些不同。在GO里，

* array是值。把一个array赋予给另一个，会复制所有的元素。
* 特别的，如果你把一个array传给一个函数，它会获得array的拷贝，而不是指针。
* array的size也是它的类型的一部分。所以，[10]int 和 [20]int 是不同的。

array是value的特性很有用，但同时代价很高。因此，如果你希望类似C的高效，你可以传一个数组的指针。

```go
func Sum(a *[3]float64) (sum float64) {
    for _, v := range *a {
        sum += v
    }
    return
}

array := [...]float64{7.0, 8.5, 9.1}
x := Sum(&array)  // Note the explicit address-of operator
```

不过，这种风格不符合go的语言习惯。最好是使用slice。

##### slice 
slice封装了array，提供一个更加通用的、强大的和方便的接口。
除了矩阵转置这种需要明确知道维度的情况，大多数程序在go里是通过slice，而不是array来完成的。

slice保存了对底层数组的引用，如果你把一个slice赋予到另一个，它们都指向同一个数组。
如果一个函数传入了一个slice的实参，改变它的元素对调用者是可见的，这等同于传递了slice底层的array的指针。
因此，Read函数可以接受一个slice的实参，而不是一个指针和一个计数；slice中的长度规定了能读取的数据的上限。
以下是os包的File类型的Read方法：

```go
func (f *File) Read(buf []byte) (n int, err error)
```

该函数返回读取的字节数，以及一个错误码--如果有的话。
如果要从一个更大的缓冲区buf读取前32个字节，只要对它切片就可以。

```go
n, err := f.Read(buf[0:32])
``` 

这样的切片是很常见而且高效的。事实上，如果不考虑效率，以下代码同样能读取前32个字节。

```go
    var n int
    var err error
    for i := 0; i < 32; i++ {
        nbytes, e := f.Read(buf[i:i+1])  // Read one byte.
        n += nbytes
        if nbytes == 0 || e != nil {
            err = e
            break
        }
    }
```

只要切片的长度不超过它底层的数组，它的长度就是可以改变的；只要将它赋予其自身的切片就可以。
切片的容量可以通过内嵌函数cap获得，它给出切片的最大长度。
如果数据超过了容量，切片会重新分配。返回值即为得到的切片。
函数len和cap在切片为nil时，也是合法的，它会返回0。

```go
func Append(slice, data []byte) []byte {
    l := len(slice)
    if l + len(data) > cap(slice) {  // reallocate
        // Allocate double what's needed, for future growth.
        newSlice := make([]byte, (l+len(data))*2)
        // The copy function is predeclared and works for any slice type.
        copy(newSlice, slice)
        slice = newSlice
    }
    slice = slice[0:l+len(data)]
    copy(slice[l:], data)
    return slice
}
```

我们必须在最后返回slice，因为虽然Append可以修改slice的元素，但是slice本身是通过值来传递的。
其run-time的数据结构包括指针、长度和容量。

向slice添加元素是很有用的，因而有专门的内建函数append。
要了解该函数的设计和思想，我们还需要一些额外的信息，我们会在稍后讨论。

##### 二维slice 

Go的array和slice是一维的。要想创建一个等价于2维的array或slice，则需要定义array-of-array或者slice-of-slice，就像这样：

```go
type Transform [3][3]float64  // A 3x3 array, really an array of arrays.
type LinesOfText [][]byte     // A slice of byte slices.
```

由于slice是长度可变的，因此它内部每个slice的长度可以不同。
这种情况很常见，正如我们LinesOfText这个例子：

```go
text := LinesOfText{
	[]byte("Now is the time"),
	[]byte("for all good gophers"),
	[]byte("to bring some fun to the party."),
}
```

有时需要分配2D的切片，例如需要处理像素的扫描时。有两种方法实现。
一种是单独地分配每个切片；另一种是分配一个单独的array，并把单独的slice指向它。
使用哪个取决于你的实际应用。
如果你的slice会增长或缩减，你需要单独分配它，避免覆盖到下一行。
但如果不是，用一个分配来构建对象会更加有效。
以下是两个方式的代码，仅供参考：

每次分配1行
```go
// Allocate the top-level slice.
picture := make([][]uint8, YSize) // One row per unit of y.
// Loop over the rows, allocating the slice for each row.
for i := range picture {
	picture[i] = make([]uint8, XSize)
}
```

一次分配，把它切片成多行：
```go
 // Allocate the top-level slice, the same as before.
 picture := make([][]uint8, YSize) // One row per unit of y.
 // Allocate one large slice to hold all the pixels.
 pixels := make([]uint8, XSize*YSize) // Has type []uint8 even though picture is [][]uint8.
 // Loop over the rows, slicing each row from the front of the remaining pixels slice.
 for i := range picture {
 	picture[i], pixels = pixels[:XSize], pixels[XSize:]
 }
```

##### Map 
Map是一个方便而强大的内置数据结构。
Map的key可以是任何的类型，只要它定义了相等操作符，包括：整型、浮点、复数、字符串、指针、接口（只要动态类型支持等号）、结构数组等。
slice不能用过map的key，因为它没有定义相等性。
像slice一样，map也是有一层引用到它底层的数据结构。如果将map传入函数中，并更改了该函数的内容，则这次修改对调用者同样可见。

Map可以用常见的composite literal来构建，要在key-value对之间加入逗号。它可以在创建的时候初始化。

```go
var timeZone = map[string]int{
    "UTC":  0*60*60,
    "EST": -5*60*60,
    "CST": -6*60*60,
    "MST": -7*60*60,
    "PST": -8*60*60,
}
```

赋值和获取map的value值，在语法上跟对array和map的操作类似，但它的index不要求是整型。

```go
offset := timeZone["EST"]
```

如果试图通过一个不存在的key从map取值，它会返回该类型对应的0值。
例如，如果map包含整型，查找一个不存在的key，将会返回0值。
set类型可以用map来实现，只要把它的value设置为bool类型。
设置map项为true，表示把这个值放入set中；然后通过简单的索引操作即可判断它是否存在。

```go
attended := map[string]bool{
    "Ann": true,
    "Joe": true,
    ...
}

if attended[person] { // will be false if person is not in the map
    fmt.Println(person, "was at the meeting")
}
```

有时你需要去区分，map返回的0是表示key不存在，还是它真正的value就是0。你可以通过多重赋值来区分开来。

```go
var seconds int
var ok bool
seconds, ok = timeZone[tz]
```

我们可以称这个为"comma ok"惯例。在这里例子中，如果tz存在，seconds会被赋予相应值，且ok为true；如果不存在，ok会被设置为0，且ok为false。
以下例子在此基础上，加了错误提示：
```go
func offset(tz string) int {
    if seconds, ok := timeZone[tz]; ok {
        return seconds
    }
    log.Println("unknown time zone:", tz)
    return 0
}
```

如果只是要测试一个值是否在map里，而不在乎它实际的值，你可以用空白标识符(_)来代替。
 
```go
_, present := timeZone[tz]
``` 

如果要删除一个map的值，可以用内置函数delete。它的实参是map，以及要被删除的key。
即便key不在map里，这样做也是安全的。

```go
delete(timeZone, "PDT")  // Now on Standard Time
```

##### printing 
Go的格式化输出跟C的printf相似，但其更加丰富而通用。
这些函数都在fmt包里，都是以大写字母开头：fmt.Printf, fmt.Fprintf和fmt.Sprintf等。
字符串函数（如Sprintf）会返回一个string，而不是填充给定的缓冲区。

你不是需要提供一个格式化的字符串。对每一个函数，如Printf、Fprintf、Sprintf都分别对应另外一个函数，如Print和Println。
这些函数不接受格式化的字符串，而是对每一个实参生成默认的格式。
Println函数会在实参之间插入空格，并在输出时追加换行符；而Print只会在操作数两侧都没有字符串时才加入空白。
这下面示例中，每一行都是相同的输出。

```go
fmt.Printf("Hello %d\n", 23)
fmt.Fprint(os.Stdout, "Hello ", 23, "\n")
fmt.Println("Hello", 23)
fmt.Println(fmt.Sprint("Hello ", 23))
```

fmt.Fprint一类格式化的打印函数，可以接受任何实现了io.Writer接口的对象做为第一个实参。；
os.Stdout和os.Stderr都是我们熟知的函数。

从这里开始，就跟C有不同了。首先，数字格式，例如%d，不需要符号标识或者大小；输出的例程会根据实参的类型对你决定。

```go
var x uint64 = 1<<64 - 1
fmt.Printf("%d %x; %d %x\n", x, x, int64(x), int64(x))
```

打印 

```
18446744073709551615 ffffffffffffffff; -1 -1
```

如果你需要默认的转化，如十进制的整数，你可以用%v来包含所有的格式；其结果与Print和Println输出的完全相同。
而且，它还可以打印任何值，包括array、struct和map。以下是打印前一节定义的map类型：

```go
fmt.Printf("%v\n", timeZone)  // or just fmt.Println(timeZone)
```

它会输出：
```
map[CST:-21600 EST:-18000 MST:-25200 PST:-28800 UTC:0]
```

对于Map，Printf一类的函数，会把它的key按照字典排序输出。

当打印一个struct时，%+v会为struct的每个filed加上字段名，而%#v则会完全按照go的语法输出。

```go
type T struct {
    a int
    b float64
    c string
}
t := &T{ 7, -2.35, "abc\tdef" }
fmt.Printf("%v\n", t)
fmt.Printf("%+v\n", t)
fmt.Printf("%#v\n", t)
fmt.Printf("%#v\n", timeZone)
```

则会打印
```go
&{7 -2.35 abc   def}
&{a:7 b:-2.35 c:abc     def}
&main.T{a:7, b:-2.35, c:"abc\tdef"}
map[string]int{"CST":-21600, "EST":-18000, "MST":-25200, "PST":-28800, "UTC":0}
```

(注意&符号)。当遇到string或者[]byte时，用%q打印带引号的字符串；而%#q会尽可能使用反引号。
此外，%x还可用于string、[]byte以及整数，并生成一个很长的十六进制字符串， 而带空格的格式（% x）还会在字节之间插入空格。

另外一个常用的格式是 %T，它会打印值的类型。

```go
fmt.Printf("%T\n", timeZone)
``` 
 
会打印

```go
map[string]int
```

如果你希望控制一个定制类型的默认输出，只需要为该类型定义 String() string 的方法。
对于一个简单的类型T，它可能看起来是这样的：

```
func (t *T) String() string {
    return fmt.Sprintf("%d/%g/%q", t.a, t.b, t.c)
}
fmt.Printf("%v\n", t)
```

它会打印如下格式：

```go
7/-2.35/"abc\tdef"
```
 
如果你需要像指向T的指针那样打印类型T，那么String的receiver就必须是值类型的；
这个例子用指针是因为它更有效，而且符合struct的惯例。
下面的章节（pointer vs value receiver）会展示更多细节。

我们的String方法也可以调用Sprintf，因为我们的print例程是fully reentrant的，而且可以按照这种方式封装。
要理解这种方式，还有一个重要的细节：请不要通过Sprintf来构造String，因为它会无限地调用你的String方法。
当Sprintf尝试把receiver当作string来打印，它会反过来又调用Sprintf。
这是一个常见的错误，正如以下的例子：

```go
type MyString string

func (m MyString) String() string {
    return fmt.Sprintf("MyString=%s", m) // Error: will recur forever.
}
```

这个问题很容易解决，把实参转化成基本的string类型，这就没有了这个方法。

```go
type MyString string
func (m MyString) String() string {
    return fmt.Sprintf("MyString=%s", string(m)) // OK: note conversion.
}
```
 
在initialization节，我们会看到另一个方法去避免这样的循环调用。 

另一个关于打印的技术，是把一个打印例程的实参传入另外一个例程。
Printf的最后一个实参用了...interface{}来定义，这样它可以使用任意数量、任意格式的参数。

```go
func Printf(format string, v ...interface{}) (n int, err error) {
```

在函数Printf中，v像是[]interface{}类型的变量，但如果把它传到一个变参函数中，它就是一个实参列表了。
以下是我们之前用过的函数log.Println的实现。它直接把实参传给fmt.Sprintln来进行实际的格式化。

```go
// Println prints to the standard logger in the manner of fmt.Println.
func Println(v ...interface{}) {
    std.Output(2, fmt.Sprintln(v...))  // Output takes parameters (int, string)
}
```

在Sprintln的内嵌调中，我们把...写在v的后面，来告诉编译器，把v当作一个实参列表；否则，它会把v当作单slice的实参。

还有很多关于打印的知识点，可以从godoc文档的fmt包查找更多的细节。

另外，... 参数可以指定特定的类型，例如，可以用 ...int 来定义min函数的入参，表示从整型的列表里找最小的数字。

```go
func Min(a ...int) int {
    min := int(^uint(0) >> 1)  // largest int
    for _, i := range a {
        if i < min {
            min = i
        }
    }
    return min
}
```

##### append 

现在我们需要解释内建函数append的设计。append函数的signature不同于我们之前定义的函数Append。
简单来说，它像这样的：

```go
func append(slice []T, elements ...T) []T
```

这里的T是任何类型的占位符。事实上，在Go里没办法写一个类型T由调用者决定的函数。
这也就是为什么append是内建函数，因为它需要编译器的支持。

append函数的作用是，把元素放在slice的后面，并返回结果。
结果必须要返回，正如我们实现的Append函数一样，内部的array可能会改变。

以下简单的示例：

```go
x := []int{1,2,3}
x = append(x, 4, 5, 6)
fmt.Println(x)
```
 
它输出的是\[1 2 3 4 5 6\]。 因此， append像Printf，可以接受任何数量的实参。

但是，如果我们想把一个slice添加到另一个slice后面呢。在调用的地方使用 ...，就像我们上面调用Output一样。
以下代码输出的结果跟上面的一样。

```go
x := []int{1,2,3}
y := []int{4,5,6}
x = append(x, y...)
fmt.Println(x)
```

如果没有 ..., 它会编译不过，因为类型错误；y不是int类型。

### 九、初始化

尽管表面看起来，Go的初始化与C/C++并没有太大区别，但Go功能更加强大。
在初始化时，可以构造复杂的结果，而且它很好地解决了不同包的不同对象之间的初始化顺序的问题。

##### 常量
Go里面的常量是不变量。它们在编译的时候创建，即便它们可能是函数中定义的局部变量；它们只能是数字，字符（包括rune），字符串和布尔变量。
由于编译器的限制，它们的表达式必须是常量的表达式，能够被编译器求值计算。
例如，1<<3是一个常量表达式，而 math.Sin(math.Pi/4)则不是，因为它需要调用math.Sin 函数，而这只会在运行时才会发生。

在Go中，枚举类型可以用枚举器iota创建。
由于iota可以是表达式的一部分，而表达式可以被隐式地重复，这就可以更容易地创建复杂的值的集合。

```go
type ByteSize float64

const (
    _           = iota // ignore first value by assigning to blank identifier
    KB ByteSize = 1 << (10 * iota)
    MB
    GB
    TB
    PB
    EB
    ZB
    YB
)
```

由于可以将类似String这样的方法附加到任何用户定义的类型上，这就使得任何值都能提供自动化的打印的格式。 
尽管你看到的大多数例子都是struct类型，但它们同样可以用于类似浮点类型的标量类型，例如ByteSize。

```go
func (b ByteSize) String() string {
    switch {
    case b >= YB:
        return fmt.Sprintf("%.2fYB", b/YB)
    case b >= ZB:
        return fmt.Sprintf("%.2fZB", b/ZB)
    case b >= EB:
        return fmt.Sprintf("%.2fEB", b/EB)
    case b >= PB:
        return fmt.Sprintf("%.2fPB", b/PB)
    case b >= TB:
        return fmt.Sprintf("%.2fTB", b/TB)
    case b >= GB:
        return fmt.Sprintf("%.2fGB", b/GB)
    case b >= MB:
        return fmt.Sprintf("%.2fMB", b/MB)
    case b >= KB:
        return fmt.Sprintf("%.2fKB", b/KB)
    }
    return fmt.Sprintf("%.2fB", b)
}
```

表达式YB会打印成1.0YB,而ByteSize(1e13)会打印成9.09TB。
在这里用Sprintf去实现ByteSize的String方法很安全（避免了循环调用），这倒不是因为类型转换，而是因为它是用%f来调用Sprintf。
这不是string格式——Sprintf只会在它需要string的时候，才调用String方法；而%f则是需要一个浮点数值。

##### 变量
变量可以像常量一样被初始化，但其initializer可以是一个运行时才被计算的通用表达式。

```go
var (
    home   = os.Getenv("HOME")
    user   = os.Getenv("USER")
    gopath = os.Getenv("GOPATH")
)
```

##### 初始化函数 
最后，任何源文件都可以定义自己的无参数init函数来设置必要的状态。
实际上，每个文件都可以定义多个init函数。
init在所有定义在包里声明的变量通过它们的initializer求值之后才被调用；
而那些声明的变量是在所有导入的包被初始化之后才会计算。
（个人理解，就是：所有导入的包初始化->所有包声明的变量初始化->init函数调用）

除了不能通过声明来表示的初始化，init函数常用于在函数真正执行之前，检验或修正程序的状态。

```go
func init() {
    if user == "" {
        log.Fatal("$USER not set")
    }
    if home == "" {
        home = "/home/" + user
    }
    if gopath == "" {
        gopath = home + "/go"
    }
    // gopath may be overridden by --gopath flag on command line.
    flag.StringVar(&gopath, "gopath", gopath, "override default GOPATH")
}
```

### 十、方法 

##### Pointers vs. Values 

正如我们看到的ByteSize一样，我们可以为任何命名的类型（除了指针和接口）定义方法；receiver不要求一定是struct。

我们之前对slice讨论时，写了一个Append函数。我可以可以把它定义为slice的方法。
要做到这点，我们首先定义了一个命名的类型，这样我们就可以绑定方法；然后使得该方法的receiver成为该值的类型。

```go
type ByteSlice []byte

func (slice ByteSlice) Append(data []byte) []byte {
    // Body exactly the same as the Append function defined above.
}
```

这仍然要求方法返回更新后的slice。
为了消除这种不便，我们可以重新定义这个方法，通过一个指向ByteSlice的指针做为它的receiver，这样这个方法可以重写调用者提供的slice。

```go
func (p *ByteSlice) Append(data []byte) {
    slice := *p
    // Body as above, without the return.
    *p = slice
}
```

实际上，我们可以做的更好。我们可以修改我们的函数，使其看起来像一个标准的Write方法，就像这样：

```go
func (p *ByteSlice) Write(data []byte) (n int, err error) {
    slice := *p
    // Again as above.
    *p = slice
    return len(data), nil
}
```

这样，*ByteSlice满足标准的接口io.Writer，这样就很方便了。我们可以打印到该类型的变量中，

```go
 	var b ByteSlice
    fmt.Fprintf(&b, "This hour has %d days\n", 7)
```

我们传递的是ByteSlice的地址，因为只有 *ByteSlice 才满足 io.Writer。
以指针或者值做为receiver的区别在于，值的方法可以通过指针和值调用，而指针的方法只能通过指针调用。

之所以有这条规则是因为，指针方法可以修改receiver；对它们进行值调用，会导致方法收到一个值的copy，这样任何的修改都会失效。
因而，从语法上禁止了这个错误。不过，这有一个好用的例外，如果值是可以寻址的，那么，语法会自动插入寻址符来处理一般的通过值调用的指针方法。
在我们这个例子中，由于变量b是可以寻址的，因此我们可以通过b.Write来使用它的Write方法。
编译器会帮我们重写成(&b).Write。

顺便说一下，在字节slice上使用Write是 bytes.Buffer 实现的核心。 

### 十一、接口与其他类型

##### 接口 

Go的接口提供了一个方式去明确一个对象的行为：如果一样东西可以做这个，那么它就可以在这里使用（if something can do this, then it can be used here）。
我们已经看到了一些简单的例子；定制化的printer可以用一个String方法实现，而Fprintf可以为任何实现了Write方法的对象产生输出。
在go代码中，仅包含1、2个方法的接口很常见，其名称来源于这个方法，例如实现Write的 io.Writer 。

一个类型可以实现多个接口。
例如，一个实现了sort.Interface的集合可以通过sort包的例程进行排序。
该接口sort.Interface包含了Len(), Less(i, j int) bool, 和 Swap(i, j int)，并且有一个可定制化的formatter。
以下构建的例子Sequence满足了这两种情况：

```go
type Sequence []int

// Methods required by sort.Interface.
func (s Sequence) Len() int {
    return len(s)
}
func (s Sequence) Less(i, j int) bool {
    return s[i] < s[j]
}
func (s Sequence) Swap(i, j int) {
    s[i], s[j] = s[j], s[i]
}

// Copy returns a copy of the Sequence.
func (s Sequence) Copy() Sequence {
    copy := make(Sequence, 0, len(s))
    return append(copy, s...)
}

// Method for printing - sorts the elements before printing.
func (s Sequence) String() string {
    s = s.Copy() // Make a copy; don't overwrite argument.
    sort.Sort(s)
    str := "["
    for i, elem := range s { // Loop is O(N²); will fix that in next example.
        if i > 0 {
            str += " "
        }
        str += fmt.Sprint(elem)
    }
    return str + "]"
}
```

##### 类型转换

Sequence的String方法重新实现了Sprint为slice实现的功能。而且它的复杂度是 ![](http://latex.codecogs.com/gif.latex?\\O(N^2)) ，这个性能是很差的。 
如果我们在调用Sprint之前把Sequence转化成纯粹的[]int，我们就可以分享slice做的工作。

```go
func (s Sequence) String() string {
    s = s.Copy()
    sort.Sort(s)
    return fmt.Sprint([]int(s))
}
```

这是另一个例子，通过类型转换技术，安全地在String方法里调用Sprintf。
如果忽略类型的名字，这两个类型（Sequence和[]int）是同样的类型，因而在它们之前转换是合法的。
转换的过程并不会创建新的值，它知识使得现有的值看起来有个新的类型而已。
（还有一种合法转换会创造新的值，例如把一个整型转成浮点型）。

Go的惯例是，转换一个表达式的类型去访问不同的方法集。
我们可以用现有的类型sort.IntSlice，来简化整个示例：

```go
type Sequence []int

// Method for printing - sorts the elements before printing
func (s Sequence) String() string {
    s = s.Copy()
    sort.IntSlice(s).Sort()
    return fmt.Sprint([]int(s))
}
```

为了不让Sequence实现多个接口（排序和打印），我们可以把数据条目（data item）转化成多个类型（Sequence, sort.IntSlice 和 []int），每个类型做一部分工作。
这在实践中不常见，但它确实很有效。

##### 接口转化和类型断言

类型选择是类型转化的一种形式：它们接受一个接口，在每一个switch的case，一定程度上相当于把它转化成那个case的类型。
以下是fmt.Printf把一个值用类型选择把值转化成string的一个简化版本。
如果它是一个string，我们需要interface中包含的string的实际值；如果它有String方法，则需要获取调用该方法之后的结果值。

```go
type Stringer interface {
    String() string
}

var value interface{} // Value provided by caller.
switch str := value.(type) {
case string:
    return str
case Stringer:
    return str.String()
}
```

第一个情况获取具体的值；第二个情况，把interface转换成另一个interface。
像这样混合类型是没有问题的。

如果我们只关心其中一种类型呢。如果我们知道该值有一个string，而想要提取它。
只有一个类型的switch是可以的，不过也可以用类型断言。
一个类型断言会接收一个接口值，并从中提取一个特定的显式类型的值。
它的语法借鉴了类型选择的开头子句，但用了显式的类型，而不是类型的关键字。

```go
value.(typeName)
```

其返回值是静态类型typeName的新的值。
它的类型必须是该接口拥有的具体类型，或者是该值可以转化成的第二个接口类型。
从我们已知的值中提取string，我们可以写成：

```go
str := value.(string)
```

但如果那个值并不包含string，程序就会出现run-time error，并崩溃。
为了进行保护，可以用"comma, ok"的惯用法来安全地测试该值是否为string。

```go
str, ok := value.(string)
if ok {
    fmt.Printf("string value is: %q\n", str)
} else {
    fmt.Printf("value is not a string\n")
}
```

如果类型断言失败了，str仍然会以一个string类型存在，但它是0值，也就是一个空的string。

这里有一个if-else的语句，它等价于本节开头的类型选择（type switch）例子：
 
```go
if str, ok := value.(string); ok {
    return str
} else if str, ok := value.(Stringer); ok {
    return str.String()
}
``` 
 
##### 通用性

如果一个类型只是用来实现接口，除此之外，没有可以导出的方法，那就不需要导出这个类型。
仅导出接口，会更加清晰地表明，除了在接口描述的，该类型没有其他行为。
它避免了需要重复的文档去描述每个通用方法的实例。

在这种情况下，构造函数应该返回一个接口值而不是实现类型。
例如，在hash库里，crc32.NewIEEE和adler32.New都返回接口类型hash.Hash32。
在Go程序里，把CRC-32替换成Adler-32，仅需要改变构造函数的调用，其他代码则不受算法改变的影响。

这种方法可以使得，不同crypto包的streaming cipher算法从它们联系在一起的block ciphers中分离出来。
crypto/cipher包中的Block接口明确了block cipher的行为，它就是提供单block数据的加密。
通过与bufio包类似的方法，cipher包实现了这个接口，用于构造以Stream接口表示的streaming ciphers算法，而不需要知道块加密的细节。

crypto/cipher 接口看起来像这样的：

```go
type Block interface {
    BlockSize() int
    Encrypt(dst, src []byte)
    Decrypt(dst, src []byte)
}

type Stream interface {
    XORKeyStream(dst, src []byte)
}
```

以下是CTR Stream的定义，它把一个block cipher转换成streaming cipher。
注意， block cipher的细节已经被抽象化了。

```go
// NewCTR returns a Stream that encrypts/decrypts using the given Block in
// counter mode. The length of iv must be the same as the Block's block size.
func NewCTR(block Block, iv []byte) Stream
```

NewCTR的应用并不只是一个特定的加密算法和数据源，它适用于任何对Block接口的实现和Stream。
因为它们返回接口值，因此用其他加密模式替代CTR加密只需要做局部的更改。
构造函数的调用必须修改，但因为其他相关的代码只把它看作Stream，因此不会意识到其中的差别。

##### 接口和方法
由于任何类型都能添加方法，因而任何类型都能满足一个接口。
一个直观的例子是http包，它定义了Handler接口。任何对象实现了Handler接口，就可以服务一个HTTP请求。

```go
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}
```

ResponseWriter接口提供了对方法的访问，这些方法需要给客户端响应请求。
这些方法包括了标准的Write方法，因此一个http.ResponseWriter可以适用于任何io.Writer适用的场景。
Request结构体包含了已解析的客户端请求。

简单起见，我们先忽略POST请求，假设HTTP请求总是GET请求；这个简化并不影响建立handler方式。
这里有一个简短但完整的handler的实现，它计算网页被访问的次数。

```go
// Simple counter server.
type Counter struct {
    n int
}

func (ctr *Counter) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    ctr.n++
    fmt.Fprintf(w, "counter = %d\n", ctr.n)
}
```

(紧跟我们的主题，注意Fprintf如何打印到http.ResponseWriter)。
做为参考，这里是如何在URL tree的节点上添加一个服务。

```go
import "net/http"
...
ctr := new(Counter)
http.Handle("/counter", ctr)
```

为什么Counter要是一个结构呢？它只需要是一个整型。（receiver需要是一个指针，这样它的增量才能对调用者可见）。

```go
// Simpler counter server.
type Counter int

func (ctr *Counter) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    *ctr++
    fmt.Fprintf(w, "counter = %d\n", *ctr)
}
```

如果有一些内部的状态，需要在页面被访问之后得到通知。这就需要为页面绑定一个channel。

```go
// A channel that sends a notification on each visit.
// (Probably want the channel to be buffered.)
type Chan chan *http.Request

func (ch Chan) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    ch <- req
    fmt.Fprint(w, "notification sent")
}
```

最后，比方说，我们想在 /args 展示调用服务时使用的实参。可以很容易写一个函数来打印实参。
```go
func ArgServer() {
    fmt.Println(os.Args)
}
```

那么，我们如何把它加到一个HTTP的服务中呢。
我们可以把ArgServer写成某个可以忽略它的值的类型的方法。不过还有更简单的方法。
既然我们可以为除了指针和接口之外的任何类型定义一个方法，同样也可以为一个函数写方法。
http包有这样的代码：

```go
// The HandlerFunc type is an adapter to allow the use of
// ordinary functions as HTTP handlers.  If f is a function
// with the appropriate signature, HandlerFunc(f) is a
// Handler object that calls f.
type HandlerFunc func(ResponseWriter, *Request)

// ServeHTTP calls f(w, req).
func (f HandlerFunc) ServeHTTP(w ResponseWriter, req *Request) {
    f(w, req)
}
```

HandlerFunc是一个有ServeHTTP方法的类型，因此该类型的方法可以服务HTTP请求。
看该方法的实现：receiver是一个函数f，而该方法调用f。
它看起来很奇怪，但它跟receiver做为一个channel，而方法在channel发送数据并无区别。

要把ArgServer加入到HTTP服务，我们首先需要修改它，使其有个合适的signature：

```go
// Argument server.
func ArgServer(w http.ResponseWriter, req *http.Request) {
    fmt.Fprintln(w, os.Args)
}
```

ArgServer现在跟HandlerFunc有同样的signature，因而它可以转换成该类型去访问它的方法，正如我们转换Sequence成IntSlice去访问IntSlice.Sort。
实现代码很简洁：

```go
http.Handle("/args", http.HandlerFunc(ArgServer))
```

当用户访问网页 /args 时，安装在该页面的handler就有了值ArgServer和类型HandlerFunc。
HTTP服务会调用该类型的ServeHTTP方法，并以ArgServer做为receiver，它就会反过来调用ArgServer（通过调用HandlerFunc.ServeHTTP里的 invocation f(w, req)）
这样实参就会被展示。

在这一节里，我们通过struct、integer、channel和函数构建了一个http服务，这都归因于接口只是方法的集合，可以被定义为任何类型。


### 十二、空白标识符

我们在前面提过几次空白标识符。它可以被赋予或者是声明为任何类型的任何值，只要它的值被丢弃不会带来问题。
这有点像是写入到Unix的 /dev/null 文件：它表示write-only的值被用于占位--它需要一个变量，但实际的值并不相关。
除了我们前面看到的，它还有其他用途。

##### 多重赋值中的空白占位符 

在 for range loop里使用空白占位符是多重赋值的一个特例。

如果一个赋值需要在左侧有多个值，但其中一个值在程序中并不会被使用，这时，一个空白占位符就可以避免创建一个冗余的变量，而且可以很明确地显示，该值已经被丢弃了。
例如，调用一个函数返回值和错误，但我们只关心错误，这时可以用空白占位符来丢弃不相关的值。

```go
if _, err := os.Stat(path); os.IsNotExist(err) {
	fmt.Printf("%s does not exist\n", path)
}
```

有时你会看到代码丢弃了error值以忽略错误；这是一个很糟糕的实践。
请无比检查错误返回值；它们提供了错误的原因：

```go
// Bad! This code will crash if path does not exist.
fi, _ := os.Stat(path)
if fi.IsDir() {
    fmt.Printf("%s is a directory\n", path)
}
```

##### 没有使用的导入和变量

导入某个没有使用的包，或者是声明没有使用的变量，会导致错误。
未使用的导入会使代码臃肿，并降低编译速度；而初始化但没有使用的变量，至少会浪费计算资源，而且可能隐藏更大的bug。
当一个程序在编写时，可能会产生未使用的包和变量，但为了编译通过，不得不删除它们，即便之后可能又会再使用它们。
空白占位符提供了一个历史解决方案。

以下这个未完成的代码有2个未使用的导入（fmt和io），以及1个未使用的变量（fd），因此它编译未通过，但我们还是希望看到代码到目前为止是正确的。

```go
package main

import (
    "fmt"
    "io"
    "log"
    "os"
)

func main() {
    fd, err := os.Open("test.go")
    if err != nil {
        log.Fatal(err)
    }
    // TODO: use fd.
}
```

为了停止编译器对未使用的导入的错误报告，可以使用空白占位符去引用已导入的包的符号。
类似的，把未使用的变量赋值为空白占位符可以停止未使用变量的错误报告。
以下这个版本可以编译通过：

```go
package main

import (
    "fmt"
    "io"
    "log"
    "os"
)

var _ = fmt.Printf // For debugging; delete when done.
var _ io.Reader    // For debugging; delete when done.

func main() {
    fd, err := os.Open("test.go")
    if err != nil {
        log.Fatal(err)
    }
    // TODO: use fd.
    _ = fd
}
```

按照惯例，为停止提示导入错误（silence import error）而做的全局声明，应该紧接在导入声明的后面，并加上注释，这样可以更容易地发现，并在随后清除它。

##### 副作用式导入

像前面例子里，未使用的包fmt和io最终会被使用或者移除：使用空白占位符只是表明代码还在完成中。
但有时导入某个包只是为了其副作用，而没有任何显示的使用。
例如，net/http/pprof包会在它的init函数中，注册HTTP的handler来提供调试信息。
它有可导出的API，但大多数客户端只需要handler registration，并通过web页面访问数据。
为了实现仅为副作用而导入包的操作，可以把包重命名为空白占位符：

```go
import _ "net/http/pprof"
```

这个导入包的形式明确表明包是为其副作用导入的，因为没有任何其他可能会用到这个包：在这个文件里，它没有名字。（如果它有的话，而我们又没有使用它，编译器就会拒绝）

##### 接口检查 
正如我们在前面对接口的讨论，一个类型无需显式地声明它实现了某个接口。相反地，一个类型只要实现了一个接口的方法，它就实现了这个接口。
在实践中，很多接口的转换是静态的，因而在编译时候进行校验。例如，传 *os.File 到入参为 io.Reader 的函数则不会编译通过，除非 *os.File 实现了接口 io.Reader 。

一些接口的检验也会发生在run-time。其中一个例子是  encoding/json 包，它定义了一个 Marshaler 接口。
当JSON的encoder接受到一个实现了该接口的值，encoder会调用该值的Marshal方法去转换JSON，而不是用标准的转换器。
而encoder会在运行时通过类型断言验证其属性：

```go
m, ok := val.(json.Marshaler)
```

如果它只是想知道一个类型是否实现了接口，而不需要用到接口本身，那就使用空白占位符去忽略类型断言的值。

```go
if _, ok := val.(json.Marshaler); ok {
    fmt.Printf("value %v of type %T implements json.Marshaler\n", val, val)
}
```

当需要确定某个包里的实现的类型满足接口时，就会出现这种情况。如果一个类型，如json.RawMessage需要一个定制化的json展示，它需要实现json.Marshaler，但现在没有静态的转换器让编译器自动去验证它。
如果一个类型无意地没有满足接口，JSON encoder仍然会工作，只是不会用到定制化的实现。
为了保证实现是正确的，可以在包中用空白占位符做一个全局声明，

```go
var _ json.Marshaler = (*RawMessage)(nil)
```

在这个声明中，赋值操作包括了把 *RawMessage 转换成 Marshaler，而这需要 *RawMessage 实现 Marshaler，这会导致在编译阶段就进行检查。
当json.Marshaler接口改变时，这个包就会编译不通过，我们就会注意到它需要更新了。

在这个构造器里出现空白占位符，只是为了表明，该声明仅仅只是为了类型校验存在，并不产生新的变量。
但不要对所有满足接口的类型都做这样的处理。
通常，这样的声明只用于代码中不存在静态类型转换（static conversion）时，而这种情况其实是很少见的。

十三、嵌入 

Go不提供典型的、类型驱动的子类概念，但通过内嵌到结构体或接口中，来"借鉴"部分实现。

内嵌接口非常简单。我们前面提到的 io.Reader 和 io.Writer i，以下是它们的定义：

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}
```

在io包中，还定义了几个其他的接口，它们定义了可以实现几个不同方法的对象。
例如，对于io.ReadWriter，一个接口包括Read和Write。
我们可以通过显式地列举两个方法来定义io.ReadWriter，但更简单而直观的方式是，把这两个接口嵌入到一个新的接口中，像这样：

```go
// ReadWriter is the interface that combines the Reader and Writer interfaces.
type ReadWriter interface {
    Reader
    Writer
}
```

正如它看起来的那样：ReadWriter可以同时做Read和Write的事情；它是内嵌接口的联合体（它们必须是不相交的方法集）。
只有接口才能跟接口内嵌。

同样的想法适用于结构，但实现更加复杂一些。bufio包有两个结构类型bufio.Reader和 bufio.Writer，它们都实现了io包的类似接口。
bufio还实现了一个缓冲的reader/writer，它的做法是，把reader和writer嵌入到接口中，在结构中列举类型名，但不给它们域的名称。

```go
// ReadWriter stores pointers to a Reader and a Writer.
// It implements io.ReadWriter.
type ReadWriter struct {
    *Reader  // *bufio.Reader
    *Writer  // *bufio.Writer
}
```

内嵌的元素为指向结构体的指针，它们会在使用前指向有效的结构。 ReadWriter也可以写成：

```go
type ReadWriter struct {
    reader *Reader
    writer *Writer
}
```

但为了使各filed对应的方法能够满足io接口规范，我们还需要提供如下forwarding方法：

```go
func (rw *ReadWriter) Read(p []byte) (n int, err error) {
    return rw.reader.Read(p)
}
```

通过直接嵌入到结构，我们就可以避免这样的繁琐。
嵌入类型的方法可以不受约束地被使用，这意味这，bufio.ReadWriter不仅可以使用bufio.Reader和bufio.Writer，它还同样满足这三个接口：io.Reader,io.Writer 和 io.ReadWriter。

内嵌和子类（subclassing）还有一个重要的区别。
当我们嵌入一个类型，它的方法会成为外部类型（outer type）的方法，当它们被调用时，它们的receiver是内部类型，而不是外部类型。
在我们的例子中，当bufio.ReadWriter的Read方法被调用时，它与之前的forwarding方法有同样的效果；receiver是ReadWriter的reader域，而不是ReadWriter本身。


嵌入还有一个简单的便利，以下例子显示如何把嵌入的域和常规的命名的域放在一起：

```go
type Job struct {
	Command string
	*log.Logger
}
```

Job类型现在有了*log.Logger的Print、Printf、Println以及其他方法。我们当然可以给Logger一个域的名字（filed name），但这没有必要。
现在，只要初始化后，我们就可以在job上使用log。

```go
job.Println("starting now...")
```

Logger是Job结构的常规的域，因而我们可以在构造器里用常规的方式初始化它，就像这样：

```go
func NewJob(command string, logger *log.Logger) *Job {
    return &Job{command, logger}
}
```

或者用composite literal的方式，

```go
job := &Job{command, log.New(os.Stderr, "Job: ", log.Ldate)}
```

如果我们需要直接引用内嵌的域，可以忽略包限定名，把该字段的类型名做为字段名，就像我们在ReadWriter里使用Read方法一样。
在这里，如果像访问Job的变量job的*log.Logger，我们可以写成job.Logger,这方便我们要优化Logger的方法：

```go
func (job *Job) Printf(format string, args ...interface{}) {
    job.Logger.Printf("%q: %s", job.Command, fmt.Sprintf(format, args...))
}
```

内嵌类型会引入命名冲突的问题，但解决它们的规则其实很简单。首先，字段或者方法X会隐藏任何其他的更深嵌套的项目X。
如果log.Logger有一个域或方法叫Command，Job里的Command会覆盖它。

其次，如果同名出现在相同的嵌套层级，这通常会产生一个错误；如果Job包含一个域或者方法叫Logger，在嵌入log.Logger的时候就会产生错误。
当然，如果重复的名字不会在类型定义之外的代码中出现，那就不会出错。
这个限定对在外部对嵌入类型进行修改时提供了保护；因此，如果一个加入的域与其他subtype的域发生冲突，但只要这两个域没有使用到，那就没问题。


十四、并发

并发变成是一个宏大的主题，在这里我们只介绍一些Go语言独有的亮点。

并发在很多环境下都很困难，因为它需要精巧的实现以正确地访问贡献变量。
Go提供了一种新的方式，它将共享的值通过channel来传递--事实上，并没有在不同执行线程间共享数据。
在任何时间，只有一个goroutine能够访问到该值。在设计上杜绝了数据竞争。
为了鼓励这种思考方式，我们把它简化成slogan：

```
Do not communicate by sharing memory; instead, share memory by communicating.
```

这个方式意义深远。例如，引用计数可以通过给一个整数变量添加互斥锁来实现。
但在更高的层次，通过channel来控制访问，能更好地写出清晰正确的代码。

我们可以用这个方法来分析上述的模型：试想一个单线程的程序运行在一个CPU上。它不需要同步原语。
现在再加入一个线程，它同样也不需要同步。现在让它们通信；如果通信是同步原语，那么它仍然不需要其他的同步机制。
Unix的管道机制就很适合这种模型。尽管Go的并发处理方式来源于Hoare的通信顺序进程（CSP，Communicating Sequential Processes），它可以看作是Unix的管道机制的类型安全的通用化实现。

##### Goroutine
之所以叫做goroutine是因为，现存的术语，线程、协程、进程等都无法准确地表达。
goroutine有一个简单的模型：它是一个与其他goroutine运行在相同地址空间的函数。
它是轻量级的，消耗的时间比栈的空间分配多不了多少。
而栈的初始分配是很小的，而之后它们才会随着需要的堆空间而分配或释放。

Goroutine在多线程操作系统上是多路复用的，所以如果一个goroutine阻塞了，例如在等待IO，其他的仍然会继续运行。
它们的设计隐藏了很多复杂的线程创建和管理。

在一个函数或方法前面加上关键词go做为前缀，则会在新的goroutine中调用该函数。
当调用结束之后，goroutine也就会安静地退出。这有点类似Unix的shell通过&符号让一个命令在后台运行。

```go
go list.Sort()  // run list.Sort concurrently; don't wait for it.
```

一个function literal在goroutine调用中很好用。

```go
func Announce(message string, delay time.Duration) {
    go func() {
        time.Sleep(delay)
        fmt.Println(message)
    }()  // Note the parentheses - must call the function.
}
```
 
在Go里，function literals是闭包的：它的实现保证了，函数引用的变量在函数结束之前不会被释放。

这些例子并不是很实用，因为执行函数没办法发送其完成的信号。因此，我们需要channel。

##### Channel 

跟Map一样，channel是通过make来分配的，其返回值实际上是一个对底层数据结构的引用。
如果填写了可选的整型参数，它会为channel设定缓冲区的大小。
它的默认值是0，表示没有缓冲区或者是同步的channel。

```go
ci := make(chan int)            // unbuffered channel of integers
cj := make(chan int, 0)         // unbuffered channel of integers
cs := make(chan *os.File, 100)  // buffered channel of pointers to Files
```

无缓冲的channel使得通信（也就是值的交换）和同步机制结合，它保证两个goroutine处于可控的状态。

有很多关于channel的惯例。以下是其中一个。
在前面的例子中，我们在后台启动了一个排序的程序。而channel可以让goroutine等待排序完成。
```go
c := make(chan int)  // Allocate a channel.
// Start the sort in a goroutine; when it completes, signal on the channel.
go func() {
    list.Sort()
    c <- 1  // Send a signal; value does not matter.
}()
doSomethingForAWhile()
<-c   // Wait for sort to finish; discard sent value.
```

接受者在收到数据之前会一直阻塞。如果channel是不带缓冲的，则发送者会一直阻塞，直到接受者拿到了数据。
如果channel是带缓冲的，发送者只是在数据拷贝到缓冲区时才阻塞。如果缓冲区满了，这意味着它会等到接受者取一个值。

带缓冲区的channel可以看作是信号量，例如限制吞吐量。
在这个例子中，请求会进入到handle，它会发送一个值到channel，处理请求，然后从channel接受一个值，表示准备好了下一次请求消费。
channel的buffer限制了同时调用请求的数量。

```go
var sem = make(chan int, MaxOutstanding)

func handle(r *Request) {
    sem <- 1    // Wait for active queue to drain.
    process(r)  // May take a long time.
    <-sem       // Done; enable next request to run.
}

func Serve(queue chan *Request) {
    for {
        req := <-queue
        go handle(req)  // Don't wait for handle to finish.
    }
}
```

一旦有 MaxOutstanding 个handle在执行处理，其他的都会阻塞在给一个满的channel缓冲区发送数据，直到一个handle完成处理，并从channel上取走一个值。

这个设计有个执行的问题，Serve为每个进来的请求都创建了新的goroutine，但实际上只有 MaxOutstanding 个可以同时运行。
因此，如果请求来的太快的话，程序会消耗无限的资源。 要解决这个缺陷，我们可以通过修改Serve来限制对goroutine的创建。
以下是一个简单的实现，它有一个bug，我们会在后面修复它。

```go
func Serve(queue chan *Request) {
    for req := range queue {
        sem <- 1
        go func() {
            process(req) // Buggy; see explanation below.
            <-sem
        }()
    }
}
```

这个bug出现在go的for循环的实现，循环的变量会在每次迭代中被重新使用，因此req变量实际上是所有goroutine共享的。
这不是我们想要的。我们希望的是每个goroutine的req都是唯一的。
下面是一种解决办法，把req做为实参传入到该goroutine的闭包中。

```go
func Serve(queue chan *Request) {
    for req := range queue {
        sem <- 1
        go func(req *Request) {
            process(req)
            <-sem
        }(req)
    }
}
```

对比这两个版本，观察闭包的声明和运行。另一个实现是，创建一个同名的新变量，如以下代码：

```go
func Serve(queue chan *Request) {
    for req := range queue {
        req := req // Create new instance of req for the goroutine.
        sem <- 1
        go func() {
            process(req)
            <-sem
        }()
    }
}
```

可能写成这样会很奇怪：

```go
req := req
```

但在Go里，这样是合法而且惯用的。你获得了一个同名的新变量，为每个goroutine创建新的私有的拷贝。

回到实现通用的server问题上来，另一个管理资源的方法就是，开启固定数量的handle的goroutine，它们都读取请求的channel。
goroutine的数量就决定了同时调用处理请求的数量。Server会接收一个通知退出的channel；在启动goroutine的时候它会停止接收channel。

```go
func handle(queue chan *Request) {
    for r := range queue {
        process(r)
    }
}

func Serve(clientRequests chan *Request, quit chan bool) {
    // Start handlers
    for i := 0; i < MaxOutstanding; i++ {
        go handle(clientRequests)
    }
    <-quit  // Wait to be told to exit.
}
```

##### Channels of channels 

Go有一个很重要的特性是，channel是一个first-class的值，它可以像其他值一样被分配和传递。
该特性的一个常见的用法是，实现安全并行的demultiplexing。

在前一节的例子中，handle是一个理想化的处理请求处理程序，但我们并没有定义它处理的类型。
如果该类型包含一个用于回复的channel，则每个客户端都可以提供它们各自的应答方式。
以下是类型Request的大致定义：

```go
type Request struct {
    args        []int
    f           func([]int) int
    resultChan  chan int
}
```

客户端提供了一个函数和他的实参，以及一个内部的channel来接收应答的消息。

```go
func sum(a []int) (s int) {
    for _, v := range a {
        s += v
    }
    return
}

request := &Request{[]int{3, 4, 5}, sum, make(chan int)}
// Send request
clientRequests <- request
// Wait for response.
fmt.Printf("answer: %d\n", <-request.resultChan)
```

在服务端，则只需要改变handle函数。

```go
func handle(queue chan *Request) {
    for req := range queue {
        req.resultChan <- req.f(req.args)
    }
}
```

使其可用还有很多工作要做，但这个代码是一个频率控制、并行、非阻塞RPC的系统框架，而且它们不包含互斥锁。

##### 并行化

这些设计的另一个应用是在多核CPU上实现并行计算。
如果计算能够分解成多片，它就可以相互独立地执行，也就是并行，并通过channel来控制每一片都完成了。

假设我们要在数组上进行比较耗时的操作，而且每一项的值的计算是相互独立的，正如以下这个理想化的例子：

```go
type Vector []float64

// Apply the operation to v[i], v[i+1] ... up to v[n-1].
func (v Vector) DoSome(i, n int, u Vector, c chan int) {
    for ; i < n; i++ {
        v[i] += u.Op(v[i])
    }
    c <- 1    // signal that this piece is done
}
```

我们在循环中启用的独立的处理块，每块占一个CPU。
它们可以按任何顺序结束，但这不重要；我们只要在所有goroutine完成之后统计完成信号就可以。

```go
const numCPU = 4 // number of CPU cores

func (v Vector) DoAll(u Vector) {
    c := make(chan int, numCPU)  // Buffering optional but sensible.
    for i := 0; i < numCPU; i++ {
        go v.DoSome(i*len(v)/numCPU, (i+1)*len(v)/numCPU, u, c)
    }
    // Drain the channel.
    for i := 0; i < numCPU; i++ {
        <-c    // wait for one task to complete
    }
    // All done.
}
```

我们可以通过runtime查看CPU数量，而不是指定一个常量。函数runtime.NumCPU返回CPU硬件的核数量。

```go
var numCPU = runtime.NumCPU()
```

还有一个函数runtime.GOMAXPROCS，它会返回user-specified的核的数量，也就是一个go程序可以同时运行的数量。
它的默认值是runtime.NumCPU，但可以用shell环境相似命名的变量重新设置它，或者传入一个正数来调用它。
用0来调用它等价于查询。因此，如果我们希望尊重用户的资源请求，我们应该写成：

```go
var numCPU = runtime.GOMAXPROCS(0)
```

注意，不要混淆并发和并行，并发是用独立执行的模块来构建程序，而并行是将任务在多个CPU上执行以提高效率。
尽管go的一些并发的特性很容易把问题构造成并行计算，但go是一个并发的语言，而不是并行的语言，并不是所有的并行语言都适用于go的模型。
要进一步讨论它们的区别，可以看这个[blog](https://blog.golang.org/concurrency-is-not-parallelism)。



##### 一个“Leaky Buffer”的示例

并发变成的工具甚至很容易表达非并发的思想。以下是一个从RPC包提取的代码。
客户端goroutine从某来源循环获取数据，比方说从网络。为了避免分配和释放缓冲区，它维护了一个空闲链表，并用带缓冲的channel进行封装。
如果channel为空，则新的缓冲区会获得分配。当消息缓冲区准备好，它会把通过serverChan把消息发送到服务端。

```go
var freeList = make(chan *Buffer, 100)
var serverChan = make(chan *Buffer)

func client() {
    for {
        var b *Buffer
        // Grab a buffer if available; allocate if not.
        select {
        case b = <-freeList:
            // Got one; nothing more to do.
        default:
            // None free, so allocate a new one.
            b = new(Buffer)
        }
        load(b)              // Read next message from the net.
        serverChan <- b      // Send to server.
    }
}
```

服务端循环从客户端接收消息，进行处理，并把buffer回收给空闲链表。

```go
func server() {
    for {
        b := <-serverChan    // Wait for work.
        process(b)
        // Reuse buffer if there's room.
        select {
        case freeList <- b:
            // Buffer on free list; nothing more to do.
        default:
            // Free list full, just carry on.
        }
    }
}
```

客户端从freeList接收缓冲区数据；如果没有，它就会分配一个新的的。
服务端将b放回freeList，除非链表已经满了，此时buffer会被丢弃，并被GC回收。
（在select语句的default子句会在没有其他条件匹配时执行，意味这select永远不会阻塞）。
利用缓冲区channel和GC机制，我们用几行代码就实现了一个有漏洞的free list。


### 十三、错误

Library routines必须经常向调用者返回错误的提示。
正如前面提到的，go的多返回值的特性，使得它在返回正常的返回值时，还可以顺带返回错误信息的详尽描述。
利用这个特性来返回详细的错误信息是个很好的风格。
例如，os.Open 在失败时不会返回nil指针，而是返回一个错误值，描述哪里出错了。

按照约定，错误的类型是error，这是一个内建的接口。

```go
type error interface {
    Error() string
}
```

库的开发者可以通过这个接口来丰富它的功能，使其不仅能够看到错误，还能提供一些上下文。
正如之前提到的，除了常见的 *os.File 返回值，os.Open还返回了错误值。
如果一个文件被成功打开，其error则是nil；但如果有错误，则它会带一个os.PathError：

```go
// PathError records an error and the operation and
// file path that caused it.
type PathError struct {
    Op string    // "open", "unlink", etc.
    Path string  // The associated file.
    Err error    // Returned by the system call.
}

func (e *PathError) Error() string {
    return e.Op + " " + e.Path + ": " + e.Err.Error()
}
```

PathError的Eror函数会产生一个类似这样的字符串：

```go
open /etc/passwx: no such file or directory
```

这条错误包含了造成错误的文件名、操作和触发的操作系统错误，即便它打印的位置距离触发它的调用很远，它也是很有用的；它比仅仅打印"文件不存在"包含了更多信息。

如果可能，错误字符串应该指出它们的来源，例如造成错误的操作或包的前缀名。
例如，在包image里，解码未知格式的错误，其错误字符串则展示为："image: unknown format"。

调用者如果关心具体的错误信息，可以用类型选择或类型断言去寻找具体的错误，并提取细节。
对于PathErrors，这包括检验其内部的域Err，以便进行错误恢复。

```go
for try := 0; try < 2; try++ {
    file, err = os.Create(filename)
    if err == nil {
        return
    }
    if e, ok := err.(*os.PathError); ok && e.Err == syscall.ENOSPC {
        deleteTempFiles()  // Recover some space.
        continue
    }
    return
}
```

这里第二个if语句是另一个类型断言。如果它失败了，ok会是false，e则为nil。
如果它成功了，ok为true，这就意味着错误是 *os.PathError 类型，我们可以对这个错误e，进行更多的错误类型校验。

##### Panic 

常用的给调用者报告错误信息的方式是把error做为一个额外的返回值。
Read方法就是一个很好的例子；它会返回byte的计数和错误。但如果这个错误是不可恢复的呢？
有时，这个程序就是无法继续执行了。

出于这个目的，有一个内建的函数panic，它会产生一个run-time的错误，并终止程序（看下一节）。
这个函数接收一个任意类型的实参（通常为字符串），并在程序终止时打印。
它能表明发生了一些意料之外的事情，例如出现了死循环。

```go
// A toy implementation of cube root using Newton's method.
func CubeRoot(x float64) float64 {
    z := x/3   // Arbitrary initial value
    for i := 0; i < 1e6; i++ {
        prevz := z
        z -= (z*z*z-x) / (3*z*z)
        if veryClose(z, prevz) {
            return z
        }
    }
    // A million iterations has not converged; something is wrong.
    panic(fmt.Sprintf("CubeRoot(%g) did not converge", x))
}
```

这仅仅只是一个示例，而真正的库函数应该避免panic。如果问题可以被屏蔽或者解决，最好就是让程序继续执行，而不是终止它。
一个反例是初始化：如果一个库真的不能完成创建，合理的做法是让其panic，并讲出来。

```go
var user = os.Getenv("USER")

func init() {
    if user == "" {
        panic("no value for $USER")
    }
}
```

##### Recover

当一个panic被调用时，包括显式的错误，如数组越界，或者是类型断言失败，它会立刻停止执行当前的函数，并开始回溯（unwinding）goroutine的调用栈，并运行defer的函数。
如果回溯到goroutine栈的顶部，程序就会结束。但是，我们可以利用内建函数Recover去重新获得goroutine的控制权，并继续正常的执行。

调用recover会停止回溯，并返回传给panic的实参。由于在回溯过程中只会运行defer函数里面的代码，因而recover只有在defer中才生效。

recover的一个应用是在server中停止一个失败的goroutine，但不杀死其他的goroutine。

```go
func server(workChan <-chan *Work) {
    for work := range workChan {
        go safelyDo(work)
    }
}

func safelyDo(work *Work) {
    defer func() {
        if err := recover(); err != nil {
            log.Println("work failed:", err)
        }
    }()
    do(work)
}
```

在这个例子中，如果 do(work) 出现panic，其结果会被日志记录，其goroutine会自动结束，而不会干扰其他的goroutine。
我们不需要在defer的闭包里做任何事情，调用recover会帮你完全搞定。 

由于recover总是返回nil，除非它直接从defer函数里调用，因此defer函数的代码可以调用本身使用了panic和recover的代码，而不会失败。
例如，defer函数 safelyDo 可能在调用recover之前调用了一个日志的函数，该日志函数不会受到panic状态的影响。

通过恰当地使用recovery模式，do函数可以通过调用panic来避免更坏的情况。
我们可以用这个思想在复杂的软件中去简化错误处理。
我们来看以下regexp包的理想版本，它会通过一个本地的类型错误，来调用panic去通知解析错误。
以下是Error的定义，error方法，以及Compile函数：

````go
// Error is the type of a parse error; it satisfies the error interface.
type Error string
func (e Error) Error() string {
    return string(e)
}

// error is a method of *Regexp that reports parsing errors by
// panicking with an Error.
func (regexp *Regexp) error(err string) {
    panic(Error(err))
}

// Compile returns a parsed representation of the regular expression.
func Compile(str string) (regexp *Regexp, err error) {
    regexp = new(Regexp)
    // doParse will panic if there is a parse error.
    defer func() {
        if e := recover(); e != nil {
            regexp = nil    // Clear return value.
            err = e.(Error) // Will re-panic if not a parse error.
        }
    }()
    return regexp.doParse(str), nil
}
````

如果 doParse 出现panic，recovery代码块会把返回值置为nil，defer函数可以修改已经命名的返回值变量。
它会在给err赋值时进行校验，通过类型断言确认它是一个本地的类型Error。
如果它不是，类型断言会失败，触发一个run-time的错误，然后继续回溯栈，此时它不会再中断了。
这个校验意味着，如果一些异常发生，例如数组越界，代码依然会失败，即便我们用panic和recover来处理解析错误。

通过合适的处理错误，error方法会很容易地报告解析错误，而不需要关心手动处理回溯的解析的栈：

```go
if pos == 0 {
    re.error("'*' illegal at start of expression")
}
```

尽管这个模式很有用，但它只能在包里面使用。Parse会把它内部的panic调用转化为error值；它不会把panic暴露给客户端。
这是一个值得遵守的好规则。

顺便说一下，这个re-panic用法会在错误真正发生时改变panic的值。
然而，不会是原始的还是新的错误，都会在crash报告里出现，所以，导致错误出现的根源仍然可见。
因此，这种简单的re-panic已经够用了，毕竟这是一个crash。
但如果你只想显示原始的错误值，你可以写代码去过滤不希望出现的错误，然后用原始值re-panic。
那就把这个练习留给读者了。

### 十四、一个web服务

让我们以一个web的服务来做为结束吧。这实际上是一个web re-server。
google提供了一个服务在chart.apis.google.com，它可以把数据转换成表格或者是图表。
它很难通过交互的方式使用，因为你需要把数据放到URL上做为请求。
我们的程序提供了一个数据格式的接口：给定一个简短的text，它会调用图表服务来生成QR码--一个对文本进行编码的二维方框。
这个图片能用手机摄像头进行抓取，并解析为URL，节省你在手机上输入URL的时间。

这是一个代码，后面会给出解释。

```go
package main

import (
    "flag"
    "html/template"
    "log"
    "net/http"
)

var addr = flag.String("addr", ":1718", "http service address") // Q=17, R=18

var templ = template.Must(template.New("qr").Parse(templateStr))

func main() {
    flag.Parse()
    http.Handle("/", http.HandlerFunc(QR))
    err := http.ListenAndServe(*addr, nil)
    if err != nil {
        log.Fatal("ListenAndServe:", err)
    }
}

func QR(w http.ResponseWriter, req *http.Request) {
    templ.Execute(w, req.FormValue("s"))
}

const templateStr = `
<html>
<head>
<title>QR Link Generator</title>
</head>
<body>
{{if .}}
<img src="http://chart.apis.google.com/chart?chs=300x300&cht=qr&choe=UTF-8&chl={{.}}" />
<br>
{{.}}
<br>
<br>
{{end}}
<form action="/" name=f method="GET"><input maxLength=1024 size=70
name=s value="" title="Text to QR Encode"><input type=submit
value="Show QR" name=qr>
</form>
</body>
</html>
`
```

main函数之前的代码很容易理解。flag设定了我们服务的默认HTTP端口。
模版变量templ则开始了有趣的部分，它建立了一个HTML模版，该模版被我们的服务执行以展示网页；我们后面会看到更多细节。

main函数解析flag标志，并用我们之前讨论过的机制，把QR函数绑定到我们服务的根路径。
调用http.ListenAndServe则开始服务，它会在服务运行过程中，处于阻塞状态。

QR接收包含格式化数据的请求，并以格式化s为参数对模版进行实例化。

html/template包功能非常强大，这个程序只是涉及到其冰山一角。
本质上，它会根据传入templ.Execute的数据，替换相应的元素，并重新生成HTML文本。
在 {{if .}} 和 {{end}}  之间的代码，只有在当前数据项，也就是 .(dot) 不为空时，才执行。
也就是，当字符串为空时，这部分模版代码将会被省略。

两个snippet {{.}} 表示把传给模版的数据展示在页面上。 HTML模版会自动对文本进行转义，因而文本的展示是安全的。

剩下的就只是页面加载时需要的HTML字符串了。如果这对你来说太笼统了，可以通过[文档](https://golang.org/pkg/html/template/)来获取详尽的讨论。

现在，你仅仅通过几行代码和一些data-driven的HTML文本，就实现了一个有用的web服务。
Go强大到用几行代码就可以解决很多事情。 

