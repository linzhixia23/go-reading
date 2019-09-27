原文：https://golang.org/doc/effective_go.html

<br>

#### 介绍
Go是一门全新的语言。虽然它从现有的语言借鉴了一些理念，但它的语言特性，使得编写高效的go程序与其他语言完全不同。直接把现有的C++或Java程序翻译成go，结果并不理想--毕竟Java程序不是go。另一方面，如果从go的视角去思考，则能够写出更好的、不一样的代码。换句话说，要想把go程序写好，需要深入理解它的特性和风格。同时，还要了解go的既定规则，包括命名规范、格式、代码结构等，这样你的代码才更有可读性。

这篇文档介绍一些tips，帮助编写清晰地道的go代码。它是对之前文档的补充说明，因此建议先阅读这些文档。

##### 示例
go源码包不仅提供了核心库，同时还有一些例子示范如何使用go语言。此外，很多库包含了可执行的示例。如果你有关于一些问题如何解决，或者如何实现的疑问，代码库的文档、代码和示例都能够提供解答、思想和背景。


```go
import "golang.org/x/oauth2"

func main() {
	ctx := context.Background()
	ts := oauth2.StaticTokenSource(
		&oauth2.Token{AccessToken: "... your access token ..."},
	)
	tc := oauth2.NewClient(ctx, ts)

	client := github.NewClient(tc)

	// list all repositories for the authenticated user
	repos, _, err := client.Repositories.List(ctx, "", nil)
}
```





