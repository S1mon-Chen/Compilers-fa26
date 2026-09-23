# LEC 4 · Introduction to Parsing

> SE3355 编译原理（2026 秋）  
> 依据：[LEC-4-Intro-to-Parsing.pptx](../lectures/LEC-4-Intro-to-Parsing.pptx)，共 42 页。以下页码指课件页码。  
> 本笔记按主题整理；标注“补充理解”的部分包含解释、推导或形式化细节，不是课件原文。  
> 前置：[LEC 3 · Lexical Analysis: Implementation of R.E](LEC-3-Lexical-Analysis-Impl-of-RE.md)。

## 1. Parser 的任务：从 Token 序列恢复程序结构（p.2–6）

Lexer 已经识别了每个词法单元，但 Token 的排列是否构成合法程序、程序内部怎样嵌套，还需要 Parser（语法分析器）判断。

```text
源程序字符流
    ↓ Lexer：识别词法单元
Token Stream
    ↓ Parser：依据文法识别组合结构
Parse Tree（语法分析树）
    ↓
后续语义分析等阶段
```

例如，下列两组输入中的单个 Token 都可以合法，但只有第一组能组成课件中的 return 语句：

| 源码 | Token 序列 | 整体结构 |
| --- | --- | --- |
| `return x + 1;` | `RETURN IDENT PLUS INT SEMI` | 合法的 return 语句 |
| `return + ; x 1` | `RETURN PLUS SEMI IDENT INT` | 无法按相应规则组成语句 |

对于 `return x + 2 * 3;`，还需要知道：

```text
2 * 3                 → 较小的表达式
x + (2 * 3)           → 更大的表达式
return [expression];  → 完整语句
```

因此，语法分析要解决两个问题：

1. **合法性：** 输入 Token 序列是否属于文法描述的语言？
2. **结构：** 这些 Token 通过哪些规则组成表达式、语句等构造？

**补充理解：** Token 通常还带有词素、数值和 source span（源码位置范围），不只是类别。课件强调的是：线性的 Token 流尚未明确表示程序的层次关系。

## 2. 上下文无关文法 CFG（p.7–12）

### 2.1 用有限规则描述递归结构

Context-Free Grammar（CFG，上下文无关文法）通过产生式描述一个结构可以由哪些部分组成。例如：

```text
expr → INT
     | expr PLUS expr
     | expr STAR expr
     | LPAREN expr RPAREN
```

- `→` 左边是要展开的非终结符，右边是替换它的符号序列。
- `|` 表示不同的候选产生式。
- 右侧再次出现 `expr`，使表达式可以递归嵌套。
- 文法规定合法结构，本身没有指定寻找这些结构的解析算法。

**规则数量有限，不意味着生成的程序长度或嵌套深度有固定上限。**

### 2.2 CFG 的四类元素

| 元素 | 含义 | 例子 |
| --- | --- | --- |
| 终结符 T（terminals） | 最终输入序列中的符号，不能再用产生式展开 | `INT`、`PLUS`、`LPAREN` |
| 非终结符 N（nonterminals） | 对语言结构的命名，可以用产生式展开 | `expr`、`stmt`、`program` |
| 开始符号 S（start symbol） | 推导整份输入时的起点，属于 N | `program`，或单独分析表达式时的 `expr` |
| 产生式 P（productions） | 非终结符允许采用的展开形式 | `expr → expr PLUS expr` |

课件约定：终结符使用大写名称，非终结符使用小写名称。这里统一使用 `expr`。

**补充理解：** 常将 CFG 写成四元组：

$$
G=(N,T,P,S),\qquad N\cap T=\varnothing
$$

每条产生式具有形式：

$$
A\to\gamma,\qquad A\in N,\quad\gamma\in(N\cup T)^*
$$

右侧可以是空串 `ε`。这里的 `*` 表示零个或多个符号组成的有限序列。作为空产生式时，`ε` 不代表需要从输入中读取一个 Token。

### 2.3 “上下文无关”中的上下文是什么

这里的上下文指待替换符号左右的符号，不是程序的运行时状态。

```text
上下文无关：B → b
出现 B 就可以应用该产生式，无论左右是什么。

带上下文条件：a B c → a b c
只有 B 位于 a 与 c 之间时，才允许作这一替换。
```

课件用一般形式 `α A β → α γ β` 展示上下文条件，并给出语言：

$$
L=\{a^n b^n c^n\mid n\ge1\}
$$

三段长度需要同步变化；该语言不是上下文无关语言，但可由上下文有关文法描述。

工程上通常用 CFG 描述程序结构，把声明、类型等约束留给语义分析。

**补充理解：** 一个表达式在语法上能被识别，不代表其中的名字已经声明、操作数类型正确，或程序执行结果正确。

### 2.4 从表达式扩展到整个程序

课件的示意规约如下，省略了参数等细节：

```text
program  → function*
function → TYPE ID LPAREN params RPAREN block
block    → LBRACE stmt* RBRACE
stmt     → RETURN expr SEMI
         | IF LPAREN expr RPAREN stmt
         | block
expr     → ID | INT | expr PLUS expr
…
```

表达式内部可以嵌套表达式，语句内部可以嵌套语句，代码块也可以继续包含代码块。

**补充理解：** 此处 `function*`、`stmt*` 是 EBNF 风格的“重复零次或多次”简写，并非普通 CFG 产生式中的字面星号。例如可以展开为：

```text
program   → function program | ε
block     → LBRACE stmt_list RBRACE
stmt_list → stmt stmt_list | ε
```

## 3. 为什么正则表达式不够（p.13–14）

任意深度的括号嵌套要求记住尚未匹配的左括号：

```text
()
(())
((()))
…
```

有限自动机只有有限个状态，无法记录无上限的嵌套深度，因此经典正则表达式无法描述这种任意深度的配对关系。CFG 可以借助递归描述它。

**补充理解：** 下面两份文法描述的语言不同：

```text
nested   → LPAREN nested RPAREN | ε
balanced → LPAREN balanced RPAREN balanced | ε
```

- `nested` 描述单层套单层的形式，包括 `ε`、`()`、`(())`，但不包括 `()()`。
- `balanced` 同时允许嵌套和并列，能够描述 `()()`、`(()())` 等所有正确配对的单种括号串。

这里“不够用”针对的是**无界嵌套**。若深度存在固定上限，则可以用有限状态记录。此处的正则表达式也指形式语言理论中的经典正则，不涉及某些正则引擎提供的递归等扩展。

## 4. 推导与文法的语言（p.15–18）

### 4.1 一次推导做什么

每一步从当前符号串中选一个非终结符，再选择它的一条产生式，将它替换为右侧：

```text
expr
⇒ expr PLUS expr
⇒ INT PLUS expr
⇒ INT PLUS INT
```

每一步有两个独立的选择：

1. 展开哪一个非终结符？
2. 对这个非终结符应用哪一条产生式？

**补充理解：** 若存在产生式 `A → γ`，则可以作一步推导：

$$
\alpha A\beta\Rightarrow\alpha\gamma\beta
$$

`→` 用于写产生式，`⇒` 表示对当前串应用一次产生式，`⇒*` 表示应用零次或多次。

### 4.2 文法描述的语言

$$
L(G)=\{w\in T^*\mid S\Rightarrow^*w\}
$$

也就是：从开始符号出发，经过有限步推导，能够得到的所有**只含终结符**的序列。

**补充理解：** 推导过程中出现的 `INT PLUS expr` 仍含非终结符，称为句型（sentential form）；它还不是最终输入句子。

### 4.3 完整例子：INT PLUS INT STAR INT

使用第 2.1 节的表达式文法，可以作如下推导：

```text
expr
⇒ expr PLUS expr
⇒ INT PLUS expr
⇒ INT PLUS expr STAR expr
⇒ INT PLUS INT STAR expr
⇒ INT PLUS INT STAR INT
```

该推导选择把整个表达式拆成“左边一个整数，加上右边一个乘法表达式”。

注意：这里展示了一个允许的结构；当前文法尚未保证它是唯一的结构。

## 5. 推导怎样形成 Parse Tree（p.17、21–23）

应用 `A → X₁ X₂ … Xₙ` 时，以 A 为父节点，按顺序建立 `X₁` 到 `Xₙ` 这些子节点。

上一节推导对应的完整语法分析树为：

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

- 根节点是开始符号。
- 非终结符节点的子节点必须对应它的一条产生式右侧。
- 从左到右读取终结符叶子，得到原始 Token 序列。
- 父子关系保留组合与嵌套关系，Token 及其 source span 可支持后续诊断。

上面的 `INT(1)` 表示类别为 `INT`、值为 `1` 的 Token。文法匹配的是 `INT` 类别。

**补充理解：Parse Tree 与 AST 的区别。** 课件也用下面这种简化运算树突出结构：

```text
    +
   / \
  1   *
     / \
    2   3
```

完整 Parse Tree 保留文法展开层次及标点等终结符；AST（Abstract Syntax Tree，抽象语法树）通常省略不必要的文法中间层和标点。上面的简图可视为 AST 风格的示意，不能据此认为完整 Parse Tree 的所有内部节点都是运算符。

若存在 `A → ε`，画树时可以用 `ε` 叶子表示空产生式，但它不属于实际输入 Token 序列。

## 6. 最左推导与最右推导（p.19–21、29）

### 6.1 两种固定的展开顺序

| 推导方式 | 每一步选择哪个非终结符 | 仍需决定什么 |
| --- | --- | --- |
| 最左推导（leftmost derivation） | 当前句型中最左侧的非终结符 | 使用它的哪条产生式 |
| 最右推导（rightmost derivation） | 当前句型中最右侧的非终结符 | 使用它的哪条产生式 |

课件中的“左推导”和“右推导”分别指最左推导和最右推导。

### 6.2 同一棵树的两种推导

对 `1 + (2 * 3)` 这一组合结构，最左推导是：

```text
expr
⇒ expr PLUS expr
⇒ INT PLUS expr
⇒ INT PLUS expr STAR expr
⇒ INT PLUS INT STAR expr
⇒ INT PLUS INT STAR INT
```

最右推导是：

```text
expr
⇒ expr PLUS expr
⇒ expr PLUS expr STAR expr
⇒ expr PLUS expr STAR INT
⇒ expr PLUS INT STAR INT
⇒ INT PLUS INT STAR INT
```

两者建立分支的先后不同，但根节点的展开方式和各分支的结构相同，因此对应同一棵 Parse Tree。

**关键区别：展开顺序与结合方向是两回事。** 最左推导不能自动规定减法左结合，也不能规定乘法优先于加法。它只消除了“下一步展开哪个非终结符”的自由，仍需选择产生式。

**补充理解：** 对一棵固定的 Parse Tree，可以确定其对应的最左推导和最右推导。因此，仅展示两条任意顺序的不同推导，不足以证明文法有歧义。

## 7. 歧义：同一输入对应不同的树（p.23–29）

### 7.1 正式定义

若存在某个 `w ∈ L(G)`，它拥有两棵或更多不同的 Parse Tree，则文法 G 是**歧义文法（ambiguous grammar）**。

等价的判据是：

- 同一个 w 存在两个不同的最左推导。
- 同一个 w 存在两个不同的最右推导。

只需要找到**一个反例字符串**即可证明歧义，不需要所有输入都出现歧义。歧义按树的结构定义，并不要求不同的树一定算出不同的值。

### 7.2 优先级歧义

考虑朴素文法：

```text
expr → expr PLUS expr
     | expr MINUS expr
     | expr STAR expr
     | LPAREN expr RPAREN
     | INT
```

`1 + 2 * 3` 可以有两种解释：

| 结构 | 根运算 | 结果 |
| --- | --- | --- |
| `1 + (2 * 3)` | `+` | 7 |
| `(1 + 2) * 3` | `*` | 9 |

第 6.2 节已经给出了第一种结构的最左推导。第二种结构也有一条最左推导：

```text
expr
⇒ expr STAR expr
⇒ expr PLUS expr STAR expr
⇒ INT PLUS expr STAR expr
⇒ INT PLUS INT STAR expr
⇒ INT PLUS INT STAR INT
```

两条最左推导得到完全相同的终结符序列，却在根处选择了不同产生式，因此构成歧义的直接证据。

这里写出的括号只是用来说明分组，**并不表示原始输入多出了 LPAREN 和 RPAREN Token**。

### 7.3 结合性歧义

`10 - 3 - 2` 也可以有两种结构：

```text
(10 - 3) - 2 = 5
10 - (3 - 2) = 9
```

- **优先级（precedence）：** 不同层级的运算符混用时如何分组，例如 `*` 比 `+` 优先。
- **结合性（associativity）：** 同优先级运算符连续出现时如何分组，例如减法左结合。

这些选择需要明确写进文法或解析策略，不会因编译器“知道数学常识”而自动成立。

## 8. 用文法层次消除表达式歧义（p.32–34）

### 8.1 每个优先级对应一个层次

课件将表达式文法改写为：

```text
expr   → expr PLUS term
       | expr MINUS term
       | term

term   → term STAR factor
       | factor

factor → LPAREN expr RPAREN
       | INT
```

| 非终结符 | 负责的结构 | 作用 |
| --- | --- | --- |
| `expr` | 加法、减法 | 组合完整的 term |
| `term` | 乘法 | 组合完整的 factor |
| `factor` | 整数、括号表达式 | 提供原子或显式分组 |

乘法必须先在 `term` 层组成结构，再作为整体参与 `expr` 层的加减。括号通过 `factor → LPAREN expr RPAREN` 重新引入完整表达式，使 `(1 + 2) * 3` 仍可被合法解析。

### 8.2 为什么 1 + 2 * 3 只剩一种结构

**补充推导：** 新文法中的最左推导为：

```text
expr
⇒ expr PLUS term
⇒ term PLUS term
⇒ factor PLUS term
⇒ INT PLUS term
⇒ INT PLUS term STAR factor
⇒ INT PLUS factor STAR factor
⇒ INT PLUS INT STAR factor
⇒ INT PLUS INT STAR INT
```

若试图把整体解释成 `(1 + 2) * 3`，则乘法左侧需要在没有括号时由 `term` 生成 `INT PLUS INT`。然而 `term` 这一层没有引入裸露 `PLUS` 的规则，因此这种解释被排除。

**补充理解：** “高优先级更靠近叶子”是在说明这种分层文法的组织方式。不要把它机械理解为任意语法树中节点越深，运算符优先级必然越高；括号内可以重新出现低优先级运算。

### 8.3 用递归方向表达结合性

左结合减法：

```text
expr → expr MINUS term | term

10 - 3 - 2  →  (10 - 3) - 2
```

右侧只允许 `term`，连续减法只能继续嵌套到左侧的 `expr`，因此形成左结合结构。

右结合幂运算：

```text
power → atom POW power | atom

2 ** 3 ** 2  →  2 ** (3 ** 2)
```

连续幂运算继续嵌套在右侧的 `power`，因此形成右结合结构。

**补充理解：** 上述结论依赖于对另一个操作数层级的限制。“文法出现左递归”本身不保证无歧义，例如 `expr → expr MINUS expr | INT` 仍同时允许两种结合方式。结合性规定分组结构，也不能直接等同于带副作用表达式的操作数求值顺序。

## 9. Dangling else：else 应属于哪个 if（p.35–39）

### 9.1 朴素文法的歧义

```text
stmt → IF cond stmt
     | IF cond stmt ELSE stmt
     | OTHER

cond → LPAREN expr RPAREN
```

考虑下面的程序：

```c
if (c1)
    if (c2)
        s1;
    else
        s2;
```

为突出结构，用 `C1`、`C2` 代表条件，用 `S1`、`S2` 代表 OTHER 语句。相同输入有两种分组：

```text
IF C1 [IF C2 S1] ELSE S2     # else 属于外层 if
IF C1 [IF C2 S1 ELSE S2]     # else 属于内层 if
```

**补充理解：** 在 C 风格语法中，缩进本身并未提供文法约束。若 `c1` 为假，第一种结构会执行 `S2`，第二种结构不会，因此差别会影响控制流。

### 9.2 消歧约定：最近的未匹配 if

课件采用通常的 **nearest unmatched if** 规则：`else` 匹配最近的、尚未拥有 `else` 的 `if`。

因此上例应解释为：

```c
if (c1) {
    if (c2) s1; else s2;
}
```

确定期望的语言行为之后，还需要让文法或 Parser 实现这个选择。

### 9.3 把语句拆成 matched 与 unmatched

```text
stmt      → matched | unmatched
cond      → LPAREN expr RPAREN

matched   → OTHER
          | IF cond matched ELSE matched

unmatched → IF cond stmt
          | IF cond matched ELSE unmatched
```

| 类别 | 含义 |
| --- | --- |
| `matched` | 没有悬空 if 的完整语句，包括 OTHER，以及两个分支也都完整的 if–else |
| `unmatched` | 仍存在等待 else 的 if，可能在外层，也可能位于内部的 else 分支 |

`unmatched → IF cond matched ELSE unmatched` 表示：当前外层 if 已经有 else，但它的 else 分支里仍有悬空 if。

**补充理解：** `unmatched` 不表示“恰好只有一个未匹配 if”。`IF C1 IF C2 S1` 就有两个未匹配 if。它也不表示语法错误，因为语言允许没有 else 的 if 语句。

### 9.4 为什么 else 只能匹配内层 if

对输入 `IF C1 IF C2 S1 ELSE S2`，若让外层 if 拥有 else，则必须使用以下形式之一：

```text
matched   → IF cond matched ELSE matched
unmatched → IF cond matched ELSE unmatched
```

两者都要求 then 分支 `IF C2 S1` 是 `matched`，但它没有 else，只能属于 `unmatched`，因此外层匹配失败。

让 else 匹配内层时可以成功推导。下面沿用条件和 OTHER 的缩写：

```text
stmt
⇒ unmatched
⇒ IF C1 stmt
⇒ IF C1 matched
⇒ IF C1 IF C2 matched ELSE matched
⇒ IF C1 IF C2 S1 ELSE matched
⇒ IF C1 IF C2 S1 ELSE S2
```

核心约束是：**要把一个 else 分配给当前 if，它之前的 then 分支必须已经没有悬空 if。** 这样就不能跳过更近的未匹配 if。

这里解决的是 dangling else 歧义；条件表达式等子文法也需要各自确定唯一结构。

## 10. 工程选择：文法改写或额外消歧策略（p.40）

除了改写文法层次，Parser generator 也可以保留简洁的表达式文法，通过额外声明选择解析结构。课件使用如下示意：

```text
expr → expr PLUS expr
     | expr STAR expr
     | INT

%left PLUS
%left STAR
```

在课件展示的声明约定中：

- `%left` 指定左结合。
- 后声明的运算符优先级更高，因此 `STAR` 高于 `PLUS`。

| 方式 | 选择写在哪里 | 特点 |
| --- | --- | --- |
| 改写 CFG | 非终结符层次、产生式和递归方向 | 文法本身表达期望结构，但可能更长 |
| 额外声明 | 工具的优先级、结合性等策略 | 保留简洁文法，由工具依据策略选择结构 |

**补充理解：** 为歧义 CFG 添加消歧声明，不等于原来的 CFG 在形式上变成了无歧义文法。得到的是“文法 + 策略”共同确定的解析行为。比较两种方案时，除了检查接受哪些字符串，还要检查它们是否为同一输入选择相同的树。

## 11. 本讲与下一讲的边界（p.30–31、41–42）

课件 p.30–31 是词法分析与本讲前半部分的回顾，可以串起下面的关系：

```text
Regex + 词法规则 → 自动机与扫描循环 → Token Stream
CFG + 消歧约定 → 合法结构的规约
Token Stream + 结构规约 + 解析算法 → Parse Tree
```

本讲回答了“结构如何描述、如何通过推导形成、为什么会不唯一”，尚未给出从实际输入高效寻找语法树的完整算法。

下一讲将从 **top-down parsing（自顶向下语法分析）** 开始。需要继续解决的问题是：面对当前输入，Parser 应该为非终结符选择哪条产生式？

## 12. 易错点与复习自测

| 易错点 | 正确理解 |
| --- | --- |
| 每个 Token 合法，程序就合法 | Token 的顺序和组合还必须满足语法规则 |
| CFG 同时保证变量声明和类型正确 | CFG 通常描述结构，相关上下文约束由语义分析处理 |
| 有限条规则只能描述有限深度 | 递归允许无固定上限的嵌套 |
| 规则右侧写在前面的候选天然优先 | 普通 CFG 的候选顺序本身不提供消歧策略 |
| 有两条不同推导就一定有歧义 | 它们可能只是同一棵树的不同展开顺序 |
| 最左推导等于左结合 | 前者规定展开顺序，后者规定分组结构 |
| 根运算符优先级最高 | 对未被括号改变的分层表达式，根通常是较低优先级运算 |
| 有左递归就能保证左结合且无歧义 | 还需限制操作数层级，避免两侧任意递归造成多种树 |
| unmatched 是不合法或只有一个悬空 if | 它是合法语句类别，可以含多个未匹配 if |
| Parser 输出唯一结果就证明 CFG 无歧义 | Parser 可能借助文法之外的消歧策略选出一个结果 |

1. **CFG 的四个组成部分是什么？**  
   终结符 T、非终结符 N、产生式 P，以及属于 N 的开始符号 S。

2. **为什么 `INT PLUS expr` 还不是 L(G) 中的最终句子？**  
   它还包含非终结符 `expr`；L(G) 的元素必须全部由终结符组成。

3. **如何从 Parse Tree 恢复输入 Token 流？**  
   从左到右读取终结符叶子；空产生式的 `ε` 不贡献输入 Token。

4. **怎样证明一份文法有歧义？**  
   找到一个相同的终结符序列，给出两棵不同的树，或两条不同的最左推导，或两条不同的最右推导。

5. **为什么最左推导不能解决 `1 + 2 * 3` 的歧义？**  
   虽然展开位置固定，但根处仍可选择加法或乘法产生式，两种选择都能生成同一输入。

6. **分层文法如何保证乘法优先？**  
   加减在 expr 层组合 term，乘法在 term 层组合 factor；无括号的加减无法藏入 term 中。

7. **`expr → expr MINUS term | term` 为什么形成左结合？**  
   连续减法只能递归进入左操作数 expr，右操作数 term 不能继续生成同层的减法。

8. **matched/unmatched 文法为什么阻止 else 匹配外层 if？**  
   有 else 的产生式要求 then 分支是 matched，因而不能在内部仍有悬空 if 时跳过它。

9. **同一个 Token 序列的两棵树恰好算出相同值，还算歧义吗？**  
   算。歧义的判据是树的结构不同，不是计算结果不同。

10. **写好了 CFG，是否就有了完整 Parser？**  
    还没有。CFG 给出结构规约，还需要解析算法根据输入选择规则、构造树并处理语法错误。
