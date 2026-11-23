# LEC 8 · 抽象语法树与名字解析

SE3355 编译原理 · 2026 秋　[本讲课件（52 页）](../lectures/LEC-8-AST.pptx)

`return y * 2;` 的语法分析树能回答“用了哪些产生式”。类型检查器关心的是“返回了什么、谁和谁相乘、`y` 来自哪个声明”。前两个问题看抽象语法树（AST）就能回答，最后一个需要名字解析。

## AST 应保留哪些信息（p.3–22）

```text
ReturnStmt
└── BinaryExpr(MUL)
    ├── NameExpr("y")
    └── IntExpr(2)
```

AST 通常省去分号、括号、`expr → term` 等中间层，保留运算符、操作数的左右顺序、声明、语句以及源码位置（span）。`1 + 2 * 3` 和 `(1 + 2) * 3` 的根节点分别是加法和乘法。即使删掉括号节点，树形也保留了分组。需要原样还原源码的格式化器则要保留具体语法或原始 Token。

| 节点类别 | 字段示例 | 由此能查出什么 |
| --- | --- | --- |
| `Expr` | `BinaryExpr(op, left: Expr, right: Expr)` | 两个操作数及其顺序 |
| `Stmt` | `IfStmt(condition: Expr, thenBranch: Stmt, elseBranch: Stmt?)` | 条件、两个分支，以及有无 else |
| `Decl` | `FunctionDecl(name, params, returnType: TypeSyntax, body)` | 函数签名和函数体 |
| `TypeSyntax` | `NamedType("User")` | 用户在源码里写的类型名，稍后再绑定到类型声明 |

**去糖**（desugaring）把方便书写的语法改成少量核心构造。可以在 *surface AST* 中保留 `ForStmt` 以便诊断，再在 *core AST* 中把它改成初始化、条件、循环体和步进。对 C 风格的 `for (init; cond; step)`，执行指向该循环的 `continue` 后也要执行 `step`。若直接挪到 `while` 循环体末尾，`continue` 会跳过这一步。`init` 中声明的变量也要留在原作用域。类似地，`a[i++] += y` 要让 `i++` 只执行一次。

在 `int y = x + ;` 中，可以用 `Add(Name("x"), ErrorExpr(span))` 占住缺失的右操作数。这样还能检查后面的语句，并把诊断标在缺口处。源码位置也应随去糖结果保留，便于把错误指回用户写下的 `for` 或 `+=`。

## LR 在归约时建树（p.23–28）

文法符号除了表示“匹配成功”，还可以携带**语义值**，例如整数、源码位置或 AST 节点。一条产生式上的**语义动作**说明如何用右侧的值计算左侧的值。在本例中，归约时创建一个加法节点。节点描述了程序将执行的操作。

| 顺序 | 归约规则 | 语义动作 | 新语义值 |
| --- | --- | --- | --- |
| 1 | `term → IDENT` | `term.ast = NameExpr(IDENT.lexeme)` | `NameExpr("x")` |
| 2 | `expr → term` | `expr.ast = term.ast` | 同一棵 `NameExpr("x")` 子树 |
| 3 | `term → INT` | `term.ast = IntExpr(INT.value)` | `IntExpr(1)` |
| 4 | `expr → expr₁ PLUS term` | 构造 `BinaryExpr(ADD, expr₁.ast, term.ast)` | 整棵加法 AST |

用生成器常见的记法，`expr → expr PLUS term` 的动作是 `$$ = BinaryExpr(ADD, $1, $3)`。`$$` 表示归约后的结果，`$1` 和 `$3` 是右侧第一、第三个符号的值。状态栈负责选择动作，语义值栈负责提供孩子。归约 `expr → term` 时可以直接传回已有子树。

## LL 怎样建出同一棵左结合树（p.29–36）

LL 文法把连续加法改成 `expr → term tail`、`tail → PLUS term tail | ε`。解析 `x + 1 + 2` 时，先把 `Name(x)` 传给尾部，读到 `1` 后构造 `Add(x,1)`，再把整棵树传给下一个尾部。最后返回 `Add(Add(x,1),2)`。

表驱动 LL 可以把动作也压入分析栈，轮到动作时执行，不读 Token。动作执行时，写在它前面的符号都已匹配。把动作放在末尾，就能使用全部孩子的值。下面 `in` 是传入的左侧结果，`ast` 是向上传回的结果。

```text
expr → term
       { A₁: expr_tail.in = term.ast; }
       expr_tail
       { A₂: expr.ast = expr_tail.ast; }

expr_tail₀ → PLUS term
             { A₃: expr_tail₁.in =
                   BinaryExpr(ADD, expr_tail₀.in, term.ast); }
             expr_tail₁
             { A₄: expr_tail₀.ast = expr_tail₁.ast; }

expr_tail₀ → ε
             { Aε: expr_tail₀.ast = expr_tail₀.in; }

term → IDENT { Aname: term.ast = NameExpr(IDENT.lexeme); }
term → INT   { Aint:  term.ast = IntExpr(INT.value); }
```

`in` 是**继承属性**，类似递归函数的参数。`ast` 是**综合属性**，类似返回值。下表把 `expr_tail` 简记为 `tail`。从第一次匹配 x 到最后一次返回 ast，输入始终只向前移动。

| 已执行的动作 | 分析栈，左侧为栈顶 | 未消费输入 | 语义值变化 |
| --- | --- | --- | --- |
| 展开 `expr` | `term A₁ tail₀ A₂ $` | `IDENT(x) PLUS INT(1) $` | — |
| 匹配 `IDENT`，执行 `Aname` | `A₁ tail₀ A₂ $` | `PLUS INT(1) $` | `term.ast = NameExpr("x")` |
| 执行 `A₁`，展开 `tail₀` | `PLUS term A₃ tail₁ A₄ A₂ $` | `PLUS INT(1) $` | `tail₀.in = NameExpr("x")` |
| 匹配 `PLUS`、`INT`，执行 `Aint` | `A₃ tail₁ A₄ A₂ $` | `$` | 当前 `term.ast = IntExpr(1)` |
| 执行 `A₃` | `tail₁ A₄ A₂ $` | `$` | `tail₁.in = Add(NameExpr("x"), IntExpr(1))` |
| `tail₁` 选 ε，执行 `Aε` | `A₄ A₂ $` | `$` | `tail₁.ast = tail₁.in` |
| 执行 `A₄` | `A₂ $` | `$` | `tail₀.ast = tail₁.ast` |
| 执行 `A₂` | `$` | `$` | `expr.ast = tail₀.ast` |

## 名字怎样连到声明（p.37–47）

`NameExpr("x")` 只记录拼写。名字解析器（Resolver）为每个声明建立一个有独立身份的 **Symbol**，再把每个使用点绑定到相应 Symbol。沿源码的嵌套关系寻找最近可见的声明，这叫词法作用域。

```c
int x = 1;                 // 全局 x：Symbol #1
int f(int x) {             // f：Symbol #5；参数 x：Symbol #2
    int y = x + 1;         // y：Symbol #3；U1 使用 x
    {
        int x = y * 2;     // 内层 x：Symbol #4；U2 使用 y
        return x;          // U3 使用 x
    }
}
```

| 使用位置 | 名字 | 绑定结果 |
| --- | --- | --- |
| U1 · `y` 的初始化表达式 | `x` | 参数 `x #2` |
| U2 · 内层 `x` 的初始化表达式 | `y` | 函数中的局部变量 `y #3` |
| U3 · 内层 `return` | `x` | 内层局部变量 `x #4` |

名字解析器维护一个栈，**每一层作用域用一张符号表**。进入作用域时压入空表，声明时在当前表登记，查名字时从栈顶向外找，离开时弹出整张表。内层可以遮蔽外层声明。处理重复声明时，只在当前层查重，是否允许重名由语言规定。离开作用域后，先前已绑定的节点仍保存各自的 Symbol。

| 遍历事件 | Scope stack | 绑定或可见性变化 |
| --- | --- | --- |
| 已登记全局声明 | `{x#1, f#5}` | 全局 `x`、`f` 可见 |
| 进入函数，登记参数 `x#2` | `{x#2} → {x#1, f#5}` | 参数遮蔽全局 `x` |
| 登记 `y#3`，处理 `x + 1` | `{x#2, y#3} → {x#1, f#5}` | U1 · `x → #2` |
| 进入内层块，登记 `x#4` | `{x#4} → {x#2, y#3} → {x#1, f#5}` | 内层 `x` 遮蔽参数 |
| 处理 `y * 2` | 同上 | U2 · `y → #3` |
| 处理 `return x` | 同上 | U3 · `x → #4` |
| 离开内层块 | `{x#2, y#3} → {x#1, f#5}` | 后续简单名字 `x` 再次查到 `#2` |
| 离开函数 | `{x#1, f#5}` | 函数局部名字不再参与当前查找 |

**声明从何时可见也要单独定。** 在外层已有 `x` 时，内层 `int x = x + 1;` 的右侧究竟指谁？若语言规定初始化结束后新名字才生效，就先解析右侧，再登记内层 `x`。若先登记，它会绑定到内层声明，还可能触发“未初始化就读取”的检查。若语言允许函数相互递归，可先登记同组函数名，再逐个进入函数体解析。

例如，把内层块中的 `return x;` 挪到内层的右花括号之后、函数的右花括号之前。只改一行的位置，使用点就从 `#4` 改绑到形参 `#2`。遍历器若在离开块时忘了弹表，就会在这个例子上出错。

## 名字空间、错误恢复和下一步（p.48–52）

作用域决定某个声明**在哪里**可见，名字空间决定查哪**一类**声明。查 `User` 时，类型标注和值表达式可能走不同的表。在 Go 的同一包块内，普通类型名和函数名重名会冲突。TypeScript 可以同时声明 `type User = …` 与 `const User = …`。Rust 也区分类型、值、宏等名字空间。[Go 规范](https://go.dev/ref/spec#Declarations_and_scope) · [TypeScript 声明类别](https://www.typescriptlang.org/docs/handbook/declaration-merging.html#basic-concepts) · [Rust 名字空间](https://doc.rust-lang.org/reference/names/namespaces.html)

若源码写了 `return totla + 1;`，而表里只有 `total`，可以在 `totla` 的 span 报错，绑定一个 `ErrorSymbol` 让后面的检查继续。即使给出拼写建议，也应保留这个错误绑定。合法的使用点会形成 *bound AST*。接下来类型检查器用 Symbol 查询声明的类型，并检查加法能否成立。[类型判断与类型环境](LEC-9-Type-Sys-I.md)
