# `golang.org/x/tools/go/ast/astutil` 简介

这个库可以修改 AST 结构本质上就是直接修改 `go/ast` 的公开数据结构来完成各种 AST 修改。

下面简单介绍一下关键的接口用法和实现原理，代码基于 `v0.0.0 (d5fe738)`。

## `AddImport`/`AddNamedImport`

`AddImport` 直接调用了 `AddNamedImport`，因此我们只需要看看 `AddNameImport` 如何实现即可。

```go
func AddNamedImport(fset *token.FileSet, f *ast.File, name, path string) (added bool) {
    // 如果已经 import 过这个 path，忽略。
    if imports(f, name, path) {
        return false
    }

    // 直接构造新的 ast.ImportSpec。
    newImport := &ast.ImportSpec{
        Path: &ast.BasicLit{
            Kind:  token.STRING,
            Value: strconv.Quote(path),
        },
    }
    if name != "" {
        newImport.Name = &ast.Ident{Name: name}
    }

    // 将 newImport 插入到正确的代码位置，从而符合 Go 编程规范。
    var (
        bestMatch  = -1         // length of longest shared prefix
        lastImport = -1         // index in f.Decls of the file's final import decl
        impDecl    *ast.GenDecl // import decl containing the best match
        impIndex   = -1         // spec index in impDecl containing the best match

        isThirdPartyPath = isThirdParty(path)
    )
    for i, decl := range f.Decls {
        // 代码略……
    }

    // 如果当前文件一个 import 都没有，将这个 newImport 作为第二行语句加在 package 关键词之后。
    if impDecl == nil {
        // 代码略……
    }

    // 修正 newImport 的 Pos 信息，让 `ast/printer` 等相关库能够正确的输出这个 import 的源码。
    // 需要注意，在这里并没有因为插入了这个 newImport 而整个修改 fset 和原来 AST 的 Pos，没必要改。

    // 代码略……

    // 处理 ast.ImportDecl 的括号，如果只有一条 import 则去掉括号，多于一条则加上括号。

    // 代码略……

    // 将 newImport 加入 f.Imports。
    f.Imports = append(f.Imports, newImport)

    if len(f.Decls) <= 1 {
        return true
    }

    // 尽量将所有散落的 ImportDecl 合并成同一个 ImportDecl。
    var first *ast.GenDecl
    for i := 0; i < len(f.Decls); i++ {
        // 代码略……
    }

    return true
}
```

## `PathEnclosingInterval`

`PathEnclosingInterval` 可以查看在指定范围 `[start, end)` 内完整的 `ast.Node`。

例如，假设制定的范围是 `A`，那么这个函数会返回 `x + y` 这个 `ast.BinaryExpr` 表达式。

```go
z := x + y // add them
     <-A->
```

几个需要注意的地方：

- 如果指定的范围在一个 Stmt 中横跨了几个兄弟 Node，则会返回他们共同的父结点。
- 如果范围仅仅指向一片空白或者注释，那么会返回这段注释绑定的 Stmt。
- `path []ast.Node` 里面包含的结点，是从 `[start, end)` 完全包含的最小 Node 开始，到这个 Node 所有父结点，以 `root` 为止。

```go
func PathEnclosingInterval(root *ast.File, start, end token.Pos) (path []ast.Node, exact bool) {
    // Precondition: node.[Pos..End) and adjoining whitespace contain [start, end).
    var visit func(node ast.Node) bool
    visit = func(node ast.Node) bool {
        path = append(path, node)

        nodePos := node.Pos()
        nodeEnd := node.End()

        // 缩小 [start, end) 以适应 node 的范围，方便后面处理。
        if start < nodePos {
            start = nodePos
        }
        if end > nodeEnd {
            end = nodeEnd
        }

        // 找到 node 所有的 children。
        // childrenOf 函数对所有 AST node 都进行了定制化的判断。
        children := childrenOf(node)
        l := len(children)
        for i, child := range children {
            // child 的源码范围。
            childPos := child.Pos()
            childEnd := child.End()

            // 包含了前后空白和注释的 child 范围。
            augPos := childPos
            augEnd := childEnd
            if i > 0 {
                augPos = children[i-1].End() // start of preceding whitespace
            }
            if i < l-1 {
                nextChildPos := children[i+1].Pos()
                // Does [start, end) lie between child and next child?
                if start >= augEnd && end <= nextChildPos {
                    return false // inexact match
                }
                augEnd = nextChildPos // end of following whitespace
            }

            // 如果 [start, end) 在这个 child 范围内，继续深度遍历。
            if augPos <= start && end <= augEnd {
                // 在 childrenOf 这个函数里面，所有的 token（比如各种运算符）都会被包装成一个 ast.Node，方便前面做一些统一的计算。
                // 很显然，如果 [start, end) 恰好落入到一个 token 范围内，这个 token 没必要进一步 visit。
                _, isToken := child.(tokenNode)
                return isToken || visit(child)
            }

            // 如果 [start, end) 横跨几个 child，那么就只有将整个 node 放到 path 里面去了。
            if start < childEnd && end > augEnd {
                break
            }
        }

        // 如果所有的 children 都不包含 [start, end)，那么当前结点就是最终结果了，停止遍历。

        // 这个判断不能放在最前面，因为存在一些 Node，父结点和 children 的范围完全一样。
        if start == nodePos && end == nodeEnd {
            return true // exact match
        }

        return false // inexact: overlaps multiple children
    }

    if start > end {
        start, end = end, start
    }

    if start < root.End() && end > root.Pos() {
        if start == end {
            end = start + 1 // empty interval => interval of size 1
        }

        // 开始深度遍历。
        exact = visit(root)

        // 做个逆序排序，使得 path[0] 是最叶子的那个结点。
        for i, l := 0, len(path); i < l/2; i++ {
            path[i], path[l-1-i] = path[l-1-i], path[i]
        }
    } else {
        // 如果 [start, end) 比 root 还大，直接在 path 里面放入 root。
        path = append(path, root)
    }

    return
}
```

## `Apply`

`Apply` 深度遍历 node，在遍历一个 node 前调用 pre，遍历结束后调用 post。在 pre 和 post 中可以修改 node，从而直接改变后续遍历的次序和结果。

这个函数和周边工具代码的行数较多，在这里不贴源码了。整体逻辑并不复杂，非常类似于 `ast.Walk` 逻辑，只是更加明确的提供了 pre 和 post 回调来方便使用者处理 node。
