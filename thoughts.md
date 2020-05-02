# 零碎的设计思路

这篇文档是一篇随笔，用来记录各种不成熟的设计想法，算是一个草稿箱。

## 设计目标

整体来说，`go-kuro` 项目是想做一个更好的 Go code generator，能够将一些 metaprogramming 的能力引入到 Go1。

虽然 Go2 可能不到 12 个月就要开始 beta 了，但是就算 Go2 的 trait 依然无法实现任何的 metaprogramming 能力，无法在编译期执行任何逻辑代码，比如 decorator、编译期代码特化（避免无节制使用 `interface{}`）、可编程的宏（即静态代码生成）等。

## 使用场景

### 减少重复代码

在 Go 里面经常会有很多很重复的代码。

- 判断错误：比如随处可见的 `if err != nil {...}`。
- 各种静态和动态 assertion：实现一些编译期 assert，并且使用 `return` 来实现动态 assert，避免使用 panic，并允许框架设计者在返回前插入逻辑和日志。
- 常见的优化与最佳实践：特别是必须用泛型才能实现的优化，方便框架开发者降低使用者的思考负担，提升最终业务代码的质量，比如封装 `chan` 的最佳实践、自动使用 `strings.Builder` 来拼接字符串、减少使用不必要的 `interface{}` 等。
- 常见的 `go generate` 场景：有一些经常会需要但很费事的场景能力，比如实现枚举变量并给每个变量来个名字。

这些能力都是为 Go 框架开发者实现的，希望在 `kuro` 开发阶段也能用于 `kuro` 代码本身——自己编译自己，这才是 metaprogramming 的本意。

### 实现 macro 能力

提供一套类似于 Rust 的 macro API，从而实现 metaprogramming 能力。

参考 [Rust Macros](https://doc.rust-lang.org/1.43.0/book/ch19-06-macros.html) 和 [Nim Macros](https://nim-lang.org/docs/manual.html#macros)。

### 实现 trait

考虑到 Go2 很可能会加上 Rust 类似的 trait 语法，在 `go-kuro` 里面不做任何 Go2 trait 相关的能力，等 Go2 发布。

不过需要实现一些 Go2 不打算实现的额外 trait，比如 `string | int` 这种同时明确的支持多种不兼容类型的代码。

参考 [Rust traits](https://doc.rust-lang.org/1.43.0/book/ch10-02-traits.html) 和 [TypeScript Advanced Types](https://www.typescriptlang.org/docs/handbook/advanced-types.html)。

## 功能规划

这个仓库主要实现三大类功能：

- `kuro`：A Go command line drop-in replacement，可以直接替代 `go` 这个命令，并且增加很多额外的功能。
- kuro meta packages：一些 metaprogramming 基础库，可以方便使用者在代码里面添加一些只有 kuro 能够识别和编译的 meta data 和 marker。
- STL-like container & algorithm packages：用来演示 metaprogramming 能力的基础数据结构和算法库，类似于 C++ STL，重新把 Go 基础库里面相关的东西实现一遍，用来验证 kuro 易用性。

### `kuro` 命令行

`kuro` 通过包装 `go` 命令的方法来实现各种 `go` 命令行已经支持的方法，比如 `kuro build` 其实就是直接通过命令行调用了 `go build` 并给它传递各种参数。

`kuro` 要做这一层包装的意义是在 `go` 命令执行前后插入一些 `kuro` 的处理逻辑，方便进行透明的进行代码转化、VCS 代码处理等。

`kuro` 也提供一些特殊的命令，方便做一些初始化工作。

- `kuro clone`：为 `kuro` 和 VCS 产生连接，初始化整个工作目录。
- `kuro help`：提供必要的帮助，同时也包装了 `go help` 的全部功能。
- `kuro magic`：各种扩展命令，用来实现各种特定功能。

### Kuro meta packages

`go-kuro` 要实现 metaprogramming，就必须提供一系列额外接口来提供 meta 数据。

`go-kuro` 的一个大原则是：在不修改 Go 语法的前提下进行 metaprogramming。

这样的原则可以方便 `kuro` 自然的与各种 IDE 进行整合，避免像 TypeScript 一样来从零构建整个生态，这种事情只有 Microsoft 这种体量和社区影响力的公司能做到。同时，这个原则的可行性也很高，具体实现思路可以参考 Rust/Nim 的 macro 系统，以及 TypeScript 的 metadata 库，一些额外的编译器执行的指令来对代码进行操作，将源码当做输入来使用。

### STL-like packages

考虑到 Go 标准库里面已经实现了不少的基础类型，代码实现难度并不大，在符合开源协议的前提下借鉴就好了，重点是用来演示 `kuro` 在 metaprogramming 的可能性。

应该会包括一下的库：

- `container/*`：标准库的各种容器。
- `sort`：标准库的排序。

## 设计思路

### 基本工作原理

以 `kuro build` 过程为例。

1. 分析当前待编译的项目代码和它依赖的所有代码，将这些代码放入 `kuro` 缓存目录中，这部分可以直接基于 `go mod graph` 的功能来实现。
2. 解析所有代码的 AST，构建一个依赖树。
3. 从依赖树的最叶子节点出发，逐个分析是否存在使用 `github.com/go-kuro/meta` 的代码，如果有，开始生成「生成代码」的程序，并执行。
   - `kuro` 将所有用到 meta 的代码转化成操作 AST 的代码，提取出来放到一个临时的目录里面，所有函数的输入都是 AST，输出是修改过后的 AST，编译相关代码，生成一个独立的 binary。
   - `kuro` 分析所有被修改的代码位置，在 AST 里面标记出来，生成 DAG。
   - `kuro` 将 AST 和 DAG 输入给生成的 binary，执行完成后得到生成好的 AST。
   - `kuro` 将 AST 转化成实际的代码，在此过程中，如果在当前仓库控制之外的代码被修改（比如框架的代码），需要将代码放入 `internal/kuro/traced` 目录里去，并改相关的 import。由于 Go 不能简单的对库进行部分修改，所以一旦一个库的某个目录有一行代码被 `kuro` 修改，则这个目录所有文件都需要放到这个目录里面进行管理；如果这个目录有代码调用了库的 `internal` 目录里面的东西，则整个库都需要放到这个 `internal` 里面。
4. 调用 `go build` 进行真正的编译工作。

### 特殊设计思路

同时，这个项目有几个很特殊的设计：

- VCS 集成：跟 `go generate` 和很多代码生成工具不一样，`kuro` 更倾向于直接管理 VCS 里面的代码，而不是生成代码后让用户自行提交，`kuro` 会（希望能做到）自动的 merge 代码。
  - 短期这个功能只会做 git 集成。
  - `kuro` 在 `clone` 用户项目的时候，为项目创建一个本地的 bare repo，并且将用户的 remote 改成这个 bare repo，`kuro` 会修改这个 bare repo 的各种相关 hook，从而实现用户使用 `git push` 进行提交的时候，`kuro` 可以自动的转化所有 Go 代码。
  - `kuro` 需要能够「智能」的与真正的 upstream 进行 merge，这个可能会非常难。
  - 如果这一切都实现的很好，那么 `kuro` 甚至应该能够解决一些 conflict，允许用户手动编辑生成后的代码，并且自动作为特例自动更新自己的生成规则，当然，这个功能过于魔幻，暂时没想到该怎么实现。
- 可编程的构建过程：`kuro` 将 `go build` 等命令的过程进行可编程化，允许用户通过某种方式（比如写一个 `func GoBeforeBuild()` 函数）来 hook 编译过程，在编译前后做点事情，从而实现 metaprogramming。实际上，这应该是 `kuro` 本身提供的最底层的 API，所有 metaprogramming 能力应该基于这种能力构建出来。
- Go AST manipulate API：用来 query、alter、traverse AST nodes，需要发明一个好用的 API，当前业界暂时没有这个东西。在开发的时候应该要尽可能的直接使用 `go/ast` 相关 API，避免完全重新造轮子，这样才可以尽可能的向后兼容 Go2。

## 使用方法

### 初始化项目

预期中基本的 `kuro` 使用方法。

```shell
$ kuro clone https://url.to.my/go-repo/name.git
$ cd name

$ # Edit Go files and test it by using kuro.
$ kuro build  # instead of go build.
$ kuro test   # instead of go test.

$ # It's time to submit code changes and push it to upstream.
$ git add .
$ git commit -m 'awesome commit'  # commit changes as usual.
$ git push                        # push to upstream controlled by kuro.
                                  # kuro will generate code and merge changes.
```

### 使用 macro

演示如何在 Go 里面使用 macro。

```go
// 业务代码。

package main

import (
    "context"

    "url.to/my/framework/foo"
)

func Hello(ctx context.Context, name string) (msg string, err error) {
    foo.Validate(ctx, name != "" && len(name) < 25) // 如果不符合要求，则返回框架预设值的错误码并返回。

    // 业务代码。
}
```

```go
// 框架代码。

package foo

import (
    "github.com/go-kuro/meta/ast"
    "github.com/go-kuro/meta/builder"
    "github.com/go-kuro/meta/macro"
)

// Validate 检查 conds，如果不为真，则终止后续逻辑。
func Validate(ctx context.Context, cond bool) {
    // macro.Caller 获得指定调用者的上下文，
    // 然后在这个上下文里面运行指定的宏。
    macro.Caller(Validate).Run(validate)
}

// validate 返回 stmt 会经过 kuro 编译成一段代码 Go 代码。
func validate(decl ast.Decl, expr ast.CallExpr) (stmt ast.Stmt) {
    // API 还没想好怎么设计，先写一个 demo。

    ctx := expr.Args(0)       // 第一个参数。
    cond := expr.ArgsRange(1) // 第二个参数。

    // 获得当前调用这段代码所在函数声明。
    results := decl.(ast.FuncDecl).Results()

    // 相当于在业务代码里面写了几个 import。
    pkg := builder.Import(macro.CurrentPackage())
    pkgLog := builder.Import("path/to/a/log/pkg")

    // 相当于写了如下代码：
    //     func Hello(ctx context.Context, name string) (msg string, err error) {
    //         if !(name != "" && len(name) < 25) {
    //             log.Errorf(ctx, "invalid query")
    //             err = foo.ServerError(ctx, foo.ErrInvalidQuery)
    //             return
    //         }
    //
    //         // 业务代码
    //     }
    stmt = builder.If(nil, builder.LogicNot(cond), builder.Block(
        builder.Call(pkgLog.Lookup("Errorf"), ctx, "invalid query"),
        results.SetError(
            builder.Call(pkg.Lookup("ServerError"), ctx, pkg.Lookup("ErrInvalidQuery")),
        )
        builder.Return(results),
    ))
    return
}
```

### 使用 decorator

演示如何在代码中添加 decorator。

```go
// 业务代码。

package main

import (
    "github.com/go-kuro/meta/deco"

    "url.to/my/framework/foo"
)

var _ = deco.Decorate(Hello).With(foo.NoPanic)

func Hello(ctx context.Context, name string) (msg string, err error) {
    panic("should be caught by foo")
}
```

```go
// 框架代码。

package foo

import (
    "github.com/go-kuro/meta/ast"
    "github.com/go-kuro/meta/builder"
)

// NoPanic 在函数开头增加一段 defer 代码。
func NoPanic(decl ast.Decl) ast.Decl {
    // API 还没想好怎么设计，先写一个 demo。

    r := builder.Var("r")
    results := decl.(ast.FuncDecl).Results()

    // 相当于在业务代码里面写了几个 import。
    pkg := builder.Import(macro.CurrentPackage())

    // 生成一段代码：
    //     defer func() {
    //         if r := recover(); r != nil {
    //             err = foo.ServerError(ctx, foo.ErrInternalServerError)
    //         }
    //     }()
    stmt := builder.Defer(builder.Call(builder.FuncLit(nil, nil, builder.Block(
        builder.If(builder.Assign(r, builder.Recover()), builder.Neg(r, nil), builder.Block(
            results.SetError(
                builder.CallExpr(pkg.Lookup("ServerError"), ctx, pkg.Lookup("ErrInternalServerError")),
            )
        ))
    ))))
    list := append([]ast.Stmt{stmt}, decl.Body.List...)
    decl.Body.List = list
    return decl
}
```

### 使用 metadata

演示如何在代码中添加 metadata。

```go
// 业务代码。

package main

import (
    "github.com/go-kuro/meta/deco"

    "url.to/my/framework/foo"
)

var _ = deco.Decorate(Hello).SetData("foo:url", "/api/foo")

func Hello(ctx context.Context, name string) (msg string, err error) {
    // 业务代码。
}
```

这样设置 metadata 之后，可以在 `ast.Node` 里面读到这个数据。

```go
type Node interface {
    Data(key string) (value interface{}, ok bool)
}
```

### 使用静态类型

```go
package fmt

import (
    "github.com/go-kuro/meta/types"
)

func Printf(format string, values ...types.Any) (n int, err error) {
    // types.Any 代表任意不同的类型，底层用 interface{} 实现，编译时候会转成相关必要代码。
    // 具体实现参考 C++20 std::format 的实现思路，由于过于复杂，不在此展开。
    // 还有很多思路和实现的细节没有想清楚，未来需要完善。
}
```
