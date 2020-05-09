# 案例库

在这里写了一些理想中的样例代码。由于 API 未定，这里面的 API 调用方式不代表最终设计，会不断修改和调整。

## Macro 基础

这里演示如何用 macro 实现一个业务 assert，特点是：

- 对条件进行 assert，如果通过什么也不会发生，如果不通过，则自动向上返回错误。
- 能自动在 err 里面写上所有上下文相关变量信息。

### 使用 `assert` 的业务代码

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

### `assert` 框架代码

```go
package assert

import (
    "fmt"

    "github.com/go-kuro/kuro/ast"
    "github.com/go-kuro/macro"
)

// Assert 实现类似 C 的 assert，能够将 cond 真正的源码放到 err 里面去。
func Assert(cond bool) (err error) {
    // 如果 cond 满足，则什么也不发生。
    if cond {
        return
    }

    // 使用 macro.New 来执行一段 macro 代码，在这里面可以拿到编译期信息。
    // 一旦使用 macro 能力，会导致每个调用这个 Assert 的代码都被「复制」一份，
    // 工作原理就像 C++ template instantiation 一样。
    //
    // 生成了 ast.Stmt 之后，kuro 将这些代码 inline 到当前函数中去。
    macro.New(func(ctx macro.Context) ast.Stmt {
        // 拿到 cond 参数的 AST 信息，由于这个函数只有一个参数，所以可以直接取数组下标来拿到参数信息。
        args := ctx.Args()
        arg := macro.Var(ctx, args[0]) // 定义一个可以在 macro.Sttm 中使用的「变量」，编译期计算。

        // macro.Parse 不会真的执行，这只是一个编译期的工具函数，用来把输入的任何表达式变成 `ast.Expr`。
        //
        // 下面这段代码会被 kuro 解析成 AST，并且将其中可以在编译器得到结果的部分替换成常量。
        // 比如这里面的 arg，会在编译期得到具体值 "a > 0 && b > 0"。
        //
        // macro.Parse 会自动解析 func() 的代码并且生成必要的 AST 构建代码，
        // 相当于手写了下面的代码：
        //
        //     fmtPkg := builder.Import("fmt")
        //     return &ast.AssignStmt{
        //         Lhs: []ast.Expr{ast.NewIdent("err")}, // 这个 err 是引用闭包外的普通变量。
        //         Rhs: []ast.Expr{
        //             &ast.CallExpr{
        //                 Fun: fmtPkg.Func("Errorf"),
        //                 Args: []ast.Expr{
        //                     &ast.BasicLit{
        //                         Kind: token.STRING,
        //                         Value: `"assert: failed on %v"`,
        //                     },
        //                     arg, // 注意这里，使用的是闭包外面的变量 arg，编译期会得到具体值。
        //                 },
        //             },
        //         },
        //     }
        return macro.Parse(func() {
            err = fmt.Errorf("assert: failed on %v", arg)
        }).(*ast.FuncLit).Body
    })
    return
}
```

## 用 Macro 修改函数参数

用 macro 的能力修改函数的入口参数，从而实现一些比较有意思的 DSL。这里演示的是 SQL builder 中的查询条件。

### 使用 SQL DSL 的业务代码

```go
package main

import (
    "time"

    "url.to/my/framework/orm"
)

type User struct {
    Name      string
    Email     string
    CreatedAt time.Time
}

func FindUserByID(ctx context.Context, uid UserID) (err error) {
    var user User

    err = orm.DB(ctx).From("users").Where(
        `uid` == orm.Var(uid) && `status` == orm.In(1, 2, 3)
    ).First(&user)

    // 使用这些变量进行业务处理。

    return
}
```

### SQL DSL 框架实现

这里仅演示 `Where` 的实现，其他方法不太需要用到 macro。

```go
package orm

import (
    "go/token"
    "strconv"
    "strings"

    "github.com/go-kuro/kuro/ast"
    "github.com/go-kuro/macro"
)

func (q *Query) Where(cond bool) *Query {
    // 虽然这个函数参数只有 cond，但可以通过 macro 能力直接改写函数参数，从而接收更多的输入。
    //
    // 一些注意事项：
    //   - 函数的参数和返回值都可以被修改，但是 Recv 不能被修改。
    //   - 如果函数返回值类型修改后造成调用者无法通过编译，kuro 会报错。
    //   - 如果这个函数是 Query 实现某种 interface 的一个必要函数，那么 kuro 会直接报错，
    //     这是因为 kuro 必然会修改函数的名字来实现 generic，必然会导致 interface 不再被满足。

    // macro.Rewrite 调用之后，整个函数签名和实现都会修改，前面任何的代码都会被忽略。
    // 使用者可以用这个特性来实现 static assert 类似的能力，仅作一些编译检查。
    macro.Rewrite(func(ctx macro.Context) (fn *ast.FuncLit, args []ast.Expr) {
        condExpr := ctx.Args()[0] // 原函数只有一个参数，可以放心的取下标。
        whereExpr, args := parseWhereExpr(condExpr)

        // 声明用于 macro.Parse 的变量，这些变量的类型通过 `.(type)` 来指定。
        // 如果以后 Go2 普及了，可以使用 Go2 的 generic 语法，即类似于 `macro.Var(*Query)(ctx.Recv())`。
        q := macro.Var(ctx, ctx.Recv()).(*Query)
        where := macro.Const(ctx, whereExpr).(string) // whereExpr 必须是一个可以赋值给 const 的表达式，否则 kuro 会报错。

        fn = macro.Parse(func(values ...interface{}) *Query {
            q.where = append(q.where, &whereExpr{
                Where: where,
                Values: values,
            })
            return q
        })
        return
    })
}

// parseWhereExpr 将参数解析成一段 SQL 字符串常量。
//
// 要实现这个 Where 的关键思路是：解析 condExpr 的逻辑表达式，并且将它转成 SQL 逻辑表达式。
// 由于完整实现这个能力会需要花费非常多的代码，这里仅作一些必要的 demo，实现下面简单形式的代码。
//
//     `uid` == orm.Var(uid) && `status` == orm.In(1, 2, 3)
//
// 因此，暂时忽略 UnaryExpr 以及括号之类的。
//
// 最终生成的 SQL 语句如下：
//
//     result = "`uid` = ? AND `status` IN (?, ?, ?)"
//     values = uid, 1, 2, 3 // 这些是代码，懒得写 AST 了。
func parseWhereExpr(expr ast.Expr) (result ast.Expr, values []ast.Expr) {
    binary := expr.(*ast.BinaryExpr)

    if isLogic(binary) {
        return parseLogic(expr)
    }

    var op string

    switch binary.Op {
    case token.LAND: // &&
        op = " AND "
    case token.LOR: // ||
        op = " OR "
    default:
        panic("orm: unsupported logic op")
    }

    x, xValues := parseWhereExpr(binary.X)
    y, yValues := parseWhereExpr(binary.Y)
    values = xValues
    values = append(values, yValues...)
    return &ast.BinaryExpr{
        X: &ast.BinaryExpr{
            X: x,
            Op: token.ADD,
            Y: &ast.BasicLit{
                Kind: token.STRING,
                Value: strconv.Quote(op),
            },
        },
        Op: token.ADD,
        Y: y,
    },
}

// isLogic 判断这个 binary 是否是一个查询逻辑了。
// 如果 X 或者 Y 是 BasicLit，则说明是形如 `col1` == value1 的形式，
// 可以直接开始分析结构并生成 SQL 了。
func isLogic(expr *ast.BinaryExpr) bool {
    _, ok1 := expr.X.(*ast.BasicLit)
    _, ok2 := expr.Y.(*ast.BasicLit)
    return ok1 || ok2
}

// parseLogic 解析逻辑表达式，并且返回 SQL 表达式。
// 为了避免示例写太长，这里假定 X 一定是 `col` 形式的常量，Y 一定是 `orm.Var(xxx)` 的函数调用。
func parseLogic(logic *ast.BinaryExpr) (expr ast.Expr, values []ast.Expr) {
    var op, value string

    switch binary.Op {
    case token.EQL: // &&
        op = " = "
    case token.NEQ: // ||
        op = " <> "
    default:
        panic("orm: unsupported logic op")
    }

    col := logic.X.(*ast.BasicLit).Value
    call := logic.Y.(*ast.CallExpr)

    switch call.Fun.(*ast.SelectorExpr).Sel.Name {
    case "Var":
        value = "?"
    case "In":
        switch binary.Op {
        case token.EQL:
            op = " IN "
        case token.NEQ:
            op = " NOT IN "
        }

        value = strings.Repeat("?, ", len(call.Args))
        value = value[:len(value)-2] // 去掉多余的逗号。
        value = "(" + value + ")"
    }

    expr = &ast.BasicLit{
        Kind: token.STRING,
        Value: strconv.Quote(col+op+value),
    }
    values = call.Args
    return
}
```
