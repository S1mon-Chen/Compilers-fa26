# LEC 5 · Top-Down Parsing

> SE3355 编译原理（2026 秋）  
> 依据：[LEC-5-Top-Down-Parsing.pptx](../lectures/LEC-5-Top-Down-Parsing.pptx)，共 66 页。以下页码指课件页码。  
> 本笔记按主题整理；标注“补充理解”“补充推导”或“补充实现”的内容用于解释算法及其边界，不是课件原文。  
> 前置：[LEC 4 · Introduction to Parsing](LEC-4-Intro-to-Parsing.md)。

## 1. 本讲主线：从推导到可执行的 Parser（p.2–5、66）

上一讲用 CFG 描述合法结构，并通过文法分层等方式消除歧义。本讲继续解决：**给定实际 Token Stream，Parser 怎样选择产生式并构造语法树？**

```text
从开始符号展开，搜索一条能够匹配输入的推导
    ↓ 每个非终结符写成函数
递归下降：匹配终结符，递归处理非终结符
    ↓ 用当前 Token 提前判断分支
FIRST / FOLLOW / PREDICT
    ↓ 每个选择都唯一时
LL(1)：预测式递归下降，或表驱动分析
```

自顶向下分析（top-down parsing）从语法树的根开始，逐步展开到叶子。按产生式右侧从左到右处理符号时，展开顺序对应**最左推导**。

本讲统一约定：

| 记号 | 含义 |
| --- | --- |
| `INT`、`PLUS` 等大写名称 | 终结符，即 Token 类别 |
| `expr`、`stmt` 等小写名称 | 非终结符 |
| `ε` | 空串，不消费任何 Token |
| `$` | 输入结束标记，不是源程序中的实际字符，也不属于原文法的终结符集合 |
| `current` / lookahead | 当前尚未消费的 Token；查看它不会推进输入 |
| `→` / `⇒` / `⇒*` | 产生式 / 一步推导 / 零步或多步推导 |

## 2. 从文法机械地生成递归下降代码（p.6–17）

### 2.1 贯穿案例：print(1 + 2 * 3);

课件使用以下文法：

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

输入 `print(1 + 2 * 3);` 对应：

```text
PRINT LPAREN INT PLUS INT STAR INT RPAREN SEMI $
```

这里的 `expr_tail` 和 `term_tail` 表示当前表达式、乘法项后面尚未解析的重复部分。文法仍保留 `expr → term → factor` 的优先级层次，但避开了上一讲表达式文法中的左递归。

### 2.2 一条产生式怎样翻译

对产生式：

```text
stmt → PRINT LPAREN expr RPAREN SEMI
```

按照右侧顺序执行：

```text
eat(PRINT)
eat(LPAREN)
e = parseExpr()
eat(RPAREN)
eat(SEMI)
return node(stmt, PRINT, LPAREN, e, RPAREN, SEMI)
```

翻译规则很直接：

| 文法元素 | Parser 操作 |
| --- | --- |
| 终结符 `t` | `eat(t)`：检查当前 Token，匹配成功才消费 |
| 非终结符 `A` | 调用 `parseA()`，得到 A 对应的子树 |
| 右侧符号序列 | 按从左到右的顺序执行 |
| `ε` | 直接成功返回，不移动输入位置 |
| 多条候选产生式 | 需要决定尝试或选择哪一条 |

```text
eat(t):
    if current.kind == t:
        token = current
        advance()
        return token
    else:
        fail("expected " + t)
```

上面的节点构造是示意；实际构造 Parse Tree 时，应保存 `eat()` 返回的 Token 对象，保留数值、词素和源码位置。

### 2.3 最直觉的选择办法：尝试与回溯

课件 p.8 的基本思路是：为当前非终结符逐条尝试产生式，失败后恢复输入位置。

```text
handle(A):
    for each production A → X₁ X₂ ... Xₙ:
        save = position
        children = []
        按顺序匹配 X₁ ... Xₙ
        if all succeed:
            return node(A, children)
        position = save
    return failure
```

除了恢复输入位置，实现时还需要丢弃失败分支构造的临时节点。

**补充理解：局部成功与整份输入成功不同。** 上述简化代码在子函数第一次成功时就返回。对于任意 CFG，外层随后失败时，可能还需要重新进入这个已成功的子函数，尝试它的其他候选。例如：

```text
s → a Z
a → X | X Y
输入：X Y Z
```

若 `a` 先接受 `X`，外层匹配 `Z` 会在 `Y` 处失败；正确解析需要重新选择 `a → X Y`。完整回溯搜索必须保留这些选择点。课件的伪代码用于展示试错思想，不应直接当作任意 CFG 的完整解析器。

### 2.4 完整解析过程

| 阶段 | 当前输入开头 | 动作 |
| --- | --- | --- |
| 开始 | `PRINT ...` | 从 `program → stmt` 展开 |
| 尝试第一条 stmt 规则 | `PRINT ...` | `eat(IF)` 失败，恢复位置 |
| 尝试第二条 stmt 规则 | `PRINT LPAREN ...` | 消费 `PRINT`、`LPAREN`，进入 `expr` |
| 解析第一个 term | `INT PLUS ...` | 展开到 `factor → INT`，消费 `INT(1)` |
| 完成第一个 term | `PLUS ...` | `term_tail → ε` |
| 继续加法 | `PLUS INT STAR ...` | `expr_tail → PLUS term expr_tail` |
| 解析第二个 term | `INT STAR INT ...` | 先读 `INT(2)`，再由 `term_tail` 读入 `STAR INT(3)` |
| 完成表达式 | `RPAREN ...` | `term_tail`、`expr_tail` 均取 `ε` |
| 返回语句 | `RPAREN SEMI $` | 匹配右括号和分号 |
| 接受 | `$` | 确认输入全部消费完毕 |

其中 `2 * 3` 始终在第二个 `term` 内部处理，因此乘法比加法结合得更紧。

**入口必须检查输入结束。** `parseProgram()` 返回成功，只能说明识别出了一个 `program`；还应要求接下来是 `$`，否则可能把一个合法前缀误当成完整程序。

### 2.5 递归下降与最左推导

函数调用顺序：

```text
parseProgram()
  parseStmt()
    eat(PRINT)
    eat(LPAREN)
    parseExpr()
      parseTerm()
        parseFactor()
          eat(INT)
        ...
```

对应的推导开头：

```text
program
⇒ stmt
⇒ PRINT LPAREN expr RPAREN SEMI
⇒ PRINT LPAREN term expr_tail RPAREN SEMI
⇒ PRINT LPAREN factor term_tail expr_tail RPAREN SEMI
⇒ PRINT LPAREN INT term_tail expr_tail RPAREN SEMI
⇒ ...
```

展开节点时从父节点向下建立结构；实现中也可以等子函数返回后，再组装父节点。父节点最终分配或返回的时机，不改变“自顶向下选择产生式”的分析方式。

## 3. 左递归：为什么函数会在读输入前无限调用（p.18–21）

### 3.1 直接左递归

上一讲的左结合加法文法为：

```text
expr → expr PLUS term | term
```

若直接把第一条规则翻译成函数：

```text
parseExpr():
    left = parseExpr()    // 尚未消费任何 Token
    eat(PLUS)
    right = parseTerm()
```

就会出现：

```text
parseExpr() → parseExpr() → parseExpr() → ...
```

问题在于递归调用没有推进输入。失败和回溯都来不及发生。

**补充理解：** 一般地，若 `A ⇒+ A α`，其中 `⇒+` 表示一步或多步推导，则 A 是左递归的。`A → A α` 是直接左递归；`A → B α`、`B → A β` 则形成间接左递归。朴素递归下降不能直接处理这些调用环。

### 3.2 把左递归改成尾部重复

原规则实际描述：先读一个 `term`，然后重复零次或多次 `PLUS term`。

```text
expr      → term expr_tail
expr_tail → PLUS term expr_tail | ε
```

其 Token 序列形状为：

```text
term (PLUS term)*
```

这里的 `*` 是用于解释重复结构的元记号。

### 3.3 通用的直接左递归消除

改写前：

```text
A → A α₁ | ... | A αₙ
  | β₁   | ... | βₘ
```

其中 `βⱼ` 不以 A 开头。引入新的非终结符 `A'`：

```text
A  → β₁ A' | ... | βₘ A'
A' → α₁ A' | ... | αₙ A' | ε
```

生成的串可理解为：

```text
βⱼ (α₁ | ... | αₙ)*
```

**补充理解：适用边界。** 这是消除直接左递归的标准改写，通常假定至少有一条基础分支，并排除 `A → A` 这类无进展规则。若还存在间接递归或可空前缀隐藏的左递归，需要进一步处理；单独套用这条公式不保证最终文法适合 LL(1)。

### 3.4 消除左递归后，怎样保留左结合

把 `PLUS` 换成 `MINUS`，考虑：

```text
expr      → term expr_tail
expr_tail → MINUS term expr_tail | ε
```

`expr_tail` 向右递归，描述的是后续减法项的列表。若希望 `10 - 3 - 2` 仍按左结合计算，构造 AST 时要**向左累积**：

```text
parseExpr():
    left = parseTerm()
    while current.kind == MINUS:
        eat(MINUS)
        right = parseTerm()
        left = node(MINUS, left, right)
    return left
```

```text
left = INT(10)
left = MINUS(INT(10), INT(3))
left = MINUS(MINUS(INT(10), INT(3)), INT(2))
```

最终结构为 `(10 - 3) - 2`，结果是 `5`。

**必须区分三个层面：**

- 改写保持可生成的 Token 序列。
- 新文法的 Parse Tree 增加了尾部非终结符，结构发生变化。
- AST 的结合方向由相应的节点构造规则恢复；不能直接把尾部的右递归理解为减法右结合。

课件 p.20 的循环片段省略了消费 `MINUS` 的语句；上面的补充实现显式加入 `eat(MINUS)`。

## 4. 从回溯转向预测（p.22–24、28）

考虑课件中的分层规则：

```text
factor  → literal | grouped
literal → number | boolean
number  → INT
boolean → TRUE | FALSE
grouped → LPAREN expr RPAREN
```

如果当前 Token 是 `LPAREN`，试错式 Parser 可能先进入 `literal`，在 `INT`、`TRUE`、`FALSE` 处分别失败，再返回去尝试 `grouped`。

但只看各分支可能的第一个 Token，就已经足够：

| 候选 | 可能的开头 |
| --- | --- |
| `literal` | `INT`、`TRUE`、`FALSE` |
| `grouped` | `LPAREN` |

于是可以在递归调用前直接选择 `grouped`，跳过整棵错误子树。这种选择依赖的是**推导之后可能出现的开头**，不能只看产生式右侧字面上的第一个符号。

## 5. FIRST：一串符号可能从什么 Token 开始（p.28–33、38）

### 5.1 定义

对于文法符号序列 `α`：

$$
\operatorname{FIRST}(\alpha)
=\{a\in T\mid\alpha\Rightarrow^*a\beta\}
\;\cup\;
\begin{cases}
\{\varepsilon\}, & \alpha\Rightarrow^*\varepsilon\\
\varnothing, & \text{otherwise}
\end{cases}
$$

- 终结符 `a` 属于 `FIRST(α)`，表示 α 可以推导出以 a 开头的串。
- `ε ∈ FIRST(α)`，表示 α 整体可以消失，称为 **nullable（可空）**。
- `FIRST` 可以作用于单个符号，也可以作用于一整段右侧序列。

基础情况：

```text
FIRST(t) = { t }    // t 是终结符
FIRST(ε) = { ε }
```

### 5.2 计算序列的 FIRST

对于 `X₁ X₂ ... Xₙ`，从左向右扫描：

1. 加入 `FIRST(X₁) − {ε}`。
2. 若 X₁ 可空，继续加入 `FIRST(X₂) − {ε}`。
3. 只有前面的所有符号均可空，才能继续查看后一个符号。
4. 遇到不可空符号就停止。
5. 若整个序列均可空，最后加入 `ε`。

因此，**不能直接把右侧所有符号的 FIRST 无条件求并集**。

例如：

```text
s → a b
a → PUBLIC | PRIVATE | ε
b → STATIC | ε
```

先得到：

```text
FIRST(a) = { PUBLIC, PRIVATE, ε }
FIRST(b) = { STATIC, ε }
```

因为 a 可空，必须继续看 b；又因为 a、b 都可空：

```text
FIRST(s) = { PUBLIC, PRIVATE, STATIC, ε }
```

若把 b 改为 `b → STATIC`，则 `FIRST(s)` 仍包含 `STATIC`，但不再包含 `ε`。

### 5.3 计算非终结符的 FIRST：迭代到不动点

对于 `A → α₁ | ... | αₙ`，把所有候选右侧的 FIRST 合并到 `FIRST(A)`。

```text
初始化：
    FIRST(t) = {t}
    FIRST(ε) = {ε}
    对每个非终结符 A，FIRST(A) = ∅

重复：
    对每条产生式 A → α：
        FIRST(A) = FIRST(A) ∪ FIRST(α)
直到所有集合都不再变化
```

`FIRST(α)` 按第 5.2 节的规则，用当前已知集合计算。空右侧的 FIRST 为 `{ε}`。

循环依赖或跨多层的依赖使一次扫描未必够用。集合只增不减，且可加入的元素有限，所以迭代会终止。

## 6. FOLLOW：一个非终结符结束后，可能遇到什么（p.33–40）

### 6.1 为什么只有 FIRST 不够

继续使用修饰符文法，输入为 `STATIC`：

```text
s → a b
a → PUBLIC | PRIVATE | ε
b → STATIC | ε
```

`STATIC ∉ FIRST(a)`，但输入是合法的，因为：

```text
s ⇒ a b ⇒ b ⇒ STATIC
```

a 应取 `ε`，把 `STATIC` 留给 b。选择空分支时，需要知道**哪些 Token 可以合法地紧跟在 a 后面**。

### 6.2 定义与基本性质

`FOLLOW(A)` 收集在从开始符号推导出的句型中，可能紧接非终结符 A 的终结符；若 A 可以出现在句尾，还包含 `$`。

**补充形式化：**

$$
\operatorname{FOLLOW}(A)
=\{a\in T\mid S\Rightarrow^*\alpha A a\beta\}
\;\cup\;
\begin{cases}
\{\$\}, & S\Rightarrow^*\alpha A\\
\varnothing, & \text{otherwise}
\end{cases}
$$

注意：

- FOLLOW 通常针对非终结符计算，用于决定它结束后能否继续。
- **FOLLOW 不包含 `ε`。** 空串不是后面的一个 Token。
- 开始符号 S 满足 `$ ∈ FOLLOW(S)`。
- FOLLOW 取决于 A 在整份文法中的使用位置，不能只看 A 自己的产生式。

### 6.3 计算规则

对任意产生式中的一次出现：

```text
A → α B β
```

执行两条规则：

**规则一：后缀能提供开头。**

$$
\operatorname{FIRST}(\beta)-\{\varepsilon\}
\subseteq\operatorname{FOLLOW}(B)
$$

**规则二：后缀能够整体消失。** 若 `β ⇒* ε`：

$$
\operatorname{FOLLOW}(A)\subseteq\operatorname{FOLLOW}(B)
$$

两条规则可以同时生效。特别地，B 位于右侧末尾时，β 就是空串，直接把 `FOLLOW(A)` 传给 `FOLLOW(B)`。

### 6.4 修饰符例子的完整计算

以 s 为开始符号：

```text
FOLLOW(s) = { $ }
```

对于 `s → a b`：

1. a 后面是 b，加入 `FIRST(b) − {ε} = {STATIC}`。
2. b 可空，所以还要把 `FOLLOW(s)` 加入 `FOLLOW(a)`。
3. b 在右侧末尾，因此 `FOLLOW(s)` 也加入 `FOLLOW(b)`。

最终结果：

| 非终结符 | FIRST | FOLLOW |
| --- | --- | --- |
| `s` | `{PUBLIC, PRIVATE, STATIC, ε}` | `{$}` |
| `a` | `{PUBLIC, PRIVATE, ε}` | `{STATIC, $}` |
| `b` | `{STATIC, ε}` | `{$}` |

因此 current 为 `STATIC` 或 `$` 时，a 可以选择空分支；其他陌生 Token 不能因为“不在非空分支 FIRST 中”就被一律放过。

### 6.5 FOLLOW 也需要迭代

先算稳定的 FIRST，再计算 FOLLOW：

```text
初始化：
    FOLLOW(S) = {$}
    其他非终结符的 FOLLOW = ∅

重复：
    对每条产生式 A → α B β 中的每个非终结符 B：
        FOLLOW(B) += FIRST(β) − {ε}
        if ε ∈ FIRST(β):
            FOLLOW(B) += FOLLOW(A)
直到所有集合都不再变化
```

这里 `+=` 表示集合并集。应处理右侧的**每一次非终结符出现**，而不仅是最后一个非终结符。

### 6.6 依赖链为什么需要多轮传播

课件 p.40 的文法：

```text
s → a
a → b c
b → ID | ε
c → NUM | ε
```

若每轮只使用上一轮的 FIRST：

| 轮次 | 新得到的信息 |
| --- | --- |
| 0 | 所有非终结符 FIRST 为空 |
| 1 | `FIRST(b) = {ID, ε}`，`FIRST(c) = {NUM, ε}` |
| 2 | `FIRST(a) = {ID, NUM, ε}` |
| 3 | `FIRST(s) = {ID, NUM, ε}` |
| 4 | 无变化，停止 |

FIRST 稳定后，FOLLOW 的最终结果是：

```text
FOLLOW(s) = { $ }
FOLLOW(a) = { $ }
FOLLOW(b) = { NUM, $ }
FOLLOW(c) = { $ }
```

`$` 先从 s 传到 a，再从 a 传给 b、c。实现若原地更新集合或使用工作队列，所需轮数可能不同，但最终最小不动点相同。

## 7. PREDICT：每条产生式的选择条件（p.41–43）

对产生式 `A → α`：

$$
\operatorname{PREDICT}(A\to\alpha)
=\bigl(\operatorname{FIRST}(\alpha)-\{\varepsilon\}\bigr)
\;\cup\;
\begin{cases}
\operatorname{FOLLOW}(A), & \varepsilon\in\operatorname{FIRST}(\alpha)\\
\varnothing, & \text{otherwise}
\end{cases}
$$

这个集合也常称为 SELECT 集合。它作用于**某一条产生式**，不是把同一非终结符的全部候选混在一起。

两种选择依据：

- current 是 α 可能的首个 Token，可以通过这条规则继续匹配。
- α 可以整体为空，且 current 属于 `FOLLOW(A)`，可以让这部分结束并把输入留给后续结构。

对于显式空产生式：

```text
PREDICT(A → ε) = FOLLOW(A)
```

修饰符例子的答案：

| 产生式 | PREDICT |
| --- | --- |
| `s → a b` | `{PUBLIC, PRIVATE, STATIC, $}` |
| `a → PUBLIC` | `{PUBLIC}` |
| `a → PRIVATE` | `{PRIVATE}` |
| `a → ε` | `{STATIC, $}` |
| `b → STATIC` | `{STATIC}` |
| `b → ε` | `{$}` |

**补充理解：** `s → a b` 虽然没有字面上的 `ε`，但整个右侧可空，所以计算 PREDICT 时同样需要 FOLLOW。只检查“是否写成 `A → ε`”会漏掉这种情况。

课件 p.41 末尾写了“为什么不求非终结符的 FOLLOW 集合？”，与前文定义矛盾，应视为疑似笔误。本算法确实计算非终结符的 FOLLOW；终结符直接通过 `eat()` 匹配，不需要为预测产生式单独计算它们的 FOLLOW。

## 8. LL(1)：一个 lookahead 唯一决定产生式（p.44–49）

### 8.1 名称与判定条件

| 记号 | 含义 |
| --- | --- |
| 第一个 L | Left-to-right，从左到右读取输入 |
| 第二个 L | Leftmost derivation，构造最左推导 |
| 1 | 一个 Token 的 lookahead |

“看一个 Token”指每次作出当前产生式选择时使用的信息量。随着匹配推进，current 会更新，并非整个解析过程只能查看同一个 Token。

对同一个非终结符 A 的任意两条不同产生式，要满足：

$$
\operatorname{PREDICT}(A\to\alpha_i)
\cap
\operatorname{PREDICT}(A\to\alpha_j)
=\varnothing\qquad(i\ne j)
$$

这样每个 `(A, lookahead)` 才至多对应一条产生式。

**补充理解：** LL(1) 文法无歧义，但无歧义文法不一定是 LL(1)。文法不能被一个 Token 唯一预测，不代表它描述的语言本身有歧义。

### 8.2 构造 LL(1) 分析表

行是非终结符，列是终结符以及 `$`。对于每条 `A → α`：

1. 对每个 `a ∈ FIRST(α) − {ε}`，把该产生式填入 `M[A, a]`。
2. 若 `ε ∈ FIRST(α)`，对每个 `b ∈ FOLLOW(A)`，也填入 `M[A, b]`。

等价于对 `PREDICT(A → α)` 中的每个元素填表。

| 单元格内容 | 含义 |
| --- | --- |
| 一条产生式 | 唯一展开动作 |
| 空白 | 当前非终结符不能接受这个 lookahead，报告语法错误 |
| 两条或更多不同产生式 | LL(1) 冲突，当前文法不能直接用于确定的 LL(1) 分析 |

**空白格与 `ε` 格完全不同。** 前者表示错误，后者表示合法地展开为空。

### 8.3 表达式文法的完整 FIRST / FOLLOW

课件 p.47–52 在这里把 **expr 作为开始符号**，独立解析表达式：

```text
expr      → term expr_tail
expr_tail → PLUS term expr_tail | ε
term      → factor term_tail
term_tail → STAR factor term_tail | ε
factor    → INT | LPAREN expr RPAREN
```

| 非终结符 | FIRST | FOLLOW |
| --- | --- | --- |
| `expr` | `{INT, LPAREN}` | `{RPAREN, $}` |
| `expr_tail` | `{PLUS, ε}` | `{RPAREN, $}` |
| `term` | `{INT, LPAREN}` | `{PLUS, RPAREN, $}` |
| `term_tail` | `{STAR, ε}` | `{PLUS, RPAREN, $}` |
| `factor` | `{INT, LPAREN}` | `{STAR, PLUS, RPAREN, $}` |

**补充推导：FOLLOW 的来源。**

- expr 是开始符号，所以有 `$`；`factor → LPAREN expr RPAREN` 又提供 `RPAREN`。
- `expr_tail` 在 expr 右侧末尾，继承 `FOLLOW(expr)`。
- term 后面的 `expr_tail` 提供 `PLUS`；它可空，因此 term 也继承 `FOLLOW(expr)`。
- `term_tail` 继承 `FOLLOW(term)`。
- factor 后面的 `term_tail` 提供 `STAR`；它可空，因此 factor 也继承 `FOLLOW(term)`。

**与开头 print 案例的区别：** 若使用第 2.1 节完整文法并以 program 为开始符号，expr 后面总有 `RPAREN`，其 FOLLOW 为 `{RPAREN}`，不包含 `$`。不能直接把独立表达式文法的 FOLLOW 当成完整程序文法的 FOLLOW。

### 8.4 完整分析表

为节省空间，单元格只写产生式右侧；`—` 表示错误。

| 非终结符 | INT | LPAREN | PLUS | STAR | RPAREN | $ |
| --- | --- | --- | --- | --- | --- | --- |
| `expr` | `term expr_tail` | `term expr_tail` | — | — | — | — |
| `expr_tail` | — | — | `PLUS term expr_tail` | — | `ε` | `ε` |
| `term` | `factor term_tail` | `factor term_tail` | — | — | — | — |
| `term_tail` | — | — | `ε` | `STAR factor term_tail` | `ε` | `ε` |
| `factor` | `INT` | `LPAREN expr RPAREN` | — | — | — | — |

例如 `M[term_tail, PLUS] = ε`：乘法项已经结束，`PLUS` 应留给外层的 `expr_tail`，这里不消费它。

## 9. 表驱动 LL(1)：栈、输入与分析表（p.50–53）

### 9.1 栈表示尚未匹配的符号

本节统一把**栈顶写在左侧**：

```text
stack = [expr, $]
input = [INT, PLUS, INT, STAR, INT, $]
```

操作规则：

- 栈顶为终结符：必须等于 current，然后弹栈并消费输入。
- 栈顶为非终结符：用 `(栈顶, current)` 查表，把栈顶替换成产生式右侧。
- 选中 `ε`：只移除非终结符，不压入任何符号，不消费输入。
- 栈与输入同时到达 `$`：接受。

**补充实现：** 以下伪代码显式处理错误格和结束标记。

```text
stack = [S, $]                  // 左端为栈顶

while true:
    X = stack.first
    a = current.kind

    if X == $:
        if a == $: accept
        else: error("unexpected trailing input")

    if X is a terminal:
        if X != a: error("terminal mismatch")
        removeFirst(stack)
        consume()
    else:
        p = M[X, a]
        if p is empty: error("no matching production")
        removeFirst(stack)
        prepend p.rhs to stack  // 保留右侧从左到右的顺序；ε 对应空列表
```

若使用传统的逐个 `push` 操作，需要把右侧**逆序入栈**，才能让最左边的符号先出栈。例如 `A → B C` 应先 push C，再 push B。

### 9.2 完整运行：1 + 2 * 3

下表记录执行动作之前的状态；数字统一用 `INT` 表示。

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

由这些展开可以构造完整 Parse Tree；省略文法中间节点后，表达式的 AST 为：

```text
PLUS
├── INT(1)
└── STAR
    ├── INT(2)
    └── INT(3)
```

**补充理解：** 只有符号栈和查表循环时，得到的是识别过程。若要返回语法树，还应在展开时创建节点，并让栈项记录对应节点或子节点位置。

### 9.3 与预测式递归下降的关系

两种实现都可以使用相同的 PREDICT 信息：

| 维度 | 预测式递归下降 | 表驱动 LL(1) |
| --- | --- | --- |
| 选择存在哪里 | 函数中的 `if` / `switch` | 分析表 `M[A, a]` |
| 未完成的工作存在哪里 | 函数调用栈 | 显式符号栈 |
| 节点构造 | 由函数组合子节点 | 为栈项关联节点或语义动作 |
| 常见优势 | 贴近文法，便于定制诊断 | 主循环统一，便于机械生成 |

递归下降是一种实现组织方式，既可以带回溯，也可以使用预测；不能把所有递归下降都等同于 LL(1)。

## 10. 公共前缀与左因子提取（p.54–55）

### 10.1 一个 Token 暂时不足以区分分支

```text
stmt → IDENT ASSIGN expr SEMI
     | IDENT LPAREN args RPAREN SEMI
```

两条规则分别表示赋值和函数调用，但都从 `IDENT` 开始：

```text
PREDICT(第一条) = { IDENT }
PREDICT(第二条) = { IDENT }
```

于是 `M[stmt, IDENT]` 出现两条规则。此时输入还没有读到能区分它们的 `ASSIGN` 或 `LPAREN`。

### 10.2 提取公共前缀，推迟选择

```text
stmt      → IDENT stmt_tail
stmt_tail → ASSIGN expr SEMI
          | LPAREN args RPAREN SEMI
```

先统一匹配 `IDENT`，再由新的 current 选择 `stmt_tail` 分支：

| current | 选择 |
| --- | --- |
| `ASSIGN` | 赋值分支 |
| `LPAREN` | 调用分支 |

一般形式为：

```text
改写前：A → α β₁ | α β₂
改写后：A → α A'
        A' → β₁ | β₂
```

左因子提取（left factoring）保留语言，并把选择推迟到公共前缀之后。

### 10.3 两种改写都不保证 LL(1)

**补充理解：** 消除左递归与提取公共前缀分别解决特定障碍。最终仍需要检查同一非终结符各候选的 PREDICT 是否两两不交。

即使没有字面上的公共前缀，也可能出现 FIRST 重叠：

```text
s → a | b
a → ID
b → ID NUM
```

两条 s 规则的 PREDICT 都包含 `ID`。

可空候选还会引起 FIRST/FOLLOW 冲突：

```text
s → a ID
a → ID | ε
```

因为 `FOLLOW(a) = {ID}`：

```text
PREDICT(a → ID) = {ID}
PREDICT(a → ε)  = {ID}
```

当前看到 `ID` 时，无法用一个 Token 判断它应由 a 消费，还是应留给 s 右侧最后的 `ID`。该文法生成 `ID` 和 `ID ID`，两者各有唯一结构，却仍不满足 LL(1)。

## 11. 语法错误：发现位置与恢复策略（p.56–59）

### 11.1 两类直接错误

**终结符不匹配：**

```text
stack: RPAREN SEMI $
input: SEMI $
```

当前期待 `RPAREN`，实际遇到 `SEMI`，可报告缺少右括号。

**分析表单元为空：**

```text
输入：1 + * 2
stack: term expr_tail $
input: STAR INT $
```

`M[term, STAR]` 为空；当前 term 只能从 `INT` 或 `LPAREN` 开始，可以据此生成期望 Token 集合。

**补充理解：** 课件给出的缺右括号状态用来说明终结符不匹配。严格按完整文法的 LL(1) 表分析 `print(1 + 2;` 时，也可能先在 `term_tail` 遇到 `SEMI` 时查到错误格。具体最先报错的位置取决于当时的解析状态，以及是否已执行恢复动作。

### 11.2 恢复的目标

恢复的目标是继续分析后面的程序，尽量给出更多有用诊断。恢复出的结构未必等于用户原本想写的程序。

| 策略 | 对输入和状态的操作 | 课件例子 |
| --- | --- | --- |
| 插入 | 合成缺失 Token，满足当前期望，不消费 current | 在 `SEMI` 前补 `RPAREN` |
| 删除 | 消费意外 Token，保留待匹配结构，再试一次 | `1 + * 2` 中删掉 `STAR` |
| 同步 | 跳过输入并调整解析状态，回到可信语法边界 | 跳到 `SEMI`、`RBRACE` 或 `$` |

局部修复可以生成少量候选，再向前模拟几步，比较：

- 修改的 Token 数量或编辑代价。
- 能否持续推进解析。
- 是否立刻产生新的错误。

课件中的删除修复会观察下一个 Token。这属于错误恢复中的试探，不改变正常路径上 LL(1) 使用一个 Token 选择产生式的定义。

### 11.3 同步集合依赖当前上下文

课件给出以下示例：

| 上下文 | 可考虑的同步 Token |
| --- | --- |
| 表达式 | `RPAREN`、`COMMA`、`SEMI` |
| 语句 | `SEMI`、`RBRACE` |
| 代码块 | `RBRACE` |
| 文件 | 声明起始 Token、`$` |

```text
while current.kind not in sync(context):
    consume()
```

**补充理解：** FOLLOW 常可用于设计同步集合，但恢复策略还要考虑语句、代码块等边界。到达同步点后，需要决定弹出哪些栈项、返回哪层函数，以及是否消费同步 Token；只跳过输入后原样重试，可能再次遇到同一个错误。恢复动作必须保证输入或解析状态有所进展，并避免越过 EOF。

## 12. BNF 与 EBNF：更紧凑地写文法（p.26）

BNF（Backus–Naur Form）是书写 CFG 的一种记法，常用 `::=` 表示产生式。EBNF（Extended BNF）增加可选与重复等简写。

课件例子：

```text
BNF：
args ::= ε | expr rest
rest ::= ε | COMMA expr rest

EBNF：
args ::= [ expr { COMMA expr } ]
```

在这套 EBNF 约定中：

- `[ ... ]` 表示可选，即出现零次或一次。
- `{ ... }` 表示重复零次或多次。
- 因而 args 可以为空，也可以是一个或多个用逗号分隔的表达式。

**补充理解：** 这里的方括号和花括号是描述文法的元记号，不是待匹配的输入 Token。EBNF 的简写可展开成普通 CFG；写得更短本身不会解决歧义或 LL(1) 冲突。

## 13. ANTLR4：由生成器处理更多预测与改写工作（p.60–65）

课件用 ANTLR4 展示超出经典 LL(1) 的工程方案：编写 `.g4` 文法，由工具生成 Lexer、Parser，并提供 Parse Tree 及 Listener / Visitor 遍历支持。

```text
Language.g4
    ↓ 生成
Lexer + Parser
    ↓ 解析输入
Parse Tree
    ↓ Listener / Visitor
后续结构构造与处理
```

### 13.1 对受支持的直接左递归进行改写

课件中的简化规则：

```text
expr
    : expr STAR expr
    | expr PLUS expr
    | INT
    ;
```

生成器区分：

| 分支 | 分类与用途 |
| --- | --- |
| `INT` | 基础分支，作为不递归的入口 |
| `expr STAR expr` | 直接左递归分支，改写为受优先级控制的后续处理 |
| `expr PLUS expr` | 同上 |

在课件这个受支持的表达式规则中，`STAR` 分支排在 `PLUS` 前面，生成器据此让乘法具有更高优先级：

```text
1 + 2 * 3  →  PLUS(INT(1), STAR(INT(2), INT(3)))
```

这是 ANTLR 的规则处理约定；普通 CFG 的候选排列顺序本身不定义运算符优先级。

课件特别强调边界：支持的是符合工具要求的直接左递归，`expr → term → expr` 这样的间接左递归不会因此自动解决。

### 13.2 Adaptive LL(*)：需要时继续观察输入

对于：

```text
stmt → IDENT ASSIGN expr SEMI
     | IDENT LPAREN args RPAREN SEMI
```

看到 `IDENT` 还无法区分，就继续观察：

```text
IDENT ASSIGN ...  → 赋值分支
IDENT LPAREN ...  → 调用分支
```

经典 LL(1) 可以通过左因子提取解决；课件介绍的 Adaptive LL(*) 则根据预测需要查看更多输入，减少手工提取公共前缀的工作。

更强的预测能力不会自动消除真正的文法歧义。若同一输入确实有多种结构，仍需明确语言希望采用哪种结构，以及由文法还是工具策略实现。

## 14. 易错点与复习自测

### 14.1 易错点速查

| 易错点 | 正确理解 |
| --- | --- |
| 非空分支都不匹配，就直接选 `ε` | 还必须检查 current 是否允许出现在该非终结符之后 |
| FIRST 只针对非终结符 | 也可以计算终结符、空串和整段符号序列的 FIRST |
| `FIRST(XY) = FIRST(X) ∪ FIRST(Y)` | 只有 X 可空，才能继续引入 Y 的开头；ε 需按整体可空性处理 |
| FOLLOW 包含 `ε` | FOLLOW 只包含终结符和可能的 `$` |
| FOLLOW 只看紧邻的下一个符号 | 要看整个右侧后缀；后缀可空时还要继承左侧的 FOLLOW |
| FIRST/FOLLOW 扫一遍就能算完 | 依赖信息可能需要多轮传播，直到集合不再变化 |
| 可空规则就是字面上的 `A → ε` | `A → B C` 也可能整体可空 |
| 空表格与填 `ε` 的表格一样 | 空表格表示错误，`ε` 表示合法的不消费输入动作 |
| 消除左递归就保证 LL(1) | 还要检查公共前缀、FIRST/FOLLOW 等造成的 PREDICT 冲突 |
| 尾部右递归让减法自动右结合 | AST 可以通过向左累积保持左结合 |
| 所有递归下降都是 LL(1) | 递归下降也可以带回溯或使用更强的预测 |
| LL(1) 冲突就证明文法有歧义 | 也可能只是一个 Token 不足以预测 |
| 分析函数成功就接受输入 | 入口还需检查是否到达 `$` |
| 同一个 expr 的 FOLLOW 永远相同 | FOLLOW 依赖完整文法及开始符号 |

### 14.2 自测与简答

1. **自顶向下分析为什么对应最左推导？**  
   按右侧从左到右递归处理时，每次展开的都是当前最左侧尚未处理的非终结符。

2. **为什么 `expr → expr PLUS term` 不能直接翻译成普通递归下降？**  
   它会在消费输入之前再次调用 `parseExpr()`，造成无进展的递归。

3. **如何把 `A → A x | y` 消除直接左递归？**  
   改为 `A → y A'`、`A' → x A' | ε`。

4. **若 `FIRST(B) = {b, ε}`、`FIRST(C) = {c}`，那么 `FIRST(BC)` 是什么？**  
   `{b, c}`。B 可空，所以加入 c；C 不可空，所以不加入 ε。

5. **在 `A → B C` 中，什么时候把 `FOLLOW(A)` 加入 `FOLLOW(B)`？**  
   当 C 可以推导为空时。无论 C 是否可空，都应加入 `FIRST(C) − {ε}`。

6. **如果 `A → α` 的右侧可空，PREDICT 应如何计算？**  
   `(FIRST(α) − {ε}) ∪ FOLLOW(A)`。

7. **怎样判断一份文法能否使用无冲突的 LL(1) 分析表？**  
   检查同一非终结符不同产生式的 PREDICT 是否两两不交，等价于每个表格单元至多一条产生式。

8. **表达式例子中，看到 PLUS 时 term_tail 为什么选择 ε？**  
   PLUS 属于 `FOLLOW(term_tail)`，应结束当前乘法项，把 PLUS 留给外层表达式处理。

9. **栈顶在左侧时，展开 `A → B C` 后谁先处理？**  
   B。若逐个 push，应按 C、B 的顺序入栈。

10. **如何保持 `10 - 3 - 2` 的左结合？**  
    每读完一个右操作数，就更新 `left = MINUS(left, right)`，得到 `MINUS(MINUS(10, 3), 2)`。

11. **左因子提取改变了什么？**  
    先匹配公共前缀，再在新的非终结符处选择后缀；语言保持不变，选择时机被推迟。

12. **错误恢复为什么不能总是跳过当前 Token？**  
    当前 Token 可能属于外层结构，例如表达式后的分号；盲目删除可能破坏原本正确的后续内容。

13. **ANTLR4 的更强预测是否意味着任意 CFG 都能直接处理？**  
    不能。课件明确指出间接左递归等限制，真正的歧义也仍需要语言设计或消歧策略处理。

## 15. 本讲与下一讲的衔接（p.66）

本讲把“寻找推导”落实为一个个可执行的选择：

```text
文法改写
    → 计算 FIRST / FOLLOW
    → 计算每条产生式的 PREDICT
    → 检查 LL(1) 冲突并构造分析表
    → 用递归函数或显式栈执行
    → 在错误处诊断并恢复
```

自顶向下分析从开始符号出发，逐步展开到输入 Token。下一讲将研究自底向上分析：从已读入的 Token 出发，通过归约逐步形成更大的结构，最终得到开始符号。
