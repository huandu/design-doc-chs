# 案例库

在这里写了一些理想中的样例代码。由于 API 未定，这里面的 API 调用方式不代表最终设计，会不断修改和调整。

## Macro

这里演示如何实现一个业务 assert，特点是：

- 对条件进行 assert，如果通过什么也不会发生，如果不通过，则自动向上返回错误。
- 能自动在 err 里面写上所有上下文相关变量信息。

业务代码。

```go
package main

import "url.to/my/framework/assert"

func F(a int, b float64) (r float64, err error) {
    // 当条件不满足的时候，返回错误并输出 a、b 的值。
    assert.Assert(a > 0 && b > 0)

    // 业务代码。
    r = float64(a) * b
    return
}
```

框架代码。

```go
package assert

import (
    "fmt"
    "go/token"

    "github.com/go-kuro/kuro/ast"
    "github.com/go-kuro/kuro/builder"
    "github.com/go-kuro/kuro/query"
    "github.com/go-kuro/meta/macro"
)

func Assert(cond bool) (err error) {
    // 如果 cond 满足，则什么也不发生。
    if cond {
        return
    }

    // 使用 macro.New 来执行一段 macro 代码，在这里面可以拿到编译期信息。
    // 一旦使用 macro 能力，会导致每个调用这个 Assert 的代码都被「复制」一份，
    // 工作原理就像 C++ template instantiation 一样。
    //
    // 需要注意，macro.New 中执行的函数其实就是一段普通函数，只是传入了当前 kuro 编译器上下文而已，
    // 这个函数内部并不能真正的控制任何的编译器行为，所有函数中的代码都是动态执行的代码。
    macro.New(func(ctx macro.Context) {
        // 拿到 cond 参数的 AST 信息，由于这个函数只有一个参数，所以可以直接取数组下标来拿到参数信息。
        arg := ctx.Args()[0]

        // 引用了外面定义的 err，在这里进行了赋值。
        // arg 默认的 %v 行为就是打印出源码。
        err = fmt.Errorf("assert: failed on %v", arg)
    })
    return
}

func init() {
    // macro.NewGlobal 可以注册一个全局 AST 处理器。
    // 只要使用者引用了当前 package，这个处理器就会被执行，将使用者的 package 代码作为输入
    // 这个处理器需要在 package init 或全局变量执行的时候注册，这样 kuro 就可以很容易编译这些处理器，
    // 仅需要简单的将当前 package 重新编译成 kuro AST manipulator binary 就可以了。
    // 一个 package 可以注册多个 AST 处理器，执行先后顺序按照注册顺序来排列。
    macro.NewGlobal(func(ctx macro.Context, pkg ast.Package) {
        // 通过 selector 语法选择所有 assert.Assert 的调用。
        // 这里选择的是符合指定表达式的最小 stmt。
        //
        // 得检查 stmt 中调用 assert.Assert 的地方是否处理了返回值，即看看 CallExpr 是不是一个 ExprStmt。
        // 如果处理了，则意味着使用者主动拿返回值做事情了，这里就不需要做什么特殊的事情；
        // 如果没有，则需要加上错误处理逻辑。
        query.Stmts("ExprStmt > CallExpr[Fun~=@.Assert]", pkg).Map(func(stmt ast.Stmt) (modified []ast.Stmt) {
            // 这个函数通过返回 modified 来修改 stmt 代码。

            fn := query.Parent("FuncDecl", stmt)

            // 同时还要看看执行 assert.Assert 的地方是否返回了 error，
            // 如果没返回 error，则意味着没法正常 assert，报编译错误。
            if !query.Test("Type.Results.List(-1)[Type=error]", fn) {
                ctx.Throw(stmt, "assert.Assert is called in a func/method which doesn't return error")
                return
            }

            // 拿到这个 CallExpr，将它作为后续处理的一部分。
            assert := query.Exprs("CallExpr").First()

            // 拿到函数返回值信息，尽可能的使用返回值名字进行赋值。
            err := builder.UseResult(query.Exprs("Type.Results", fn), -1)

            // 生成代码。
            //     if err = assert.Assert(a > 0 && b > 0); err != nil {
            //         return
            //     }
            modified = append(modified, &builder.IfStmt{
                Init: &builder.AssignStmt{
                    Lhs: []ast.Expr{err},
                    Rhs: []ast.Expr{assert},
                },
                Cond: &builder.BinaryExpr{
                    X: err,
                    Op: token.NEQ, // !=
                    Y: builder.Nil,
                },
                Body: &builder.BlockStmt{
                    List: []ast.Stmt{
                        builder.Return(err),
                    },
                },
            })
            return
        })
    })
}
```

## Decorator

演示如何在代码中添加 decorator。

TODO: 代码需要更新，API 升级了。

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
    "github.com/go-kuro/kuro/ast"
    "github.com/go-kuro/kuro/builder"
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

## Metadata

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

## Generic Types

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
