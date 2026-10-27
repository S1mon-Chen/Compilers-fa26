# LEC 8 · AST and Name Resolving

> SE3355 编译原理（2026 秋）  
> 依据：[LEC-8-AST-and-Name-Resolving.pptx](../lectures/LEC-8-AST-and-Name-Resolving.pptx)，共 52 页。以下页码指课件页码。  
> 本笔记按主题整理；标注“补充理解”“补充实现”或“补充例子”的内容用于解释算法及其边界，不是课件原文。语言名字空间的例子附官方资料链接。  
> 前置：[LEC 5 · Top-Down Parsing](LEC-5-Top-Down-Parsing.md)、[LEC 6 · Introduction to LR Parsing](LEC-6-Intro-to-LR-Parsing.md)、[LEC 7 · Constructing LR Parsing Generators](LEC-7-Constructing-LR-Parsing-Generators.md)。

## 1. 本讲主线：程序结构与声明身份（p.2、37、51–52）

前几讲解决了如何根据文法识别 Token 序列，以及如何记录它的推导结构。本讲进一步把解析结果组织成后续编译阶段能够直接使用的数据模型。

```text
Source Code → Token Stream → Parser
                               │ semantic action
                               ▼
                              AST
                               │ Resolver：按作用域查找声明
                               ▼
                           Bound AST
                               │ Type Checker
                               ▼
                        类型信息与语义检查结果
```

Parser 可以先生成 Parse Tree，再转换成 AST；也可以在解析时直接构造 AST，无须实际保存完整的 Parse Tree。

| 表示 / 阶段 | 主要回答的问题 | 例子 |
| --- | --- | --- |
| Parse Tree | 输入按哪些产生式匹配？ | `expr → expr PLUS term` |
| AST | 程序有哪些构造，它们怎样组合？ | `BinaryExpr(ADD, NameExpr("x"), IntExpr(1))` |
| 名字解析 / Binding | 这里使用的名字指向哪个声明？ | `x → 参数 Symbol #2` |
| 类型检查 | 已确定身份的对象能否参与当前运算？ | 参数 `x` 与整数 `1` 能否相加？ |

**本讲的两个核心结果：AST 保存程序结构，Bound AST 进一步保存名字使用与声明之间的稳定连接。**

## 2. Parse Tree 与 AST（p.3–10）

### 2.1 Parser 已经能够构造 Parse Tree

在输出 accept / error 的同时，Parser 可以记录匹配过程：

- **LL：** 展开产生式时连接子节点，匹配终结符时填入 Token。
- **LR：** shift 时记录 Token，reduce 时把右侧的子树组合成左侧非终结符的子树。

例如归约 `E → E PLUS T`，可以弹出对应的三棵子树，接到一个新的 `E` 节点下，再把新子树压回栈。

这棵树忠实反映文法，但后续阶段往往不需要重复处理所有文法层次。

### 2.2 AST 抽象掉语法脚手架

对于：

```c
return y * 2;
```

后续阶段真正关心的是：返回值为一个乘法表达式，左操作数是 `y`，右操作数是 `2`。可以表示为：

```text
ReturnStmt
└── BinaryExpr(MUL)
    ├── NameExpr("y")
    └── IntExpr(2)
```

AST 通常省略以下内容：

| 可以省略的结构 | 省略的原因 |
| --- | --- |
| `expr / term / factor` 等中间非终结符 | 它们主要用于组织文法层次 |
| 分号、分隔逗号、部分括号 | 其作用已经由节点种类、列表或父子关系体现 |
| 消除左递归后引入的 `expr_tail` | 它服务于 Parser 的分析方式 |
| 单独的优先级与结合性标记 | 正确的 AST 形状已经体现了分组关系 |

“省略 Token 节点”不等于丢掉 Token 的全部信息。例如 `PLUS` 可以不再是一片叶子，但它对应的 `ADD` 运算符种类必须保留。

### 2.3 AST 必须保留什么

| 信息 | 示例 | 后续用途 |
| --- | --- | --- |
| 节点种类 | `IfStmt`、`CallExpr`、`BinaryExpr` | 确定当前构造的处理规则 |
| 有角色的子节点关系 | `condition`、`thenBranch`、`elseBranch` | 区分条件与分支等不同位置 |
| Token 携带的值 | 名字、字面量值、运算符种类 | 名字解析、类型检查、代码生成 |
| 源码位置 | `SourceSpan(start, end)` | 错误定位、IDE 跳转、源码映射 |

### 2.4 表示由消费者决定

编译器的名字解析、类型检查和 lowering 需要清晰的程序结构。格式化器、重构工具和 IDE 则可能需要保留注释、原始括号以及尚未完成的语法。

因此，真实工具可以同时维护具体语法表示和 AST；不存在适合所有用途的唯一 AST。

**补充理解：** 普通 Parse Tree 不自动等于“无损源码表示”。若词法阶段已经丢弃空白和注释，Parse Tree 也无法凭空恢复它们。需要无损重建源码时，应显式保留这些 trivia 或相应的原文信息。

## 3. AST 节点设计：用类型表达角色（p.11–17）

### 3.1 节点不能任意组合

课件将节点分为四类：

| 类别 | 含义 | 示例 |
| --- | --- | --- |
| `Decl` | 声明 | `FunctionDecl`、`VarDecl` |
| `Stmt` | 语句 | `BlockStmt`、`IfStmt`、`ReturnStmt` |
| `Expr` | 表达式 | `BinaryExpr`、`CallExpr`、`NameExpr` |
| `TypeSyntax` | 源码中的类型语法 | `NamedType`、`ArrayType` |

例如 `IfStmt.condition` 必须是表达式，两个分支则是语句。这种约束直接进入 AST API，能避免把函数声明误放到条件表达式位置。

### 3.2 表达式、语句与声明的字段

以下为课件的 Java 风格定义，省略构造函数及公共字段：

```java
abstract class Expr {}

class IntExpr extends Expr {
    int value;
}

class NameExpr extends Expr {
    String name;
}

class BinaryExpr extends Expr {
    BinaryOp op;
    Expr left;
    Expr right;
}

class IfStmt extends Stmt {
    Expr condition;
    Stmt thenBranch;
    Stmt elseBranch;   // 可以为 null，表示没有 else
}

class FunctionDecl extends Decl {
    String name;
    List<Param> params;
    TypeSyntax returnType;
    BlockStmt body;
}
```

于是 `x + 1` 表示为：

```text
BinaryExpr(
    op = ADD,
    left = NameExpr("x"),
    right = IntExpr(1)
)
```

后续 Pass 只需要处理 `BinaryExpr`，不用询问它经过了多少层 `expr / term / factor`。

**补充理解：** `TypeSyntax` 描述源码怎样写一个类型，还不是类型检查器最终使用的语义类型。例如 `NamedType("User")` 仍需要确定 `User` 指向哪个类型声明。

### 3.3 优先级与结合性体现在树形中

```text
1 + 2 * 3                 (1 + 2) * 3

Add                       Mul
├── Int(1)                ├── Add
└── Mul                   │   ├── Int(1)
    ├── Int(2)            │   └── Int(2)
    └── Int(3)            └── Int(3)
```

左图把 `2 * 3` 放在同一子树中，体现乘法优先。右图中括号改变了分组，构树后通常可以删除括号节点，但必须保留它造成的树形变化。

**补充例子：** 左结合的减法应生成：

```text
10 - 3 - 2 → Sub(Sub(Int(10), Int(3)), Int(2))
```

这与 `Sub(Int(10), Sub(Int(3), Int(2)))` 不同，二者分别得到 `5` 和 `9`。

**补充理解：** AST 的分组关系与运行时操作数的求值顺序是两个问题。树形确定如何组合运算结果；是否先计算左操作数、短路时是否计算右操作数，还需要遵循语言的语义规则。

## 4. Surface AST、Core AST 与去糖（p.18–20）

### 4.1 语法糖与 desugaring

语法糖使用更方便的表层写法表达已有核心构造能够表达的行为。课件列举：

| 表层写法 | 对应的核心机制 |
| --- | --- |
| `x += y` | 赋值与加法 |
| `for (...) { ... }` | 循环与顺序执行 |
| `foreach (x in xs)` | 迭代协议 |
| `user?.address` | 空值判断与字段访问 |

把表层构造转换为核心构造的过程称为 **desugaring（去糖）**。

```text
Parse Tree → Surface AST → desugar → Core AST
                 │                      │
              ForStmt           BlockStmt + WhileStmt
```

### 4.2 何时去糖是一项设计选择

| 选择 | 收益 | 代价 |
| --- | --- | --- |
| 暂时保留 `ForStmt` | 更接近源码，诊断和源码工具处理自然 | 后续 Pass 需要认识更多节点 |
| 尽早转成核心构造 | 后续 Pass 处理的节点种类更少 | 必须保存来源信息，保证变换语义正确 |

课件用下面的改写说明基本方向：

```text
for (init; cond; step) { body }

    ↓ 示意展开

init;
while (cond) {
    body;
    step;
}
```

**补充理解：这不是对所有程序都可直接套用的文本替换。** 实际去糖至少要考虑：

- 若 `init` 声明了循环局部变量，需要保留它原有的作用域。
- 对 C 风格 `for`，指向该循环的 `continue` 仍应经过 `step`。直接保留成 `while` 的 `continue` 会跳过尾部 `step`。
- 带副作用的子表达式必须保持原有求值次数。例如不能把 `a[i++] += y` 简单复制成 `a[i++] = a[i++] + y`。

所以去糖必须保留程序含义，并为生成的节点保存适当的源码映射。

## 5. 源码位置与不完整 AST（p.21–22）

### 5.1 SourceSpan 随 AST 向后传递

```java
class SourceSpan {
    int startOffset;
    int endOffset;
}
```

课件使用左闭右开的区间 `[startOffset, endOffset)`。例如从 0 开始计数：

```text
int y = x + 1;
        ^^^^^
        [8, 13)
```

`BinaryExpr` 的范围覆盖 `x + 1`。Token 提供初始位置信息，AST 及后续变换后的表示继续保存来源，供诊断、跳转与高亮使用。

**补充实现：** 多文件编译通常还需要文件身份。偏移量究竟按字节、字符还是其他单位计数，也需要在整个前端保持一致。

### 5.2 ErrorExpr 保留错误附近的结构

对于不完整输入：

```c
int y = x + ;
return y;
```

Parser 可以构造：

```text
VarDecl(
    name = "y",
    init = BinaryExpr(ADD, NameExpr("x"), ErrorExpr(span))
)
```

这样，后面的 `return y` 仍有机会继续解析和分析。

**补充理解：** 错误占位节点需要配合 Parser 的同步与恢复策略。`ErrorExpr` 保存缺失位置；后续 Pass 也必须识别它，减少由同一缺失操作数引起的连锁诊断。

## 6. 语义、语义值与 semantic action（p.23–25）

这三个术语不能混为一谈：

| 术语 | 含义 | 示例 |
| --- | --- | --- |
| Semantics，语义 | 程序表示什么、执行时发生什么 | `1 + 2` 得到 `3`；赋值改变状态 |
| Semantic value，语义值 | 解析时附着在语法符号上、供组合的数据 | `INT.value = 1`、`expr.ast = 某棵子树` |
| Semantic action，语义动作 | 解析过程中读取已有值、计算新值的操作 | `expr.ast = BinaryExpr(ADD, left, right)` |

**构造 AST 时，语义值通常是节点或子树，并不要求在解析阶段就计算源程序的运行结果。**

本节采用课件中的两套表达式文法：

```text
LR：保留左递归

expr → expr PLUS term | term
term → IDENT | INT

LL：消除左递归

expr      → term expr_tail
expr_tail → PLUS term expr_tail | ε
term      → IDENT | INT
```

它们识别相同的 Token 序列，例如 `IDENT PLUS INT PLUS INT`。但它们的 Parse Tree 形状不同，因此需要正确设计 semantic action，才能得到相同的、符合预期结合性的 AST。

## 7. LR 构造 AST：在 reduce 时组合结果（p.26–28）

### 7.1 归约时右侧的语义值已经就绪

```text
expr → expr₁ PLUS term
       { expr.ast = BinaryExpr(ADD, expr₁.ast, term.ast); }
```

下标 `₁` 用来区分右侧出现的 `expr` 与左侧结果，不是另一个非终结符。

归约前后可理解为：

```text
归约前：… expr₁               PLUS       term
语义值：… expr₁.ast           PLUS 信息  term.ast

归约后：… expr
语义值：… BinaryExpr(ADD, expr₁.ast, term.ast)
```

状态栈仍按前几讲的规则弹出右侧并查 `GOTO`。与此同时，Parser 取出右侧语义值，计算左侧语义值，再保存新结果。

### 7.2 x + 1 的构造过程

输入：`IDENT(x) PLUS INT(1)`。下表只列 reduce，省略中间的 shift：

| 顺序 | 归约规则 | 语义动作 | 新语义值 |
| --- | --- | --- | --- |
| 1 | `term → IDENT` | `term.ast = NameExpr(IDENT.lexeme)` | `NameExpr("x")` |
| 2 | `expr → term` | `expr.ast = term.ast` | 同一棵 `NameExpr("x")` 子树 |
| 3 | `term → INT` | `term.ast = IntExpr(INT.value)` | `IntExpr(1)` |
| 4 | `expr → expr₁ PLUS term` | 构造 `BinaryExpr(ADD, expr₁.ast, term.ast)` | 整棵加法 AST |

第二步直接传递子树，没有为 `expr → term` 创建额外的 AST 包装节点。这正是省略语法脚手架的一种实现方式。

### 7.3 `$$`、`$1`、`$2`、`$3` 的含义

Parser generator 常按产生式中的位置引用语义值：

```text
expr → expr₁ PLUS term
 $$      $1    $2   $3

{ $$ = BinaryExpr(ADD, $1, $3); }
```

- `$$`：左侧符号的语义值。
- `$i`：右侧第 `i` 个符号的语义值。
- 本例通过产生式已经知道运算符是加法，因此无需读取 `$2`。

**位置编号仍然包含 `PLUS`。** 不使用 `$2`，不表示可以把右侧的 `term` 改称 `$2`。

**补充实现：** 可以使用与符号位置对应的语义值栈，也可以将值放进栈帧。无论选哪种表示，都必须维持“值属于哪一次符号出现”的对应关系。

## 8. LL 构造 AST：把 action 放入分析栈（p.29–34）

### 8.1 消除左递归后，AST 仍可保持左结合

把 `PLUS` 换成 `MINUS`，更容易观察结合性：

```text
expr      → term expr_tail
expr_tail → MINUS term expr_tail | ε
```

虽然 `expr_tail` 向右递归，但预期的减法 AST 仍然是左结合的。递归下降可以维护一个累积结果：

```text
parseExpr():
    left = parseTerm()
    while current == MINUS:
        consume(MINUS)
        right = parseTerm()
        left = BinaryExpr(SUB, left, right)
    return left
```

对 `10 - 3 - 2`，累积过程为：

```text
left = IntExpr(10)
left = Sub(IntExpr(10), IntExpr(3))
left = Sub(Sub(IntExpr(10), IntExpr(3)), IntExpr(2))
```

**右递归的是消除左递归后的文法结构；AST 的结合方向由构造动作决定。**

### 8.2 给表驱动 LL 增加 action 栈元素

表驱动 LL 没有显式 reduce，可以把 action 也放入产生式和分析栈。为了与课件 p.31、34 一致，**本节分析栈统一写成左侧为栈顶**。

```text
expr → term
       { A₁: expr_tail.in = term.ast; }
       expr_tail
       { A₂: expr.ast = expr_tail.ast; }
```

展开后，执行顺序为：

```text
栈顶 → term  A₁  expr_tail  A₂  …
```

因此 `term` 完成之后执行 `A₁`，给尚未展开的 `expr_tail` 传入已有 AST；`expr_tail` 完成之后执行 `A₂`，取回结果。

### 8.3 用 in 传入累积树，用 ast 传回结果

完整动作如下：

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

这里的 `expr_tail₀` 与 `expr_tail₁` 是同一非终结符在一次产生式应用中的不同出现位置：

- `expr_tail₀.in`：进入本层之前已经构造好的左侧 AST。
- `expr_tail₁.in`：把本次 `PLUS term` 接到左侧之后，传给下一层的 AST。
- `expr_tail₀.ast`：处理完剩余所有尾部后，传回的最终 AST。
- ε 分支不再添加运算，直接把传入的累积结果作为输出。

对应的递归形式更容易看出数据流：

```text
parseTail(left):
    if current == PLUS:
        consume(PLUS)
        right = parseTerm()
        nextLeft = BinaryExpr(ADD, left, right)
        return parseTail(nextLeft)
    else if current ∈ FOLLOW(expr_tail):
        return left
    else:
        syntaxError()
```

**补充实现：** 上述函数是对课件属性传递的等价解释。ε 分支只能在分析表允许时选择，不能把任意未知 Token 都当作表达式结束。

### 8.4 action 的位置就是触发时机

| action 位置 | 执行时哪些值已经可用 | 用途 |
| --- | --- | --- |
| 产生式末尾 | 右侧结构已经完成 | 构造父节点、汇总或返回结果 |
| 两个符号之间 | 左侧已处理部分的值可用，后续符号尚未处理 | 为后续符号设置输入上下文 |

扩展后的通用循环可以写成：

```text
while stack is not empty:
    X = pop(stack)
    if X is an action:
        execute(X)
    else if X is a terminal:
        matchAndRecordToken(X)
        consume()
    else:
        production = table[X, current]
        if production is absent:
            syntaxError()
        else:
            push production's symbols and actions in reverse order
```

反向压栈使产生式最左侧的符号最先执行。action 与 ε 都不消费输入。

**补充实现：** 实际栈元素还应携带对应产生式实例的属性槽引用或上下文。不能给所有 `expr_tail` 共用一个全局 `.in` / `.ast`，否则递归展开会覆盖尚未返回的结果。

### 8.5 手算：表驱动 LL 构造 x + 1

采用上述动作编号。`$` 表示输入结束；表中状态为该步动作完成之后，省略部分常规的 `term` 展开步骤。

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

`tail` 是表中对 `expr_tail` 的缩写。最终得到的 AST 与第 7 节的 LR 构造结果相同。

## 9. 综合属性与继承属性（p.35–36）

| 属性 | 信息流向 | 本讲例子 | 代码中的常见对应 |
| --- | --- | --- | --- |
| 综合属性（synthesized attribute） | 从子结构的结果计算当前结构的结果 | `expr.ast = term.ast` | 函数返回值 |
| 继承属性（inherited attribute） | 从父结构或已有上下文传入当前子结构 | `expr_tail.in = term.ast` | 函数参数 |

例如：

```text
expr → term { tail.in = term.ast; } tail { expr.ast = tail.ast; }
             └── 向后传入上下文 ──┘      └── 向上传回结果 ──┘
```

本讲中 LR 在 reduce 时组合 AST，是综合属性的直接应用；LL 则通过中间 action 传入累积结果，再通过末尾 action 返回结果。

**补充理解：** “继承”描述数据来源，不表示面向对象的类继承。也不能据此断言 LL 只用继承属性、LR 绝不传递上下文；这里讨论的是本讲的构树方案。

## 10. 名字解析：把使用连接到声明（p.37–41、47）

### 10.1 NameExpr 起初只保存文本

Parser 看到表达式中的 `x`，可以构造 `NameExpr("x")`。但仅凭这个字符串，还不知道它代表全局变量、参数还是某个内层局部变量。

课件使用如下例子：

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

`#1` 等编号用于表示声明身份，不要求按源码出现顺序分配。解析结果为：

| 使用位置 | 名字 | 绑定结果 |
| --- | --- | --- |
| U1：`y` 的初始化表达式 | `x` | 参数 `x #2` |
| U2：内层 `x` 的初始化表达式 | `y` | 函数中的局部变量 `y #3` |
| U3：内层 `return` | `x` | 内层局部变量 `x #4` |

三个 `x` 的拼写相同，但它们具有不同的声明身份。

### 10.2 Symbol 与 Symbol Table

| 概念 | 职责 |
| --- | --- |
| 名字字符串 | 保存源码拼写，如 `"x"` |
| Symbol | 表示声明身份，如参数 `x #2` |
| Symbol Table | 按名字查找 Symbol 的映射 |
| Binding | 某个使用节点到其所指 Symbol 的连接 |

Symbol 可以记录 `name`、`kind` 及其他声明信息。后续阶段区分变量时，应比较 Symbol 身份，而不是只比较名字字符串。

### 10.3 AST → Bound AST

```text
解析前：                          解析后：

ReturnStmt                      ReturnStmt
└── NameExpr                    └── BoundNameExpr
    └── name: "x"                   ├── name: "x"
                                    └── symbol: LocalSymbol #4
```

Bound AST 的整体程序结构基本不变，名字节点增加了确定的 binding。原始名字仍可保留，用于打印和诊断。

**补充实现：** 可以创建新的一组 Bound 节点，也可以给原节点写入 `symbol` 字段，或使用 `NodeId → Symbol` 的旁表。本讲的关键是保存绑定结果，而不是要求某一种对象布局。

## 11. 词法作用域与 scope stack（p.42–46）

### 11.1 从最内层向外查找

词法作用域（lexical scope）依据源码中的嵌套结构确定名字的含义。对于本讲的简单名字查找，从使用位置所在作用域向外寻找，优先选择最近的可见声明。

在上述内层块中：

```text
栈顶 → 内层块：{ x → #4 }
       函数层：{ x → #2, y → #3 }
       全局层：{ x → #1, f → #5 }

lookup("x") → #4
lookup("y") → #3
lookup("f") → #5
```

内层声明使同名外层声明暂时不能通过简单名字访问，称为 **shadowing（遮蔽）**。

**补充理解：** 这里的“向外”沿源码的词法嵌套关系，不是沿运行时函数调用链。Resolver 的 scope stack 也不是程序执行时的调用栈。

### 11.2 四个基本操作

| 操作 | 行为 |
| --- | --- |
| `beginScope()` | 压入一张空的 `name → Symbol` 表 |
| `declare(name, symbol)` | 在当前作用域登记声明 |
| `lookup(name)` | 从栈顶向下搜索，返回第一次命中的 Symbol |
| `endScope()` | 弹出当前作用域 |

**补充实现：** 查找可直接按作用域的嵌套顺序进行：

```text
lookup(name):
    for scope in scopes from innermost to outermost:
        if name in scope:
            return scope[name]
    return NOT_FOUND
```

命中后立即返回，体现“最近的可见声明优先”；全部查找失败时，再进入第 14 节的错误处理。

课件的遍历框架为：

```text
resolveBlock(block):
    beginScope()
    for item in block.items:
        resolve(item)
    endScope()

resolveName(node):
    node.symbol = lookup(node.name)
```

遇到表达式时继续遍历其子节点，遇到声明时登记 Symbol，遇到名字使用时查找并绑定。

**补充实现：** 若实现会提前返回或抛出异常，应使用 `finally`、RAII 等方式保证作用域成对进入和退出。

### 11.3 完整状态变化

下表按课件示例，把参数与函数体最外层局部变量放在同一函数作用域。栈从左到右表示由内向外，即左侧为栈顶。

| 遍历事件 | Scope stack | 绑定或可见性变化 |
| --- | --- | --- |
| 已登记全局声明 | `{x#1, f#5}` | 全局 `x`、`f` 可见 |
| 进入函数，登记参数 `x#2` | `{x#2} → {x#1, f#5}` | 参数遮蔽全局 `x` |
| 登记 `y#3`，处理 `x + 1` | `{x#2, y#3} → {x#1, f#5}` | U1：`x → #2` |
| 进入内层块，登记 `x#4` | `{x#4} → {x#2, y#3} → {x#1, f#5}` | 内层 `x` 遮蔽参数 |
| 处理 `y * 2` | 同上 | U2：`y → #3` |
| 处理 `return x` | 同上 | U3：`x → #4` |
| 离开内层块 | `{x#2, y#3} → {x#1, f#5}` | 后续简单名字 `x` 再次查到 `#2` |
| 离开函数 | `{x#1, f#5}` | 函数局部名字不再参与当前查找 |

**补充理解：弹出作用域不等于销毁已绑定的 Symbol。** `endScope()` 撤下的是当前查找环境。Bound AST 中原有的 `symbol = #4` 仍然有效，后续类型检查和代码生成还会使用它。编译器内部 Symbol 的存活时间与源程序中变量的运行时生命周期也应区分。

## 12. 声明顺序与语言规则（补充实现）

课件给出了作用域栈的核心机制，但完整 Resolver 还需要依据目标语言规定声明何时可见。以下是实现时需要明确的边界。

### 12.1 遮蔽与同层重复声明

对于“允许内层遮蔽、禁止同一作用域重复声明”的简单语言，登记过程可以写成：

```text
declare(name, symbol):
    if name in scopes.top:
        reportDuplicateDeclaration(name)
        return failure
    scopes.top[name] = symbol
    return success
```

重复声明检查只查当前层；名字使用的 `lookup` 则查所有可见层。如果用完整 `lookup` 判断“名字是否已经声明”，就可能错误地拒绝合法的内层遮蔽。

支持重载或声明合并的语言，还需要更具体的同名处理规则。

### 12.2 声明何时进入作用域

```c
int x = 1;
{
    int x = x + 1;
}
```

初始化表达式中的 `x` 指向哪里，取决于语言规则：

| 规则选择 | Resolver 的处理方向 |
| --- | --- |
| 新声明从初始化完成后才可见 | 先解析初始化式，再登记新名字 |
| 新声明在初始化式中已经可见 | 先登记，再解析初始化式 |
| 已进入作用域，但尚不允许读取 | 先登记并标为未完成，再在使用处检查状态 |

本讲的 `int y = x + 1` 没有同名自引用，因此不能用它推导所有语言的初始化可见性规则。**确定 binding 与判断“读取前是否已经初始化”也可能属于不同的检查步骤。**

### 12.3 前向引用与递归

如果语言允许函数在声明之前被调用，或允许两个函数相互递归，单纯按源码顺序“遇到声明才登记”可能不够。

一种常见处理方式是：

```text
第一遍：收集当前声明区域中的函数名字，建立 Symbol
第二遍：进入各个函数体，解析参数、局部声明与名字使用
```

是否预先收集、收集哪些声明，都由语言规则决定。局部变量不一定与函数使用同一套可见性策略。

同样，函数参数与最外层函数体是否共享作用域，需要在实现中明确；不能无条件给每层花括号额外建表，再假定它自然符合所有语言的规则。

## 13. 多种名字空间（p.48–49）

### 13.1 拼写之外，还需要知道名字类别

真实语言不一定只使用一张统一的 `name → Symbol` 表。查找可能区分：

| 名字类别 | 示例 | 查找接口示意 |
| --- | --- | --- |
| 值名字 | 变量、参数、函数 | `lookupValue(name)` |
| 类型名字 | 类、结构体、类型别名 | `lookupType(name)` |
| 成员名字 | 字段、方法、模块成员 | `lookupMember(owner, name)` |

这里的 namespace 指标识符的分类，不特指 C++ 的 `namespace` 语法。

**补充理解：** scope 回答“在哪些源码区域中可见”，namespace 回答“在当前语法上下文中查哪一类名字”。二者是不同维度，查找常常需要同时考虑。

### 13.2 课件中的语言对比

| 语言 | 课件强调的规则 | 示例或影响 |
| --- | --- | --- |
| Go | 同一 package block 内，普通类型名、函数名与变量名不能因类别不同而重复声明 | `type User struct{}` 与 `func User() {}` 冲突 |
| TypeScript | 类型与值可以分别占据不同的名字类别 | `type User = ...` 与 `const User = ...` 可以共存 |
| Rust | 区分 type、value、macro 等名字空间 | 相同拼写是否冲突，要看声明进入哪个名字空间 |

Go 的示例对应规范中的同一 block 不得重复声明同一标识符规则。参见 [Go 规范：Declarations and scope](https://go.dev/ref/spec#Declarations_and_scope)。

TypeScript 的一个具体例子是：

```typescript
type User = { name: string };
const User = { name: "Alice" };

const u: User = User;
//       ↑类型   ↑值
```

**补充理解：** TypeScript 的声明可能创建 namespace、type、value 中的一种或多种实体，例如 class 同时创建类型和值。因此不能把规则概括为“任何同名声明都可以共存”。参见 [TypeScript Handbook：Declaration Merging / Basic Concepts](https://www.typescriptlang.org/docs/handbook/declaration-merging.html#basic-concepts)。

Rust 也不只包含课件列出的三类，还区分 lifetime、label 等名字空间。具体声明所属的类别见 [Rust Reference：Namespaces](https://doc.rust-lang.org/reference/names/namespaces.html)。

划分更多名字空间可以增加命名自由，但也要求解析器、导入机制和诊断系统准确区分查找上下文。

## 14. 名字解析的错误恢复（p.50）

```c
int total = 0;
return totla + 1;
```

查找 `totla` 失败后，Resolver 可以报告未声明名字，并把该使用绑定到一个特殊的错误 Symbol：

```text
lookup("totla") → not found

BoundNameExpr(
    name = "totla",
    symbol = ErrorSymbol("totla")
)
```

这样仍能保留完整表达式结构，后续阶段不必因为普通名字节点缺少绑定而崩溃。

| 错误占位 | 出现阶段 | 保留的信息 |
| --- | --- | --- |
| `ErrorExpr` | 语法分析 / AST 构造 | 缺失或错误的表达式位置 |
| `ErrorSymbol` | 名字解析 | 已识别的名字使用，但未找到有效声明 |

**补充实现：** 拼写建议只是诊断信息。即使提示“是否想写 `total`”，也不能悄悄把 `totla` 绑定成 `total`，否则就改变了用户程序的含义。后续检查应识别错误标记，抑制无意义的重复诊断。

## 15. Bound AST 与类型检查的交接（p.51–52）

完成名字解析后，表达式可以表示为：

```text
BoundBinaryExpr(ADD)
├── BoundNameExpr(symbol = 参数 x #2)
└── IntExpr(1)

类型环境 Γ：
    Symbol #2 → int
```

此时类型检查可以根据声明身份查询类型，继续判断：

1. 左、右操作数的类型分别是什么？
2. 当前 `ADD` 是否允许这两个类型组合？
3. 整个表达式的结果类型是什么？

名字解析已经回答了“这里的 `x` 是谁”，但还没有完成所有语义检查。例如 `x` 成功绑定，并不能直接推出 `x + 1` 类型正确。

**补充理解：** 本讲按阶段展示简单名字解析。在具有重载、成员查找和类型推断的真实语言中，某些名字解析可能需要类型信息，阶段间会发生协作；不能假定所有语言都能一次性把每个调用解析到唯一函数。

## 16. 易错点速查

| 易错点 | 正确理解 |
| --- | --- |
| AST 就是把 Parse Tree 的所有叶子删掉 | 名字、字面量值、运算符和必要结构仍须保留 |
| Parser 必须先保存完整 Parse Tree 才能构造 AST | 可以直接用 semantic action 构造 AST |
| 删除括号就是忽略括号作用 | 括号引起的分组必须先体现在树形中 |
| AST 节点越少越好 | 保留哪些构造取决于后续消费者及诊断需求 |
| 去糖只需做文本替换 | 必须保持作用域、控制流和求值次数等语义 |
| Semantic value 就是源程序算出来的值 | 它也可以是 AST、Token 信息、列表或其他解析数据 |
| `$$` 是输入结束标记 | `$$` 指左侧语义值；本笔记单独的 `$` 才表示输入结束 |
| 不读取 `$2` 就能把 `$3` 改写成 `$2` | 编号由产生式右侧位置决定 |
| LL 没有 reduce，所以无法在解析时建树 | 可以在分析栈中安排中间与末尾 action |
| 消除左递归后，AST 必然右结合 | 可以用继承属性或循环向左累积 |
| action 执行时会消费一个 Token | 消费输入的是终结符匹配，action 只处理语义值 |
| 所有非终结符出现可以共用一个属性槽 | 属性必须对应具体的出现实例 |
| 名字相同就代表同一个变量 | 不同声明具有不同 Symbol 身份 |
| 检查重复声明与查找名字使用使用同一搜索范围 | 简单模型中前者查当前层，后者由内向外查找 |
| 离开块就能销毁该块的所有 Symbol | 后续阶段仍可能通过 Bound AST 引用它们 |
| Scope stack 就是运行时调用栈 | 它表示编译期间的词法嵌套环境 |
| 名字解析成功就说明表达式合法 | 还需类型检查及其他语义检查 |
| 报错后必须立即丢弃整棵树 | 可以用 `ErrorExpr` / `ErrorSymbol` 保留结构并继续 |

## 17. 复习自测与简答

1. **`return y * 2;` 的 AST 最少需要哪些语义构造？**  
   一个返回语句、一个乘法表达式、一个名字表达式和一个整数字面量；同时保留必要的源码位置。

2. **为什么 `expr → term` 通常不需要创建新 AST 节点？**  
   它只是文法层次的传递，可以直接令 `expr.ast = term.ast`。

3. **构造 `expr → expr PLUS term` 时，为什么 LR 的 reduce 是合适时机？**  
   右侧已经识别完毕，左右操作数的 AST 都已就绪，可以立即组合父节点。

4. **LL 的中间 action 为什么必须放在 `term` 后、`expr_tail` 前？**  
   它要读取已经完成的 `term.ast`，并在尾部开始解析前，把该结果设为尾部的输入上下文。

5. **`expr_tail → ε` 的动作返回什么？**  
   返回传入的累积 AST，即 `expr_tail.ast = expr_tail.in`。

6. **怎样让 `10 - 3 - 2` 保持左结合？**  
   每读完一个右操作数，构造 `left = Sub(left, right)`，最后得到 `Sub(Sub(10, 3), 2)`。

7. **综合属性与继承属性最直观的代码类比是什么？**  
   综合属性像返回值，继承属性像参数；前者传回子结构结果，后者传入已有上下文。

8. **第 10 节三个使用 U1、U2、U3 分别绑定到谁？**  
   U1 的 `x → #2`，U2 的 `y → #3`，U3 的 `x → #4`。

9. **为什么内层 `x` 不会覆盖掉外层 Symbol 对象？**  
   它被登记到另一张作用域表中，查找顺序使它优先命中。退出内层后，外层表仍然存在。

10. **退出作用域后，之前的 `BoundNameExpr(symbol = #4)` 会失效吗？**  
    不应失效。退出只改变当前可见环境，已经建立的 binding 仍供后续阶段使用。

11. **若函数允许前向调用，Resolver 为什么可能需要两遍？**  
    第一遍登记函数身份，第二遍处理函数体，才能让较早的使用找到较晚的声明。

12. **`ErrorExpr` 和 `ErrorSymbol` 的区别是什么？**  
    前者占据错误或缺失的表达式位置；后者为已解析但无法绑定的名字提供错误身份。

13. **为什么类型环境用 `Symbol → Type` 比只用 `String → Type` 更合适？**  
    不同作用域可以有同名声明，Symbol 能明确区分它们，避免再次依赖字符串与临时作用域查找。

14. **本讲结束后，类型检查器从前端获得了什么？**  
    程序结构、源码位置、已确定的声明身份，以及需要继续传播和处理的错误标记。
