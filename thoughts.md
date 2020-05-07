# 零碎的设计思路

这篇文档是一篇随笔，用来记录各种不成熟的设计想法，算是一个草稿箱。

## 设计目标

整体来说，`go-kuro` 项目是想做一个更好的 Go code generator，能够将一些 metaprogramming 的能力引入到 Go1。

虽然 Go2 可能不到 12 个月就要开始 beta 了，但是就算 Go2 的 trait 依然无法实现任何的 metaprogramming 能力，无法在编译期执行任何逻辑代码，比如编译期代码特化（避免无节制使用 `interface{}`）、可编程的宏（即静态代码生成）等。

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

## 功能规划

这个仓库主要实现三大类功能：

- `kuro`：A Go command line drop-in replacement，可以直接替代 `go` 这个命令，并且增加很多额外的功能。
- Kuro AST manipulation：各种 AST manipulation 库。
- Kuro macro：实现 metaprogramming 的基础「安全宏」，其本质上是一个代码预编译器。

### `kuro` 命令行

`kuro` 通过包装 `go` 命令的方法来实现各种 `go` 命令行已经支持的方法，比如 `kuro build` 其实就是直接通过命令行调用了 `go build` 并给它传递各种参数。

`kuro` 要做这一层包装的意义是在 `go` 命令执行前后插入一些 `kuro` 的处理逻辑，方便进行透明的进行代码转化、VCS 代码处理等。

`kuro` 也提供一些特殊的命令，方便做一些初始化工作。

- `kuro clone`：为 `kuro` 和 VCS 产生连接，初始化整个工作目录。
- `kuro help`：提供必要的帮助，同时也包装了 `go help` 的全部功能。
- `kuro magic`：各种扩展命令，用来实现各种特定功能。

### Kuro AST manipulation

包括以下一些库：

- `github.com/go-kuro/kuro`：生成用于分析 AST 的程序框架，由 `kuro` 命令行来调用。
- `github.com/go-kuro/kuro/ast`：重新定义、更易于修改、提供更多上下文信息的 AST。
- `github.com/go-kuro/kuro/builder`：帮助快速生成 AST。
- `github.com/go-kuro/kuro/query`：AST 的 query 机制，提供类似 CSS selector 的查询语言，以及类似 spark 的函数式编程机制。
- `github.com/go-kuro/kuro/parser`：文件解析器，将源码解析成 `kuro/ast`。
- `github.com/go-kuro/kuro/printer`：将 `kuro/ast` 还原成 Go1 源码，并提供反编译信息和 patch 信息。

### Kuro macro

`go-kuro` 要实现 metaprogramming，就需要有一些接口来操作预编译期间的 AST。

`go-kuro` 的一个大原则是：在不修改 Go 语法的前提下进行 metaprogramming。

这样的原则可以方便 `kuro` 自然的与各种 IDE 进行整合，避免像 TypeScript 一样来从零构建整个生态，这种事情只有 Microsoft 这种体量和社区影响力的公司能做到。同时，这个原则的可行性也很高，具体实现思路可以参考 Rust/Nim 的 macro 系统，一些额外的编译器执行的指令来对代码进行操作，将源码当做输入来使用。

## 设计思路

### 基本工作原理

以 `kuro build` 过程为例。

1. 分析当前待编译的项目代码和它依赖的所有代码，将这些代码放入 `kuro` 缓存目录中，这部分可以直接基于 `go mod graph` 的功能来实现。
2. 解析所有代码的 AST，构建一个依赖树。
3. 从依赖树的最叶子节点出发，逐个分析是否存在使用 `github.com/go-kuro/macro` 的代码，如果有，开始生成「生成代码」的程序，并执行。
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
# Install kuro.
$ go install github.com/go-kuro/cmd/kuro

# Ask kuro to trace a project.
$ kuro clone https://url.to.my/go-repo/name.git
$ cd name

# Edit Go files and test it by using kuro.
$ kuro build  # A replacement of `go build`.
$ kuro test   # A replacement of `go test`.

# It's time to submit code changes and push it to upstream.
$ git add .
$ git commit -m 'awesome commit'  # Commit changes as usual.
$ git push                        # Push to upstream which is controlled by kuro.
# Then, kuro will generate code and submit changes to real repo.
```

## 废弃的想法

一些想过但是后来觉得不合理的东西。

### Type guard

之前曾想过要实现类似 TypeScript 的 type guard 和各种运算符，但是由于这个机制并不是很容易在 Go 语法基础上优雅的实现，包括 Go2 trait 支持后也不容易写出像 TypeScript 一样的 alias 和 guard 语法。

特别是仔细评估了 TypeScript type guard 和 Rust/Nim 泛型能力之后，Go2 trait + interface 已经能够解决大部分泛型问题，再实现一个语法并不优雅的 type guard 收益不大。而且 type guard 主要解决的是不得不使用 `interface{}` 时候，减少调用者的代码错误，编译器多做一些事情来做输入检查，这是 TypeScript 之于 Javascript 要解决的主要问题。

现有的大原则是「编译层面上完全兼容 Go1」从而避免自己开发各种开发工具，在没有很好的设计和收益情况下，倾向于不做。

### STL-like packages

由于 Go2 的 trait 一旦启用，这些 STL-like 库已经可以得到比较好的开发和使用体验，所以看起来已经不太需要重新用 macro 或者其他机制实现了。
