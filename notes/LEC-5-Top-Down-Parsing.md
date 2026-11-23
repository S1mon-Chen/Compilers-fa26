# LEC 5 · 自顶向下分析与 LL(1)

SE3355 编译原理 · 2026 秋　[本讲课件（66 页）](../lectures/LEC-5-Parsing-II.pptx)

递归下降分析器把每个非终结符写成一个函数。以 `print(1 + 2 * 3);` 为例，`parseStmt` 负责认出 `print(…);`，括号里的表达式交给 `parseExpr`。难点在于函数有多个候选时，怎样只看下一个 Token 就选中一条。

## 从文法到函数（p.6–26）

```text
program   → stmt

stmt      → IF LPAREN expr RPAREN stmt
          | PRINT LPAREN expr RPAREN SEMI

expr      → term expr_tail
expr_tail → PLUS term expr_tail | ε

term      → factor term_tail
term_tail → STAR factor term_tail | ε

factor    → INT | LPAREN expr RPAREN
```

终结符对应一次 `eat`（检查类别，匹配后前进），非终结符对应函数调用，右侧序列从左到右执行。这样得到最左推导。若依次试每条产生式，失败时必须恢复输入和临时节点。`s → a Z, a → X | X Y` 解析 `X Y Z` 时，`a → X` 在局部已经成功，外层却会在 `Y` 处失败。完整回溯要能回到 `a` 改选 `X Y`。

**左递归。** `expr → expr PLUS term | term` 若直接译成递归调用，`parseExpr()` 会在读第一个 Token 前再次调用自己。把开头不递归的分支留下，再把重复部分放进一个新的尾部非终结符。

```text
原文法
A  → A α₁ | … | A αₙ | β₁ | … | βₘ

改写后
A  → β₁ A' | … | βₘ A'
A' → α₁ A' | … | αₙ A' | ε
```

每个 β 都是非左递归的起始分支，重复部分 α 应能推进解析。改写的是直接左递归，若 `A ⇒ B ⇒ A …`，还要先处理间接递归。用 BNF 要为重复的尾部单写产生式。扩展巴科斯范式（EBNF）可以直接写 `expr → term (PLUS term)*`，可选成分写作 `X?`。这两种记法描述同一套输入。

改写后仍可让减法左结合。下面的 `parseExpr` 每读到一个减号，就把此前的结果作为新节点的左孩子。

```text
parseExpr():
    left = parseTerm()
    while current.kind == MINUS:
        eat(MINUS)
        right = parseTerm()
        left = node(MINUS, left, right)
    return left
```

`10 - 3 - 2` 先建 `Sub(10,3)`，再建 `Sub(Sub(10,3),2)`，结果为 5。尾部规则朝右递归，并不要求 AST 也朝右长。

## 用 FIRST、FOLLOW 选择产生式（p.28–43）

| 集合 | 回答的问题 | 计算方法 |
| --- | --- | --- |
| `FIRST(α)` | α 能以什么终结符开头？能否整个消失？ | 从左向右并入各符号 FIRST 中除 ε 以外的元素。遇到不能推出 ε 的符号就停。若整个右侧可空，再加入 ε。 |
| `FOLLOW(A)` | 在完整推导中，A 后面可能紧跟什么？ | 先把结束标记 `$` 放入开始符号的 FOLLOW。对 `X → α A β`，加入 `FIRST(β) − {ε}`。若 β 可空，再并入 `FOLLOW(X)`。 |
| `PREDICT(A → α)` | 当前 Token 在哪些情况下可以选这一条？ | 取 `FIRST(α) − {ε}`。若 α 可空，再并入 `FOLLOW(A)`。 |

终结符 `t` 的 FIRST 为 `{t}`，空串的 FIRST 为 `{ε}`。先算 FIRST 到没有新增元素，再算 FOLLOW 到稳定。FOLLOW 和 PREDICT 中都没有 ε。对于同一个左侧 A，各候选的 PREDICT 两两不交，分析器才能靠一个向前看的 Token（lookahead）唯一选择。LL(1) 的两个 L 分别指从左到右读输入、构造最左推导，1 指预测时看一个 Token。

下面的文法允许省略访问修饰符 a 和静态修饰符 b。

```text
s → a b
a → PUBLIC | PRIVATE | ε
b → STATIC | ε
```

| 非终结符 | FIRST | FOLLOW |
| --- | --- | --- |
| `s` | `{PUBLIC, PRIVATE, STATIC, ε}` | `{$}` |
| `a` | `{PUBLIC, PRIVATE, ε}` | `{STATIC, $}` |
| `b` | `{STATIC, ε}` | `{$}` |

为什么 `STATIC` 同时出现在 `FIRST(s)` 和 `FOLLOW(a)`？`a` 可以为空，所以 `s` 能用 `b` 的 `STATIC` 开头。另一方面，`b` 正好跟在 `a` 后面，所以这个词也能紧跟 `a`。`b` 又可空，因此结束标记也能紧跟 `a`。

| 产生式 | PREDICT |
| --- | --- |
| `s → a b` | `{PUBLIC, PRIVATE, STATIC, $}` |
| `a → PUBLIC` | `{PUBLIC}` |
| `a → PRIVATE` | `{PRIVATE}` |
| `a → ε` | `{STATIC, $}` |
| `b → STATIC` | `{STATIC}` |
| `b → ε` | `{$}` |

输入只有 `STATIC` 时，`a` 选 ε，保留当前 Token 给 `b`。`PREDICT(s → a b)` 也包含 `$`，因为右侧两个符号都可以消失。若两个可空候选都要填入同一个 FOLLOW 列，就出现冲突。

**为什么要迭代？** 对 `s → a, a → b c, b → ID | ε, c → NUM | ε`，若每轮只读上一轮的集合，先算出 b、c 的 FIRST，第二轮才得到 `FIRST(a) = {ID, NUM, ε}`，第三轮才传到 s。FOLLOW 的传播方向不同，`$` 从 s 传给 a，再传给 b、c。由于 c 可以从 `NUM` 开始，`NUM` 也进了 `FOLLOW(b)`。漏做最后一轮会把合法的空输入判错。

## 填表，再用表解析表达式（p.44–53）

下面把开头文法中的 `expr` 单独作为开始符号。因此 `FOLLOW(expr)` 同时含括号里的 `RPAREN` 和独立表达式末尾的 `$`。在开头以 `program` 为起点的完整文法里，`expr` 总是后接 `RPAREN`。

| 非终结符 | FIRST | FOLLOW |
| --- | --- | --- |
| `expr` | `{INT, LPAREN}` | `{RPAREN, $}` |
| `expr_tail` | `{PLUS, ε}` | `{RPAREN, $}` |
| `term` | `{INT, LPAREN}` | `{PLUS, RPAREN, $}` |
| `term_tail` | `{STAR, ε}` | `{PLUS, RPAREN, $}` |
| `factor` | `{INT, LPAREN}` | `{STAR, PLUS, RPAREN, $}` |

对每条 `A → α`，把 α 填进 `M[A,t]` 的每个 `t ∈ PREDICT(A → α)` 列。空格表示语法错误。

| 非终结符 | INT | LPAREN | PLUS | STAR | RPAREN | $ |
| --- | --- | --- | --- | --- | --- | --- |
| `expr` | `term expr_tail` | `term expr_tail` | — | — | — | — |
| `expr_tail` | — | — | `PLUS term expr_tail` | — | `ε` | `ε` |
| `term` | `factor term_tail` | `factor term_tail` | — | — | — | — |
| `term_tail` | — | — | `ε` | `STAR factor term_tail` | `ε` | `ε` |
| `factor` | `INT` | `LPAREN expr RPAREN` | — | — | — | — |

栈保存**还要匹配的符号**。下表左端为栈顶。栈顶是终结符时，匹配并读入一个 Token。遇到非终结符就查表，弹出它并把右侧逆序压栈。选 ε 时只弹栈，不读输入。两边同时到 `$` 才接受。

| 步骤 | 栈（左侧为栈顶） | 剩余输入 | 动作 |
| --- | --- | --- | --- |
| 1 | `expr $` | `INT PLUS INT STAR INT $` | `expr → term expr_tail` |
| 2 | `term expr_tail $` | `INT PLUS INT STAR INT $` | `term → factor term_tail` |
| 3 | `factor term_tail expr_tail $` | `INT PLUS INT STAR INT $` | `factor → INT` |
| 4 | `INT term_tail expr_tail $` | `INT PLUS INT STAR INT $` | 匹配 `INT(1)` |
| 5 | `term_tail expr_tail $` | `PLUS INT STAR INT $` | `term_tail → ε` |
| 6 | `expr_tail $` | `PLUS INT STAR INT $` | `expr_tail → PLUS term expr_tail` |
| 7 | `PLUS term expr_tail $` | `PLUS INT STAR INT $` | 匹配 `PLUS` |
| 8 | `term expr_tail $` | `INT STAR INT $` | `term → factor term_tail` |
| 9 | `factor term_tail expr_tail $` | `INT STAR INT $` | `factor → INT` |
| 10 | `INT term_tail expr_tail $` | `INT STAR INT $` | 匹配 `INT(2)` |
| 11 | `term_tail expr_tail $` | `STAR INT $` | `term_tail → STAR factor term_tail` |
| 12 | `STAR factor term_tail expr_tail $` | `STAR INT $` | 匹配 `STAR` |
| 13 | `factor term_tail expr_tail $` | `INT $` | `factor → INT` |
| 14 | `INT term_tail expr_tail $` | `INT $` | 匹配 `INT(3)` |
| 15 | `term_tail expr_tail $` | `$` | `term_tail → ε` |
| 16 | `expr_tail $` | `$` | `expr_tail → ε` |
| 17 | `$` | `$` | 接受 |

注意第 5 步和第 11 步。第一个整数之后是加号，`term_tail` 结束，把加号留给外层。第二个整数之后是乘号，仍由 `term_tail` 接着处理。最后的 AST 是 `Add(1, Mul(2,3))`。预测式递归下降使用同一份选择集合，只是由函数调用栈保存进度。

## 公共前缀、报错与生成器（p.54–65）

`stmt → IDENT ASSIGN expr SEMI | IDENT LPAREN args RPAREN SEMI` 的两条规则在 `IDENT` 列冲突。提取公共前缀后，先读名字。接下来看到 `=` 走赋值分支，看到 `(` 走调用分支。

```text
stmt      → IDENT stmt_tail
stmt_tail → ASSIGN expr SEMI
          | LPAREN args RPAREN SEMI
```

只提取文本上相同的前缀并不保证 LL(1)。例如 `s → a | b, a → ID, b → ID` 的候选虽然写法不同，PREDICT 仍有交集。每次改写后要重新算集合。

栈顶终结符与输入不符，或非终结符的表格为空，分析器就会报错。恢复时可以尝试局部修复，也可以跳到语句边界。

| 策略 | 对输入和状态的操作 | 课件例子 |
| --- | --- | --- |
| 插入 | 补一个期望的 Token，保留当前输入 | 在 `SEMI` 前补 `RPAREN` |
| 删除 | 跳过这个意外 Token，保留栈顶，再试一次 | `1 + * 2` 中删掉 `STAR` |
| 同步 | 跳过输入并调整解析状态，回到可信语法边界 | 跳到 `SEMI`、`RBRACE` 或 `$` |

同步到 `}` 后，应同时退回能处理右花括号的栈层，或按上下文消费它。原样重试会在同一个位置再次报错。提取当前表行的非空列，还能给出“此处期待哪些 Token”的诊断。

课件用 ANTLR4 说明生成器能承担哪些工作。对支持的直接左递归表达式规则，它能把非递归分支当起点，再按优先级处理后续运算。课件示例把 `STAR` 分支写在 `PLUS` 之前，得到乘法优先。Adaptive LL(*) 遇到公共前缀时可以继续观察输入，在能区分候选时作出选择。LR 分析可以先读入符号，等右侧出现后再归约。[第 6 讲 · LR 分析](LEC-6-Intro2LR-Parsing.md)
