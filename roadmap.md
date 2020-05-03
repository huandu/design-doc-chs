# 路线图

整体开发应该采用逐步迭代的方式进行，每一步都会产生一些相对有用的库和工具，最后才开发出这个项目的完整形态。一旦完整形态开发完成，则进入持续迭代阶段。

## 阶段 0：实现 AST manipulator

主要工作包括：

- 设计整体架构，验证各种细节功能的基本可行性。
- 实现一个基于 `go/ast` 和相关准官方库的 Go AST query/alter/patch 的库。
- 实现各种 scope/declaration/variable tracking 等基本技术。

## 阶段 1：实现代码生成器与 macro

主要工作包括：

- 实现 `github.com/go-kuro/meta` 基本能力，这里面所有的库都会需要 `kuro` 进行代码生成或者重写，一旦引用了（包括间接易用），就会激活 `kuro` 的功能。
  - `github.com/go-kuro/meta/types`：编译器类型 beacon/marker，用于做各种 intersection/union/type guard/etc，用于「激活」 `kuro` 的类型系统，都是一些纯粹编译期使用的类型。参考 [TypeScript Advanced Types](https://www.typescriptlang.org/docs/handbook/advanced-types.html)。
  - `github.com/go-kuro/meta/macro`：各种 macro 函数真正使用的 API，里面应该包含各种 AST 相关的上下文信息，由 `kuro` 负责填充其中的真实内容。
- 实现 `github.com/go-kuro/kuro` 基本能力，这里所有的库都与 `kuro` 的核心功能紧密相连，包括 `kuro` 自己应该也在使用这些能力来完成各种功能，原则上，用这些公开的 API 就可以重写实现 `kuro` 命令。
  - `github.com/go-kuro/kuro/builder`：各种用于构建 AST 的工具函数和类型。
  - `github.com/go-kuro/kuro/parser`：各种 Go 源码解析器，能够将一个 package 解析成 kuro AST。
  - `github.com/go-kuro/kuro/ast`：各种 AST 元素定义，与 `go/ast` 在名字上保持一致，但是可以被编辑，且全部都是 interface。
  - `github.com/go-kuro/kuro/printer`：将 AST 重新输出成 Go1 可以编译的代码。
  - `github.com/go-kuro/kuro/query`：各种 AST 查询语言，类似 CSS selector，支持通过 map/reduce 方式修改，方便未来做更好的并发控制。

考虑到阶段 1 要做的事情实在太多了，这里仅需要完成所有库的接口设计，以及支持实现一个最基本的 macro 用来实现基于 return 的 assert 逻辑。

## 阶段 2：实现项目 cache

TBD

## 阶段 3：实现 Git trampoline

TBD

## 阶段 4：实现 `kuro` 命令行

TBD

## 阶段 5：完成整个 `go-kuro` 第一个可用版本

TBD
