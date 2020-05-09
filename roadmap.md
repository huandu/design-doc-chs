# 路线图

整体开发应该采用逐步迭代的方式进行，每一步都会产生一些相对有用的库和工具，最后才开发出这个项目的完整形态。一旦完整形态开发完成，则进入持续迭代阶段。

## 阶段 0：实现 AST manipulator

主要工作包括：

- 设计整体架构，验证各种细节功能的基本可行性。
- 调研各种 meta language 和 metaprogramming 设计，特别是与 Go 类似的静态编译和代码转化型语言 Rust/Nim/TypeScript，以及 C# Expression List 和 OCaml。
- 实现一个基于 `go/ast` 和相关准官方库的 Go AST query/alter/patch 的库。
- 实现各种 scope/declaration/variable tracking 等基本技术。

## 阶段 1：实现代码生成器与 macro

主要工作包括：

- 实现 `github.com/go-kuro/macro`：各种 macro 函数真正使用的 API，里面应该包含各种 AST 相关的上下文信息，由 `kuro` 负责填充其中的真实内容。
- 实现 `github.com/go-kuro/kuro` 基本能力，这里所有的库都与 `kuro` 的核心功能紧密相连，包括 `kuro` 自己应该也在使用这些能力来完成各种功能，原则上，用这些公开的 API 就可以重写实现 `kuro` 命令。
  - `github.com/go-kuro/kuro/parser`：各种 Go 源码解析器，能够将一个 package 解析成 kuro AST。
  - `github.com/go-kuro/kuro/ast`：各种 AST 元素定义，与 `go/ast` 在名字上保持一致，但是可以被编辑。
  - `github.com/go-kuro/kuro/printer`：将 AST 重新输出成 Go1 可以编译的代码，短期来说可以直接复制 `go/printer` 代码，但长期必须自己重写，从而能够更好的定制，甚至实现基于 AST 的反向代码生成。

考虑到阶段 1 要做的事情实在太多了，这里仅需要完成所有库的接口设计，以及支持实现一个最基本的 macro 用来实现基于 return 的 assert 逻辑。

## 阶段 2：实现 `kuro` 命令行基本功能

实现 `kuro` 命令中的基本功能，主要工作包括：

- 实现 `github.com/go-kuro/cmd/kuro`：
  - 劫持 `go build`/`go run`/`go test`/`go get`/`go install` 等常见命令。
  - 实现项目 cache，每次在执行 `kuro build` 等需要编译的命令时候，将本地代码生成到一个 cache 目录里面去，此时还不用做性能优化，每次都重新生成就好了。
- 实现 `github.com/go-kuro/kuro`
  - 实际上 `kuro` 命令行仅是一个启动器，所有真正跟分析代码和 AST 相关的工作都在这个库中完成，特别是这个库的根目录，里面是真正用来编译 macro 程序的框架逻辑。
  - `kuro` 会根据用户代码所引用的 `github.com/go-kuro/kuro` 版本来构建 macro 程序。

## 阶段 3：实现 Git trampoline

主要实现 `kuro` 对 Git 仓库的管理。

- 实现 `github.com/go-kuro/vcs` 中跟 Git 相关功能。
- 实现更高效的 cache 目录，确保代码可以在每次提交之间增量的进行更新。
- 实现基本的 Git auto merge 能力。

这个阶段结束之后，`kuro` 就可以算是 alpha 了。

## 阶段 4：完成第一个可用版本

清理各种代码，完善 API、文档、测试用例等。

## 规划中但暂无计划的功能

包含一些重要但不是那么必要的功能。

- `github.com/go-kuro/kuro` 的扩展功能：
  - `github.com/go-kuro/kuro/builder`：各种用于构建 AST 的工具函数和类型。
  - `github.com/go-kuro/kuro/query`：各种 AST 查询语言，类似 CSS selector，支持通过 map/reduce 方式修改，方便未来做更好的并发控制。
- 调试工具：
  - 基于 `delve` 或者 `gdb` 的调试器扩展功能，让调试器可以将断点设置在 `kuro` 处理前的源码上。
  - 调试 metaprogramming 相关代码。
