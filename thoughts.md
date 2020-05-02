# 一些零碎的设计想法

整体来说，`go-kuro` 项目是想做一个更好的 Go code generator，能够将一些 metaprogramming 的能力引入到 Go1，虽然可能再不到 12 个月 Go2 就可能要开始 beta 了，但是就算 Go2 的 trait 依然无法实现任何的 metaprogramming 能力，无法很好的在编译期解决很多常见的问题，比如 annotation、编译期代码特化（避免无节制使用 `interface{}`）、可编程的宏（即 metaprogramming）等。

这个仓库主要实现三大类功能：

- `kuro`：A Go command line drop-in replacement，可以直接替代 `go` 这个命令，并且增加很多额外的功能。
- kuro meta packages：一些 metaprogramming 基础库，可以方便使用者在代码里面添加一些只有 kuro 能够识别和编译的 meta data 和 marker。
- STL-like container & algorithm packages：用来演示 metaprogramming 能力的基础数据结构和算法库，类似于 C++ STL，重新把 Go 基础库里面相关的东西实现一遍，用来验证 kuro 易用性。

同时，这个项目有几个很特殊的设计：

- VCS 集成：跟 `go generate` 和很多代码生成工具不一样，`kuro` 更倾向于直接管理 VCS 里面的代码，而不是生成代码后让用户自行提交，`kuro` 会（希望能做到）自动的 merge 代码。
  - 短期这个功能只会做 git 集成。
  - `kuro` 在 `clone` 用户项目的时候，为项目创建一个本地的 bare repo，并且将用户的 remote 改成这个 bare repo，`kuro` 会修改这个 bare repo 的各种相关 hook，从而实现用户使用 `git push` 进行提交的时候，`kuro` 可以自动的转化所有 Go 代码。
  - `kuro` 需要能够「智能」的与真正的 upstream 进行 merge，这个可能会非常难。
  - 如果这一切都实现的很好，那么 `kuro` 甚至应该能够解决一些 conflict，允许用户手动编辑生成后的代码，并且自动作为特例自动更新自己的生成规则，当然，这个功能过于魔幻，暂时没想到该怎么实现。
- 可编程的构建过程：`kuro` 将 `go build` 等命令的过程进行可编程化，允许用户通过某种方式（比如写一个 `func GoBeforeBuild()` 函数）来 hook 编译过程，在编译前后做点事情，从而实现 metaprogramming。实际上，这应该是 `kuro` 本身提供的最底层的 API，所有 metaprogramming 能力应该基于这种能力构建出来。
- Go AST manipulate API：用来 query、alter、traverse AST nodes，需要发明一个好用的 API，当前业界暂时没有这个东西。在开发的时候应该要尽可能的直接使用 `go/ast` 相关 API，避免完全重新造轮子，这样才可以尽可能的向后兼容 Go2。

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
