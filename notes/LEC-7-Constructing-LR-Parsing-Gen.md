# LEC 7 · SLR、LR(1) 与 LALR 分析表

SE3355 编译原理 · 2026 秋　[本讲课件（68 页）](../lectures/LEC-7-Parsing-IV.pptx)

在 LR(0) 表中，看到 `term → INT ·` 就在每个终结符及结束标记那一列填归约。如果还有 `term → INT · LPAREN RPAREN`，左括号这一列就既要归约又要移进。本讲的几种算法沿用同一个状态栈和运行循环，最重要的区别是**完成项在哪些列允许归约**。[第 6 讲的状态栈与分析表](LEC-6-Intro2LR-Parsing.md)

| 方法 | 状态信息 | 普通完成项的归约条件 | 状态规模与代价 |
| --- | --- | --- | --- |
| LR(0) | 产生式与点位置 | 所有终结符及 `$` | 状态少，归约条件最粗 |
| SLR(1) | 同 LR(0) | `FOLLOW(A)` | 与 LR(0) 相同，额外计算 FOLLOW |
| LALR(1) | 同 core 合并后的 LR(1) item | 对应 item 的 lookahead 并集 | 保留 LR(0) 骨架，可能因合并产生冲突 |
| 规范 LR(1) | 产生式、点位置、具体 lookahead | 当前完成项的 lookahead | 上下文最精确，状态可能显著增多 |

## SLR 的 FOLLOW 能解决哪种冲突（p.3–14）

在 `expr → expr PLUS term | term` 中，`FOLLOW(term) = {PLUS, $}`。加上 `term → INT LPAREN RPAREN` 后，左括号只出现在一个尚未读完的 `term` 内部。SLR 只在 `PLUS` 和 `$` 上归约 `term → INT`，给 `LPAREN` 留下唯一动作 shift。

赋值文法能说明仅凭 FOLLOW 为什么仍会误判。

```text
start → stmt
stmt  → left EQUAL right
      | right
left  → STAR right
      | ID
right → left
```

识别完整的 `left` 后，一个项目要继续读 `EQUAL`，另一个项目 `right → left ·` 已经完成。`stmt → left EQUAL right` 让 `EQUAL` 进入 `FOLLOW(left)`，而 `left → STAR right` 又把它传进 `FOLLOW(right)`。SLR 因此仍在 `EQUAL` 一列发生冲突。

沿输入 `STAR ID EQUAL ID` 看，左边 `STAR right` 里的 `right` 后面确实可以有等号。但在起点直接把 `left` 归约成 `right` 的这条路径上，完成的是整个 `stmt → right`，后面应该是文件末尾。**FOLLOW 把这两个位置的后继合在了一起。**

## LR(1) 给每个项目单独算展望符（p.16–31、40–42）

`[A → α · B β, a]` 中，点后第一个待识别的符号是 B。展望符 a 说的是**整条 A 的右侧完成后**允许看到什么。展开 B 时，它后面还有 β，因此 B 的展望符来自 `FIRST(βa)`。

```text
对 B 的每条规则 B → γ：
    对 FIRST(βa) 中的每个终结符 b：
        把 [B → · γ, b] 加入闭包
```

若 β 能推出 ε，就把 a 也传给 B。β 完全为空时，只传 a。例如 `β = C`、`FIRST(C) = {c, ε}`、`a = $`，就要加入分别带 `c` 和 `$` 的两个项目。展望符增加后要继续展开，直到没有新项目。`goto(I,X)` 把点移过 X、保留原展望符，再取闭包。用队列发现状态时，规范 LR(1) 按**包括展望符在内的完整集合**判重。

在赋值文法中，从起点识别 `left` 后留下的两个项目各自有了展望符。

```text
[stmt  → left · EQUAL right, $]
[right → left ·, $]
```

所以遇 `EQUAL` 移进，遇 `$` 才按 `right → left` 归约。

## 手算一张 LR(1) 表（p.21–39）

下面沿用第 6 讲的四条规则，此处的 `term` 只生成 `INT`。

```text
r0: start → expr
r1: expr  → expr PLUS term
r2: expr  → term
r3: term  → INT
```

从 `[start → · expr, $]` 展开，两条 `expr` 规则先得到 `$`。再看 `expr → · expr PLUS term`，内层 `expr` 后面紧跟 `PLUS`，两条规则都要补上 `PLUS`。`expr → · term` 又把两个展望符传给 `term`。初态有 7 个单展望符项目，合写后只有 4 行。

```text
I₀:
    [start → · expr, $]
    [expr  → · expr PLUS term, {$, PLUS}]
    [expr  → · term, {$, PLUS}]
    [term  → · INT, {$, PLUS}]
```

```text
I₁ = goto(I₀, expr):
    [start → expr ·, $]
    [expr  → expr · PLUS term, {$, PLUS}]

I₂ = goto(I₀, term):
    [expr → term ·, {$, PLUS}]

I₃ = goto(I₀, INT):
    [term → INT ·, {$, PLUS}]

I₄ = goto(I₁, PLUS):
    [expr → expr PLUS · term, {$, PLUS}]
    [term → · INT, {$, PLUS}]

I₅ = goto(I₄, term):
    [expr → expr PLUS term ·, {$, PLUS}]
```

此外 `goto(I₄, INT) = I₃`。终结符边填 shift，非终结符边填 GOTO。普通完成项只在自己的展望符列填 reduce，增广完成项在 `$` 上填 accept。空格是错误。

| 状态 | ACTION: `INT` | ACTION: `PLUS` | ACTION: `$` | GOTO: `expr` | GOTO: `term` |
| --- | --- | --- | --- | --- | --- |
| `I₀` | `s3` | — | — | `1` | `2` |
| `I₁` | — | `s4` | accept | — | — |
| `I₂` | — | `r2` | `r2` | — | — |
| `I₃` | — | `r3` | `r3` | — | — |
| `I₄` | `s3` | — | — | — | `5` |
| `I₅` | — | `r1` | `r1` | — | — |

`INT INT` 是个很小的验算例。在 `I₃` 看见第二个 `INT` 就报错。对合法输入 `INT PLUS INT`，动作依次是 `s3, r3, r2, s4, s3, r3, r1, accept`。

期中复习题把文法换成 `E → E + M | M`、`M → M @ T | T`、`T → f(M) | t`。从 `[S → · E, $]` 算闭包时，E 项得到 `{$,+}`，M 项和 T 项多出 `@`。进入函数括号后，`T → f( · M )` 又会给里面的 M 带来 `)`。所以画图时必须把展望符写在状态里。输入 `f(t@t)@t+t` 的最外层是加法，它的左孩子是 `f(t@t)@t`。[期中复习课件，第 67–83 页](../lectures/LEC-17-Midterm-Review.pptx)

## LALR 合并时会丢掉什么（p.43–49）

从规范 LR(1) 状态中擦掉展望符，剩下的带点产生式集合称为 core。只合并 core 相同的状态，**按同一条带点产生式**对展望符取并集，出边连向对应的合并状态。只共享一两个项目的状态不能这样合并。

```text
start → s
s → a A p | b A q | a B q | b B p
A → x
B → x
```

读完 `a x` 和 `b x`，得到相同的 core。两种前缀原本要求相反的归约。读过 `a x` 时 p 对应 A，读过 `b x` 时 p 对应 B。

```text
I₁:                        I₂:
    [A → x ·, p]               [A → x ·, q]
    [B → x ·, q]               [B → x ·, p]
```

合并后两个完成项的展望符都成了 `{p,q}`，两列都发生 reduce/reduce 冲突。若原规范表无冲突，这样合并只可能新添 reduce/reduce 冲突。能否在某个终结符上 shift 已经由 core 决定，合并前每个成员状态都允许同样的 shift。

按给定文法能否直接建出无冲突表比较，依次有 `LR(0) ⊊ SLR(1) ⊊ LALR(1) ⊊ LR(1)`。规范 LR(1) 可能拆出更多状态。LR 本来就能先移进两个候选共用的前缀，例如遇到 `ID` 后再看 `(` 或 `=` 区分调用和赋值。区分这两个分支只需在适当的状态看一个 Token。[Bison 的 LR 构表方法](https://www.gnu.org/software/bison/manual/html_node/LR-Table-Construction.html)

## 生成器怎样处理冲突（p.50–62）

读冲突报告时，先找**状态、当前 Token 和竞争的两个项目**。例如在 `1 * 2` 后看见 `+`，移进会让加法先结合，归约才会先形成乘法。下面的 Bison 声明让乘法优先，并让同级运算向左结合。

```bison
%token INT PLUS STAR
%left PLUS
%left STAR

%%
expr:
    expr PLUS expr
  | expr STAR expr
  | INT
;
```

后声明的 `STAR` 优先级较高。若当前 Token 的优先级较高就移进，待归约规则较高就归约。同级时，`%left` 归约，`%right` 移进，`%nonassoc` 可使该情况报错。规则默认取右侧最后一个终结符的优先级，`%prec` 可覆盖。若一方无优先级，Bison 默认以 shift 解决该类冲突。[Bison 优先级规则](https://www.gnu.org/software/bison/manual/html_node/How-Precedence.html)

在 `if c1 if c2 s1 else s2` 上移进 `else`，就先配到内层 `if`。两个归约竞争时，要查是否有重复的推导路径，或是 LALR 合并抹去了差异。能证明**同一个完整输入有两棵树**才算证明文法有歧义。生成器在读取文法规约时报告冲突、生成表。生成的分析器在读取源码时执行移进、归约和相应的语义动作。[Bison 的 dangling else 示例](https://www.gnu.org/software/bison/manual/html_node/Shift_002fReduce.html)

## 编辑器怎样复用旧树（p.63–68）

只把函数 f 里的 `2` 改成 `20` 时，另一个函数 g 的子树通常可以直接复用。若当前状态能够接上旧子树代表的非终结符 A，且子树文本、边界和词法上下文仍然有效，解析器就可跳到子树末尾，省掉 A 内部的移进和归约。删掉引号却可能一路改变后文的分词，源码偏移相同不足以判断能否复用。

课件里的 Tree-sitter 先把 `grammar.js` 编成语言专用的分析器。运行时，分析器用修改范围、旧树和新源码更新具体语法树。即使用户还没输完，也可以在错误附近保留结构，供高亮、折叠和选区使用。它还可对声明的冲突保留多条分析路径。[Tree-sitter 文档](https://tree-sitter.github.io/tree-sitter/) · [子树复用实现](https://github.com/tree-sitter/tree-sitter/blob/master/lib/src/parser.c) · [文法中的冲突声明](https://tree-sitter.github.io/tree-sitter/creating-parsers/2-the-grammar-dsl.html)
