# LEC 4 · 语法分析、文法与歧义

SE3355 编译原理 · 2026 秋　[本讲课件（42 页）](../lectures/LEC-4-Parsing-I.pptx)

`return x + 1;` 和 `return + ; x 1` 里的每个词都能通过词法分析。要判断整句是否合法，还要知道 `return` 后面能接什么、加号左右各需要什么。语法分析器（Parser）按文法把 Token 组成树，树上每个非终结符节点都对应一次产生式展开。

## 上下文无关文法能描述什么（p.2–14）

上下文无关文法 `G = (N,T,P,S)` 包含非终结符 N、终结符 T、产生式 P、开始符号 S。终结符是输入中的 Token 类别。产生式 `A → γ` 说明非终结符 A 可以展开成 γ，无论 A 两侧是什么符号。例如 `stmt → RETURN expr SEMI` 要求返回语句中间有一个表达式。γ 可以为空（ε），此时不消耗 Token。

```text
expr → INT
     | expr PLUS expr
     | expr STAR expr
     | LPAREN expr RPAREN
```

`expr` 反复出现在右侧，有限条规则就能描述任意深的表达式。同样，`program → function*`、`block → LBRACE stmt* RBRACE` 可以表示多个函数和多条语句。这里的星号是 EBNF 简写，例如 `stmt_list → stmt stmt_list | ε`。

经典正则表达式无法识别任意深度的配对括号。若确定有限自动机（DFA）只有 n 个状态，读入 0 到 n 个左括号后，至少有两种深度会落到同一状态。此时接上刚好能闭合较浅前缀的右括号，两条路径会给出相同结论，实际却只有较浅的那串配平。用递归调用记住尚待闭合的左括号，就能用下面两条文法分别描述单纯嵌套和任意配对。

```text
nested   → LPAREN nested RPAREN | ε
balanced → LPAREN balanced RPAREN balanced | ε
```

`nested` 接受 `(())`，`balanced` 还接受 `()()`，因为它允许在一对括号之后再接一段。CFG 也有界限，例如同时约束三段长度的 `{aⁿbⁿcⁿ | n ≥ 1}` 超出了它的能力。源码中的声明和类型约束通常交给语义分析。

## 从推导读出一棵树（p.15–23）

每一步选一个非终结符，用某条产生式右侧替换它。文法的语言是从 S 经过零步或多步替换，能得到的**全终结符串**。下面分别用最左推导和最右推导构造 `1 + (2 * 3)` 对应的结构。输入本身是 `1 + 2 * 3`，括号只用来说明树形。

```text
expr
⇒ expr PLUS expr
⇒ INT PLUS expr
⇒ INT PLUS expr STAR expr
⇒ INT PLUS INT STAR expr
⇒ INT PLUS INT STAR INT
```

```text
expr
⇒ expr PLUS expr
⇒ expr PLUS expr STAR expr
⇒ expr PLUS expr STAR INT
⇒ expr PLUS INT STAR INT
⇒ INT PLUS INT STAR INT
```

两次都是先把根展开成加法，乘法都在右子树里。最左、最右规定的是**先展开哪个非终结符**。完整语法分析树保留文法节点和终结符，从左到右读叶子就能还原输入。若用 ε 叶子画空产生式，还原时跳过它。

```text
expr
├── expr
│   └── INT(1)
├── PLUS
└── expr
    ├── expr
    │   └── INT(2)
    ├── STAR
    └── expr
        └── INT(3)
```

后续计算只需要 `Add(Int(1), Mul(Int(2), Int(3)))` 这样的抽象语法树（AST）。括号的作用已经体现在树形里。

## 歧义怎样改变结果（p.23–34）

若同一个终结符串能生成两棵不同的语法分析树，文法就有歧义。也可以给出**两条不同的最左推导**证明它。上面的文法允许第一步改用 `expr → expr STAR expr`，再在乘法的左子树里展开加法。下面是这棵树的最左推导。

```text
expr
⇒ expr STAR expr
⇒ expr PLUS expr STAR expr
⇒ INT PLUS expr STAR expr
⇒ INT PLUS INT STAR expr
⇒ INT PLUS INT STAR INT
```

这样 `1 + 2 * 3` 就既能算出 `7`，也能算出 `9`。这是**优先级**问题。即使只有减号，`(10 - 3) - 2 = 5` 和 `10 - (3 - 2) = 9` 也会给出不同结果。这是**结合性**问题。

把加减、乘法和原子分成三层，可以同时规定两者。

```text
expr   → expr PLUS term
       | expr MINUS term
       | term

term   → term STAR factor
       | factor

factor → LPAREN expr RPAREN
       | INT
```

`term` 不能生成裸露的 `+`，因此 `1 + 2 * 3` 只能把 `2 * 3` 作为一个 `term` 放到加法右侧。在 `expr → expr MINUS term` 中，减号的右操作数必须是 `term`，只有左操作数 `expr` 能直接再展开一个减号。所以 `10 - 3 - 2` 只能解析成 `(10 - 3) - 2`。显式括号允许 `factor` 回到 `expr`。若幂运算要求右结合，可写 `power → atom POW power | atom`。

## 一个 else 属于哪个 if（p.35–40）

给 `stmt` 同时加上“有 else”和“无 else”两条规则，`if (c1) if (c2) s1; else s2;` 就有两种读法。常见约定让 `else` 配最近的、还没有 `else` 的 `if`。当 `c1` 为假时，按此约定连 `s2` 也不执行。

```text
stmt      → matched | unmatched
cond      → LPAREN expr RPAREN

matched   → OTHER
          | IF cond matched ELSE matched

unmatched → IF cond stmt
          | IF cond matched ELSE unmatched
```

`matched` 表示里面的 `if` 都已配好。想把 `else` 配给外层，外层的 then 分支就必须是 `matched`。这里内层 `if (c2) s1;` 还有空位，只能先配给它。`unmatched` 本身是合法语句，允许有一个或多个 `if` 省略 else。

也可以保留短文法，再让生成器用优先级和结合性解决冲突。例如课件先声明 `%left PLUS`，再声明 `%left STAR`，较晚声明的乘法优先级更高。这样选定了 Parser 的行为，原文法仍可推导出两棵树。[LR 冲突与优先级声明](LEC-7-Constructing-LR-Parsing-Gen.md)

## 自测

1. 证明文法有歧义，需要找两个不同的输入吗？只需找**一个输入**，给它画出两棵树，或者给它写出两条不同的最右推导。
2. 为什么 `(1 + 2) * 3` 在分层文法里仍合法？乘法的左操作数可由 `factor → LPAREN expr RPAREN` 生成，括号里的 `expr` 可以生成加法。
3. 文法 `S → ( S ) S | ε` 如何读 `()()`？第一次展开后，第一个 S 取 ε，第二个 S 再展开一对括号，最后两个 S 都取 ε。何时取 ε 可以查预测表。[第 5 讲 · LL(1) 分析](LEC-5-Top-Down-Parsing.md)
