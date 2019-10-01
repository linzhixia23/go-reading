原文：https://golang.org/doc/effective_go.html

<br>

#### 介绍
Go是一门全新的语言。虽然它从现有的语言借鉴了一些理念，但它的语言特性，使得编写高效的go程序与其他语言完全不同。
直接把现有的C++或Java程序翻译成go，结果并不理想--毕竟Java程序不是go。
另一方面，如果从go的视角去思考，则能够写出更好的、不一样的代码。
换句话说，要想把go程序写好，需要深入理解它的特性和风格。
同时，还要了解go的既定规则，包括命名规范、格式、代码结构等，这样你的代码才更有可读性。

这篇文档介绍一些tips，帮助编写清晰地道的go代码。它是对之前文档的补充说明，因此建议先阅读这些文档。

##### 示例
go源码包不仅提供了核心库，同时还有一些例子示范如何使用go语言。
此外，很多库包含了可执行的示例。
如果你有关于一些问题如何解决，或者如何实现的疑问，代码库的文档、代码和示例都能够提供解答、思想和背景。


#### 格式
格式问题经常引起争议，而且很少能讨论出结果。尽管码农能适应各种编码风格，但如果没有岂不是更好。倘若大家都使用同样的风格，那么在这上面花的时间就越少。问题在于，如何在没有强制的风格指引的条件下，实现这个乌托邦。

在go里面，我门采取了不同的方式--通过让机器来解决大多数的格式问题。
*gofmt*程序把go代码按照标准风格缩进、对齐，保留注释、并在必要的是否重新优化它们的格式。
如果你想知道如何处理新的代码布局，请运行*gofmt*；如果结果不尽如人意，可以重新组织你的程序（或者提交bug单到*gofmt*），而不需要为此纠结。

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
  Go没用行长度的限制，不要担心打点机溢出。如果觉得一行太长，也可以换行，并插入tab。
  
##### 括号
  Go比C和Java的行数更少：控制结构（if，for，switch）在语法上不需要括号。因此操作符优先级更加简洁，"x<<8 + y<<16" 表示空格分割表达的意思，而不像其他语言。 
   

#### 注释
Go支持C-风格的块注释/**/，以及C++风格的行注释//。行注释比较常见；块注释常见于包的注释，同时对于大段代码的注释也比较管用。

*godoc*程序是个web服务，能够从go的源代码中提取包的注释。注释通常出现在top-level的声明位置，与声明之间没有空行。
它会跟声明一起被提取出来，作为该条目的说明文档。这些注释的风格和类型，决定了*godoc*生成文档的质量。

每个包都应该有一个包注释，即一段在包子句（package clause）前面的块注释。
对于有多个文件的包，包注释只要放在其中任意一个文件就可以。
包注释应该介绍包，以及包的相关信息。它会出现在*godoc*页面的最前面，并为其后的内容建立文档。

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
*godoc*会像*gofmt*把这个做好。注释是不会被解析的纯文本，像HTML格式的*_this_*，则会按原样输出，所以最好不要这样使用。
*godoc*做的另一个调整，就是把缩进文本按等宽字体调整，更好地适应snippet。*gofmt*包就用了包注释的方式。

*godoc*是否重新格式化注释取决于上下文，因而要保证它们看起来清晰明了：正确的拼写、标点和句式结构，折叠长行等。

在一个包里，任何top-level前面的注释，都会做为其注释文档。每个可以导出的命名，都应该有注释文档。

注释文档最好是完整的句子，这样才能适应自动化的展示。它最好是一句话的总结，并且以被申明的名称做为第一个词。

```
// Compile parses a regular expression and returns, if successful,
// a Regexp that can be used to match against text.
func Compile(str string) (*Regexp, error) {
```

如果每个注释文档都以名称开头，你可以用go tool的doc子命令，并对输出进行grep。
试想，你记不住名称"Compile"，但又想查找正则表达式的解析函数，你可以运行如下命令：

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

分组还可以用来指示各指标之间的关系，例如一组由互斥变量进行保护的变量。
```go
var (
    countLock   sync.Mutex
    inputCount  uint32
    outputCount uint32
    errorCount  uint32
)
```

#### 命名 
跟其他语言一样，命名在Go里是很重要的。它们甚至还会影响语义：一个命名的首字母是否大写，影响到它在包外是否可见。
因此，需要花一些时间介绍Go的命名规范。

##### 包名 
当一个包被导入后，就可以用包名访问它的内容。在
```go
import "bytes"
```
之后，就可以引用bytes.Buffer。每个人都可以通过包名来引用其内容，这就意味着包名需要更简短、易于记忆。
通常惯例，包名是一个小写的单词；而不应该使用下划线或者是驼峰。
Err是为了要简洁，因为每个用用你的包的人都会输入这个名字，不必担心引用顺序的优先级。
包名是import时默认的名称，它不强制要求在所有源码里是唯一的，对于很少出现冲突的情况下，导入程序包可以选择一个不同的名字在本地使用。
一般情况下，冲突是很少见的，因为文件名决定了要使用哪个包。

另一个惯例是，包名是其源码路径的基础名称；在路径 src/encoding/base64 下的包应该用 "encoding/base64" 来导入，其包名是 base64， 而不是 encoding_base64 或 encodingBase64。

导入包后，可以使用包名来引用其内容，因此引用包里的名字时，可以利用这一点来避免歧义。（不要使用 import . 符号，它可以简化必须在包之外运行的测试，除此之外，应该尽量避免使用）。
例如，*bufio*包中 buffered reader type叫做Reader，而不是BufReader，因为我们是按照bufio.Reader来使用它，这样更加简洁清晰。
另外，由于导入的包总是通过它的包名来确定，因此 bufio.Reader 并不会与 io.Reader 冲突。
类似的，用于初始化ring.Ring实例的函数（相当于go中的构造函数），它通常会命名为NewRing，但因为Ring是保留唯一的类型，而且包名叫ring，因此它可以只叫New。这样看到的就是ring.New。
利用包的结构，可以帮助你选一个好的名字。

另一个小例子是*once.Do*;once.Do表达很清晰，而没有必要写成once.DoOrWaitUntilDone。
长的名字并不会增加可读性。好的注释文档会比额外的长名称更有价值。

##### Getter 
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
为了避免冲突，不要用它们来命名你的方法，除非你有相同的含义。
反过来，如果你实现了一个类型的方法，与常见的方法有相同的含义，那就用相同的名字；将你的string转换方法命名为String，而不是ToString。 

##### 驼峰记法
最后，在go里通常是用驼峰，而不是下划线。 





