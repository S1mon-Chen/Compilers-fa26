# LEC 9 · Type System (I)

> SE3355 编译原理（2026 秋）  
> 依据：[LEC-9-Type-System-I.pptx](../lectures/LEC-9-Type-System-I.pptx)，共 54 页。以下页码指课件页码。  
> 本笔记按主题整理；标注“补充理解”“补充推导”或“补充实现”的内容用于解释规则及其边界，不是课件原文。涉及具体语言的澄清附官方资料链接。  
> 前置：[LEC 8 · AST and Name Resolving](LEC-8-AST-and-Name-Resolving.md)；正则语言与自动机可回顾 [LEC 2](LEC-2-Lexical-Analysis-Regex.md) 和 [LEC 3](LEC-3-Lexical-Analysis-Impl-of-RE.md)。

## 1. 本讲主线：把语义约束变成可执行规则（p.2–8、23）

下面的表达式可能符合文法，却不符合我们希望赋予语言的含义：

```text
1 + true
if 1 then 2 else 3
false(42)
```

对于本讲的 SimpleType，它们分别违反“加法要求整数”“条件要求布尔值”“被调用对象必须是函数”的约束。

```text
源代码 → Parser → AST → Resolver → Bound AST → Type Checker
                         程序结构      声明身份       类型结论与诊断
```

本讲要建立的联系是：

```text
语言的语义约定
    ↓ 写成局部类型规则
Γ ⊢ e : T
    ↓ 按 AST 递归实现
check(e, Γ) → Type
    ↓ 保存检查结果
带类型信息的 AST、转换节点、错误诊断
```

### 1.1 不同错误属于不同阶段

| 错误类别 | 例子 | 主要检查内容 |
| --- | --- | --- |
| 语法错误 | 缺少括号、关键字、分隔符 | Token 序列能否构成合法结构 |
| 名字错误 | 未定义变量、同层重复声明、越出作用域 | 名字指向哪个声明，该声明是否可见 |
| 类型错误 | 把 Bool 用于整数加法、函数参数类型不符 | 操作与操作数是否满足类型规则 |
| 运行时错误 | 除零、越界、资源耗尽 | 具体执行状态是否满足运行条件 |

**类型检查通过，只说明程序满足该类型系统检查的性质。** 它不自动证明算法正确、程序会终止，或所有运行时错误都已排除。

**补充理解：** 表中的阶段是本讲的职责划分。更强的静态分析可以提前发现部分越界、除零等问题；但本讲的 `Int` / `Bool` 规则没有表达这些约束。

### 1.2 静态类型与动态类型

| 维度 | 静态类型检查 | 动态类型检查 |
| --- | --- | --- |
| 何时判断 | 执行前，根据程序结构建立类型结论 | 执行时，根据实际值判断操作是否合法 |
| 主要依据 | 声明、环境、表达式规则 | 运行时的值及其类型信息 |
| 非法操作 | 在静态检查阶段拒绝 | 执行到该操作时报告错误 |

二者可以并存。课件中的 Java 例子是：

```java
Object x = read();
String s = (String) x;
```

- 编译期确认这种显式转换允许出现，并以 `String` 检查后续对 `s` 的使用。
- 运行时检查非空引用是否指向可转换为 `String` 的对象；不满足则抛出 `ClassCastException`。

**补充理解：** `x == null` 时，这个转换成功，结果仍是 `null`。因此“不是 String 就失败”需要排除空引用。见 [Java 规范：Checked Narrowing Reference Conversions](https://docs.oracle.com/javase/specs/jls/se25/html/jls-5.html#jls-5.1.6.3)。静态类型也不要求每个表达式都由程序员手写类型标注，编译器可以推导部分类型。

## 2. 类型也可以描述组件接口：Shell 的 RT 案例（p.9–10）

### 2.1 语法正确，管道仍可能组合错误

```sh
find . |
grep -E 'book[0-9]+[.]txt' |
xargs cat
```

假设目录下存在文件名 ` book0.txt`，其开头有一个空格。`find` 可以输出：

```text
./ book0.txt
```

我们希望把它作为一个完整路径传给 `cat`，但默认的 `xargs` 会按空白等规则解析输入，可能形成两个参数：`./` 和 `book0.txt`。

问题出在相邻命令对数据格式的约定不一致：生产者输出的一个路径，消费者可能理解为两个参数。

### 2.2 用正则语言表达输入、输出约束

课件将流类型简化为“每行可能具有的形状”，将命令类型表示为：

```text
输入流类型 → 输出流类型

grep ...  : .*       → .*book[0-9]+[.]txt.*
xargs cat : SafePath → .*

SafePath = [^ \t]+ | ".+" | '.+'
```

组合时要检查：

$$
L(\operatorname{Output}(\texttt{grep}))
\subseteq
L(\operatorname{Input}(\texttt{xargs cat}))
$$

其中 `L(R)` 表示正则表达式 `R` 描述的语言。`./ book0.txt` 是包含关系不成立的一个反例：它可能由上游产生，但不符合课件给出的安全输入约束。

**这里的类型作用是把组件之间的隐含约定写成机器可以检查的接口。** RT 在 Shell 和命令之上叠加类型规范，用正则语言描述流。论文见 [RT: Regular Types for the Streaming Shell](https://atlas.cs.brown.edu/pdf/rt:osdi:2026.pdf)。

**补充理解：** 上述 `SafePath` 是课件示意，不能当作覆盖所有文件名、引号、转义和换行行为的完整 Shell 安全规范。类型检查的保证依赖接口规范是否准确。

**补充推导：** 正则语言的包含关系可以转为判空问题：

$$
L_{\mathrm{out}} \subseteq L_{\mathrm{in}}
\iff
L_{\mathrm{out}} \cap \overline{L_{\mathrm{in}}} = \varnothing
$$

这把类型兼容性与前几讲的自动机知识联系起来：若交集非空，还可以寻找一个被上游允许、被下游拒绝的反例串。

## 3. SimpleType：语言、类型与判断式（p.12–13）

### 3.1 先固定讨论的语言

```text
类型：
T ::= Int | Bool | T -> T

表达式：
e ::= n | true | false | x
    | e + e | e < e
    | if e then e else e
    | let x : T = e in e
    | fun (x : T) -> T { e }
    | e(e)
```

- `n` 表示整数字面量，`x` 表示变量。
- 函数是单参数函数，参数和返回值都有显式类型标注。
- `if`、`let` 都是表达式，因此也有结果类型。
- 这一核心语言尚未加入字符串、用户定义类型、隐式转换或递归绑定；后文将另行讨论扩展。

**补充理解：** `Int -> Bool` 表示接收整数、返回布尔值的函数。本笔记按常用的右结合约定解释箭头：`Int -> Int -> Bool` 即 `Int -> (Int -> Bool)`；若参数本身是函数，则用 `(Int -> Int) -> Bool` 明确分组。

### 3.2 `Γ ⊢ e : T` 怎么读

$$
\Gamma \vdash e : T
$$

读作：**在类型环境 Γ 中，表达式 e 的类型是 T。**

| 记号 | 含义 |
| --- | --- |
| `Γ`，Gamma | 类型环境，记录变量或 Symbol 的类型 |
| `⊢` | 在左侧假设下，可以推导右侧结论 |
| `e` | 正在检查的表达式 |
| `T`、`τ` | 类型的元变量，课件中两种记法都出现过 |
| `Γ(x) = T` | 从环境查到 x 的类型为 T |
| `Γ, x:T` | 在当前环境上扩展一个局部绑定 |
| `dom(Γ)` | 环境中已经登记的键的集合 |

`Γ ⊢ e : T` 是关于表达式的静态判断，不是在计算 `e` 的运行结果。

### 3.3 类型规则怎么读

$$
\frac{\text{前提}_1 \qquad \text{前提}_2 \qquad \cdots}
     {\text{结论}}
\;[\text{规则名}]
$$

横线上方列出必须成立的前提，横线下方列出可推出的结论。没有前提的规则可以直接作为推导树的叶子。

检查表达式时，可以从目标结论向上寻找所需前提；构成证明时，则由叶子的事实逐层推出根节点结论。

## 4. 基础规则：常量、变量与运算符（p.14–16）

### 4.1 常量规则

$$
\frac{}{\Gamma \vdash n : \mathrm{Int}}\;[\mathrm{INT}]
\qquad
\frac{}{\Gamma \vdash \mathrm{true} : \mathrm{Bool}}\;[\mathrm{TRUE}]
\qquad
\frac{}{\Gamma \vdash \mathrm{false} : \mathrm{Bool}}\;[\mathrm{FALSE}]
$$

常量不需要查询环境，但判断式仍可以带着 Γ。比如在任意合法环境中，`42 : Int` 都成立。

### 4.2 变量规则

$$
\frac{\Gamma(x)=T}{\Gamma \vdash x:T}\;[\mathrm{VAR}]
$$

例如：

```text
Γ = {x : Int, flag : Bool}

Γ ⊢ x    : Int
Γ ⊢ flag : Bool
```

若 `x ∉ dom(Γ)`，就无法建立这条规则的前提。在只使用一个环境的教学模型中，可在这里报告未定义变量；在上一讲的分阶段实现中，Resolver 应已处理该名字错误，类型检查器接收绑定结果或 `ErrorSymbol`。

### 4.3 加法与小于比较

$$
\frac{\Gamma \vdash e_1 : \mathrm{Int}\qquad
      \Gamma \vdash e_2 : \mathrm{Int}}
     {\Gamma \vdash e_1+e_2 : \mathrm{Int}}
\;[\mathrm{ADD}]
$$

$$
\frac{\Gamma \vdash e_1 : \mathrm{Int}\qquad
      \Gamma \vdash e_2 : \mathrm{Int}}
     {\Gamma \vdash e_1<e_2 : \mathrm{Bool}}
\;[\mathrm{LT}]
$$

二者对输入的要求相同，结果类型不同：

| 表达式 | 操作数类型 | 结果 |
| --- | --- | --- |
| `1 + 2` | Int、Int | Int |
| `1 < 2` | Int、Int | Bool |
| `1 + true` | Int、Bool | 无法应用 `[ADD]` |
| `true < false` | Bool、Bool | 无法应用 `[LT]` |

**本讲的规则不允许把 Bool 自动转换成 Int。** 其他语言是否允许，必须查它们自己的规则。

## 5. 条件表达式与手工推导（p.17、21–22、25）

### 5.1 `[IF]` 有三个前提

$$
\frac{\Gamma \vdash c : \mathrm{Bool}\qquad
      \Gamma \vdash e_1 : T\qquad
      \Gamma \vdash e_2 : T}
     {\Gamma \vdash \mathrm{if}\ c\ \mathrm{then}\ e_1\ \mathrm{else}\ e_2 : T}
\;[\mathrm{IF}]
$$

检查时必须确认：

1. 条件是 `Bool`。
2. 两个分支具有同一个类型 `T`。
3. 整个 `if` 的结果类型为这个 `T`。

这里的 `T` 可以是 `Int`、`Bool`，也可以是函数类型；规则要求同型，没有把分支限制成整数。

### 5.2 正例：`if x < 0 then 0 else x + 1`

已知 `Γ = {x : Int}`。

```text
Γ(x) = Int      [INT]
──────────       ─────────
x : Int          0 : Int
───────────────────────── [LT]
         x < 0 : Bool

[INT]            Γ(x) = Int      [INT]
─────────        ──────────      ─────────
0 : Int          x : Int         1 : Int
                 ───────────────────────── [ADD]
                          x + 1 : Int
```

上图为紧凑起见省略了每行重复的 `Γ ⊢`。最后应用：

$$
\frac{\Gamma \vdash x<0 : \mathrm{Bool}\qquad
      \Gamma \vdash 0 : \mathrm{Int}\qquad
      \Gamma \vdash x+1 : \mathrm{Int}}
     {\Gamma \vdash \mathrm{if}\ x<0\ \mathrm{then}\ 0\ \mathrm{else}\ x+1 : \mathrm{Int}}
\;[\mathrm{IF}]
$$

对应带类型的 AST：

```text
If : Int
├── condition: LessThan : Bool
│   ├── Name(x) : Int
│   └── Int(0)  : Int
├── then: Int(0) : Int
└── else: Add : Int
    ├── Name(x) : Int
    └── Int(1)  : Int
```

### 5.3 反例：失败的是哪条约束

```text
if x < 0 then true else x + 1
```

| 检查点 | 已知类型 | 是否满足 `[IF]` |
| --- | --- | --- |
| 条件 `x < 0` | Bool | 满足 |
| then 分支 `true` | Bool | 单个分支本身合法 |
| else 分支 `x + 1` | Int | 单个分支本身合法 |
| 两分支同型 | Bool 与 Int | 不满足 |

诊断应指向“两个分支类型不一致”，而不是笼统地报告整个表达式无法解析。

**补充理解：** 静态检查会检查两个分支，即使某次执行只会走其中一个。按照当前规则，`if true then 1 else false` 仍不合法。自动取联合类型或寻找共同父类型都需要新增规则。

## 6. `let`：先检查初始化，再扩展环境（p.18、28）

$$
\frac{\Gamma \vdash e_1 : T_1\qquad
      \Gamma,x:T_1 \vdash e_2 : T_2}
     {\Gamma \vdash \mathrm{let}\ x:T_1=e_1\ \mathrm{in}\ e_2 : T_2}
\;[\mathrm{LET}]
$$

`T₁` 同时出现在声明标注与初始化式的类型结论中，因此规则已经包含“初始化式类型必须等于声明类型”的要求。

```text
旧环境 Γ
    │ 检查 e₁，要求得到声明类型 T₁
    ▼
局部环境 Γ′ = Γ 扩展 {x : T₁}
    │ 检查 e₂，得到 T₂
    ▼
整个 let : T₂
离开后继续使用旧环境 Γ
```

### 6.1 `let` 的类型由主体决定

```text
let x : Int = 1 in x < 2
```

初始化式是 `Int`，但主体 `x < 2` 是 `Bool`，所以整个 `let` 是 `Bool`。

```text
let x : Int = true in x + 1
```

初始化式得到 `Bool`，无法满足声明要求的 `Int`，因此不能建立完整推导。

### 6.2 新变量只在主体中可见

**补充推导：** 设外层有 `x₀ : Int`：

```text
let x : Int = x + 1 in x < 10
              ↑外层     ↑内层
```

- 检查 `x + 1` 时使用旧 Γ，其中的 `x` 指向外层声明 `x₀`。
- 检查 `x < 10` 时使用扩展环境，其中的 `x` 指向新声明 `x₁`。
- 离开 `let` 后，新绑定不影响外层环境。

若外层没有 `x`，`let x : Int = x + 1 in x` 中的初始化式使用未定义名字。**本讲 `[LET]` 是非递归绑定规则**；支持 `let rec` 需要单独定义环境进入时机和递归约束。

## 7. 函数定义与函数调用（p.19–20）

### 7.1 函数定义 `[FUN]`

$$
\frac{\Gamma,x:T_1 \vdash e : T_2}
     {\Gamma \vdash \mathrm{fun}(x:T_1)\to T_2\{e\} : T_1\to T_2}
\;[\mathrm{FUN}]
$$

检查步骤：

1. 读取参数类型 `T₁` 和声明的返回类型 `T₂`。
2. 在局部环境中加入 `x:T₁`。
3. 检查函数体 `e`，要求它的类型等于 `T₂`。
4. 返回整个函数表达式的类型 `T₁ → T₂`。

```text
fun (x : Int) -> Bool { x < 0 }   : Int -> Bool
fun (x : Int) -> Int  { x < 0 }   : 类型错误
```

第二个函数的标注声称返回 `Int`，但函数体是 `Bool`。显式标注是待验证的约定，不能代替对函数体的检查。

### 7.2 函数调用 `[APP]`

$$
\frac{\Gamma \vdash f : T_1\to T_2\qquad
      \Gamma \vdash a : T_1}
     {\Gamma \vdash f(a) : T_2}
\;[\mathrm{APP}]
$$

| 检查 | 失败例子 |
| --- | --- |
| 被调用表达式具有函数类型 | `false(42)` |
| 实参类型等于形参类型 | 已知 `f : Int -> Bool`，却调用 `f(true)` |
| 成功后返回函数的结果类型 | `f(42) : Bool` |

`f` 可以是任意表达式，只要它的类型是函数类型，不必是一个简单函数名。

### 7.3 完整例子：`let`、函数和调用组合（补充推导）

```text
let inc : Int -> Int =
    fun (x : Int) -> Int { x + 1 }
in
    inc(3)
```

| 顺序 | 当前环境 | 结论 |
| --- | --- | --- |
| 1 | `Γ₁ = Γ₀, x:Int` | `x + 1 : Int`，可应用 `[ADD]` |
| 2 | `Γ₀` | 函数体满足返回标注，由 `[FUN]` 得函数类型 `Int -> Int` |
| 3 | `Γ₂ = Γ₀, inc:(Int -> Int)` | 初始化式满足 `inc` 的声明类型，检查主体 |
| 4 | `Γ₂` | `inc : Int -> Int`，`3 : Int` |
| 5 | `Γ₂` | 由 `[APP]` 得 `inc(3) : Int` |
| 6 | `Γ₀` | 由 `[LET]` 得整个表达式类型为 `Int` |

注意 `x` 只在函数体中可见；`inc` 只在 `let` 主体中可见。检查这段程序不要求实际调用函数或算出结果 `4`。

## 8. 类型环境复用 Resolver 的结果（p.26–29、33）

### 8.1 名字身份和类型信息分开保存

上一讲的 Resolver 已经区分：

```text
外层 x 的使用 → Symbol x#2
内层 x 的使用 → Symbol x#4
```

于是实现中的类型环境可以是：

```text
Γ : Symbol → Type

x#2 ↦ Int
x#4 ↦ String
y#3 ↦ Bool
```

这里的 `String` 来自课件的工程示例，不属于前面 SimpleType 的核心类型集合。

类型检查器通过 `NameExpr.symbol` 查 Γ，不必重新按字符串搜索作用域。两个都叫 `x` 的变量可以有不同类型，因为它们具有不同声明身份。

### 8.2 `let` 的绑定与环境扩展发生在不同 Pass

```text
Resolver 阶段：
    为 let 声明建立 Symbol x#k
    初始化式中的名字按旧作用域解析
    主体中的相关使用绑定到 x#k

Type Checker 阶段：
    检查初始化式 e₁
    Γ′ = Γ[x#k ↦ T₁]
    用 Γ′ 检查主体 e₂
```

课件 p.28 把这些动作放在同一张图中说明依赖关系。在分阶段实现里，不必让 Type Checker 每进入一个 `let` 就重新调用 Resolver。

### 8.3 Pass 的输入与输出契约

| Pass | 主要输入 | 主要输出 |
| --- | --- | --- |
| Parser | Token 序列 | AST 与源码位置 |
| Resolve | AST、作用域规则 | Symbol 与 binding |
| TypeCheck | Bound AST、类型环境 | 类型标注、类型诊断，必要时还有转换节点 |
| CustomCheck | 已有程序结构及语义信息 | API 使用约束、安全策略等检查结果 |
| Lowering | 已分析的 AST | 更低层 IR，例如 LLVM IR |

**Pass 是针对一个目标、具有输入与输出契约的一次分析或变换。** Multi-pass 按依赖关系组织这些工作，让后续阶段复用前面已经建立的信息。

**补充理解：** 一个 Pass 可以包含多种遍历，也可能生成新表示。真实语言的成员查找、重载解析与类型推断可能互相依赖，课件的线性阶段图是当前语言的清晰起点。

## 9. 从规则到递归检查代码（p.25、30）

### 9.1 规则与程序之间的对应

| 规则中的部分 | 检查器中的对应 |
| --- | --- |
| 结论的表达式形状 | AST 节点种类与分派分支 |
| 子表达式的类型前提 | 递归调用 `check(child, Γ)` |
| `Γ(x)=T` | 查询类型环境 |
| `Γ,x:T` | 扩展局部环境 |
| 同型、必须为 Bool 等约束 | 比较并报告诊断 |
| 结论中的结果类型 | 返回值，或写入节点类型表 |

总体上是“先递归检查子节点，再应用当前节点规则”。但对于 `let` 和函数，必须在递归进入特定子树之前准备对应环境，不能只做一个完全不带上下文的后序遍历。

### 9.2 SimpleType 的检查器骨架（补充实现）

下列伪代码先展示成功路径与约束位置。`requireSame` 失败会报告错误并终止当前推导；下一节再加入可继续遍历的错误恢复策略。

```text
check(e, Γ):
    match e:
        IntLiteral(_):
            return Int

        BoolLiteral(_):
            return Bool

        Name(symbol):
            return Γ.lookup(symbol)

        Add(left, right):
            a = check(left, Γ)
            b = check(right, Γ)
            requireSame(a, Int)
            requireSame(b, Int)
            return Int

        LessThan(left, right):
            a = check(left, Γ)
            b = check(right, Γ)
            requireSame(a, Int)
            requireSame(b, Int)
            return Bool

        If(condition, thenExpr, elseExpr):
            c = check(condition, Γ)
            a = check(thenExpr, Γ)
            b = check(elseExpr, Γ)
            requireSame(c, Bool)
            requireSame(a, b)
            return a

        Let(symbol, declaredType, init, body):
            actual = check(init, Γ)
            requireSame(actual, declaredType)
            local = Γ.extend(symbol, declaredType)
            return check(body, local)

        Fun(parameter, parameterType, returnType, body):
            local = Γ.extend(parameter, parameterType)
            actual = check(body, local)
            requireSame(actual, returnType)
            return FunctionType(parameterType, returnType)

        Call(callee, argument):
            f = check(callee, Γ)
            a = check(argument, Γ)
            requireFunction(f)
            requireSame(a, f.parameterType)
            return f.returnType
```

这里假定名字绑定和类型标注解析已经完成。`Γ.extend` 返回局部环境而不破坏原环境，因此离开递归调用就自然回到外层；若实现使用可变栈，则须保证正常返回与出错路径都会弹栈。

### 9.3 `sameType` 也需要定义

对于只有 `Int`、`Bool`、函数类型的 SimpleType，可以递归比较：

```text
sameType(Int, Int)   = true
sameType(Bool, Bool) = true

sameType(A1 -> R1, A2 -> R2)
    = sameType(A1, A2) and sameType(R1, R2)

其他情况 = false
```

这是当前核心语言的等价规则。后面加入名义类型、别名和转换时，不能继续假定“所有类型都逐字段比较”。

## 10. `ErrorType`：继续检查并减少级联错误（p.31–32）

### 10.1 首个错误不应掩盖独立错误

课件中的 C++ 程序：

```cpp
int f() {
    int y = unknown_value + 1;
    return "oops";
}
```

至少存在两个独立问题：`unknown_value` 未声明，以及返回值不符合函数的 `int` 返回类型。前一个错误不应使编译器跳过后面的返回检查。

### 10.2 错误信息沿前端传递

| 错误标记 | 主要职责 |
| --- | --- |
| `ErrorExpr` | Parser 保留出错或缺失的表达式位置 |
| `ErrorSymbol` | Resolver 为无法绑定的名字保留错误身份 |
| `ErrorType` | Type Checker 让失败子树仍能返回一个可识别结果 |

**`ErrorType` 是检查器内部的恢复标记，不是用户可声明的普通类型，也不证明程序合法。** 有诊断的程序仍应被视为检查失败。

课件 p.32 使用“查不到名字就插入 Error”的合并式教学代码。若采用本仓库上一讲的分阶段设计，更自然的衔接是：

```text
checkName(e, Γ):
    if e.symbol is ErrorSymbol:
        return Error          // Resolver 已报告，不重复报未定义名字
    return Γ.lookup(e.symbol)
```

**补充实现：** 正常 Symbol 缺少类型登记，可能反映类型收集顺序或编译器实现的问题，不能一律重新解释成源程序“未定义名字”。错误去重也应依据作用域、声明身份和诊断来源，不能用一个全局字符串表压掉所有同名错误。

### 10.3 先访问兄弟子树，再传播错误

```text
checkAdd(e, Γ):
    a = check(e.left, Γ)
    b = check(e.right, Γ)       // 即使左侧错误，仍检查右侧

    if a == Error or b == Error:
        return Error

    if not sameType(a, Int) or not sameType(b, Int):
        report(e.span, "加法要求两个 Int 操作数")
        return Error

    return Int
```

例如：

```text
missing + false(42)
```

- `missing` 的名字错误已经记录，类型检查返回 `Error`。
- 右侧仍被访问，因此可以报告“Bool 值不可调用”。
- 加法节点收到错误结果后返回 `Error`，不再追加一条没有帮助的操作数类型诊断。

如果左侧得到 `Error` 后立即返回，就会漏掉右侧的独立错误。

### 10.4 保留仍然有效的局部检查（补充实现）

对于 `if`，可以先检查三个子树，再分别验证仍有充分类型信息的约束：

```text
checkIf(e, Γ):
    c = check(e.condition, Γ)
    a = check(e.thenExpr, Γ)
    b = check(e.elseExpr, Γ)
    failed = (c == Error or a == Error or b == Error)

    if c != Error and not sameType(c, Bool):
        report(e.condition.span, "条件必须是 Bool")
        failed = true

    if a != Error and b != Error and not sameType(a, b):
        report(e.span, "两个分支类型不一致")
        failed = true

    return Error if failed else a
```

例如条件有名字错误，但两个分支分别为 `Int` 和 `Bool`，分支不一致仍是可以独立判断的问题。

课件采用“子树有 Error，父表达式传播 Error”的简单策略；更精细的实现也可以保留已知的声明类型或函数签名，用于后续诊断，但必须另外保留失败状态，不能把错误静默当作成功。

## 11. 用户定义类型需要另一套环境 Δ（p.35）

除了记录变量类型的 Γ，还需要记录类型定义信息的环境 Δ：

| 环境 | 课件记号 | 映射内容 | 查询示例 |
| --- | --- | --- | --- |
| 变量环境 | `Γ` / VEnv | 变量或 Symbol → Type | `x#2 → Int` |
| 类型定义环境 | `Δ` / TEnv | 类型名字或类型 Symbol → 类型信息 | `Process → 字段、关系等信息` |

Δ 中可以保存记录字段及其类型、数组元素类型、长度信息、继承关系等。哪些信息属于类型，要由具体语言规定。

```text
x ∉ dom(Γ) → 没有找到变量类型绑定
T ∉ dom(Δ) → 没有找到类型定义
```

若类型定义也具有局部作用域，Δ 同样可以按栈式作用域组织。

**补充理解：** 前面已经称 Γ 为 typing environment。这里引入 Δ 是为了区分“表达式使用的变量有什么类型”和“一个命名类型具体是什么”。实现可以采用其他名称，但不能混淆两种映射。

## 12. 递归类型：先建立身份，再补全定义（p.36–38）

### 12.1 单遍立即展开会遇到循环依赖

```text
type process = {
    pid: int,
    main: thread
}

type thread = {
    tid: int,
    owner: process
}
```

如果处理一条声明时就要求所有引用的类型已经完全定义，那么无论先处理哪一条，都可能遇到尚未处理的名字。

### 12.2 用可回填的 `NAME` 保存稳定身份

课件介绍 Tiger 风格的类型对象：

```text
INT | STRING | NIL | VOID
| NAME(symbol, target)
| RECORD(fields)
| ARRAY(elementType)
```

其中：

```text
NAME(process, target = null)
```

表示“已经知道存在名为 `process` 的类型，但具体定义尚待回填”。

| 状态 | 含义 |
| --- | --- |
| `Δ.lookup(name)` 失败 | 类型名未定义 |
| 找到 `NAME`，但 `target == null` | 类型身份已存在，定义尚未完成 |
| 找到 `NAME`，且 target 已填充 | 可以沿引用获得定义 |

`target == null` 只是构建过程中的状态，不能把它当作已经完成验证的普通类型。

### 12.3 两遍处理同一个声明组

**第一遍：为同一组所有类型建立占位对象。**

```text
P = NameTy(process, target = null)
T = NameTy(thread,  target = null)

Δ₁ = Δ₀ + {process ↦ P, thread ↦ T}
```

**第二遍：在完整的名字环境中翻译 RHS，再回填 target。**

```text
P.target = RecordTy {
    pid:  INT,
    main: T
}

T.target = RecordTy {
    tid:   INT,
    owner: P
}
```

类型表示现在是一张可能带环的图：

```text
P: NAME(process)
└── target: RECORD
    ├── pid  → INT
    └── main → T: NAME(thread)
               └── target: RECORD
                   ├── tid   → INT
                   └── owner → P
```

### 12.4 为什么必须回填同一个对象

当 `process.main` 保存了引用 `T` 后，只执行：

```text
Δ[thread] = 新建的 RecordTy(...)
```

不会自动更新 `process.main` 已经持有的旧 `T`。正确做法是补全原来的 `T.target`，让所有引用共享同一份完成后的定义。

**稳定身份解决的是引用一致性；两遍处理解决的是信息到达顺序。** 两者配合，才让相互递归的声明可以正确连接。

**补充实现：** “同一组”指语言允许相互引用的声明区域，不能无条件把整个程序的类型名都提前设为可见。第一遍还要检查该作用域的重复声明，第二遍仍要报告真正不存在的类型名。

## 13. 纯别名环：两遍完成后还需要验证（p.39–40）

### 13.1 所有名字都存在，也可能没有有效定义

```text
type A = B
type B = A
```

两遍处理可以顺利建立：

```text
NA.target = NB
NB.target = NA
```

查名字没有失败，但沿别名链永远得不到具体类型构造器。

### 13.2 只沿 `NAME.target` 检查环

课件的算法是：

```text
checkAliasCycle(start):
    seen = {}
    t = start

    while t is NameTy:
        if t in seen:
            reject("纯别名环")
        seen.add(t)
        t = t.target

    accept
```

检查 `A → B → A`：

| 步骤 | 当前 t | seen | 动作 |
| --- | --- | --- | --- |
| 0 | NA | `{}` | 加入 NA，转到 NB |
| 1 | NB | `{NA}` | 加入 NB，转到 NA |
| 2 | NA | `{NA, NB}` | 再次遇到同一对象，拒绝 |

`seen` 比较的是对象身份，不能只比较显示出来的名字字符串。

### 13.3 为什么遇到 RECORD 就停止

| 类型图 | 结果 | 原因 |
| --- | --- | --- |
| `NA → NB → INT` | 合法别名链 | 最终到达基本类型 |
| `NA → RECORD(... NA ...)` | 通过此项检查 | 已经到达具体类型构造器 |
| `NA → NB → NA` | 非法纯别名环 | 永远只在 NAME 之间循环 |
| `NA → NA` | 非法纯别名环 | 自引用也是 NAME 环 |

此检查只拒绝纯别名循环，不是拒绝类型图中的所有循环。若无区别地沿 RECORD 字段继续检查并拒绝所有环，就会误伤课件希望支持的递归记录类型。

**补充实现：** 应在本组所有 `target` 回填后，再从各个新声明检查。若此前翻译 RHS 失败，遇到 `null` 或错误标记时应保留失败状态，不能把“不是 NAME”直接当成成功。合法递归类型是否还能用于具体布局，则取决于语言的表示规则，见下一节。

## 14. C++ 不完整类型：身份与对象布局（p.41–44）

### 14.1 前向声明让类型先获得身份

```cpp
struct Process;
struct Thread;

struct Thread {
    int tid;
    Process* owner;
};

struct Process {
    int pid;
    Thread* main;
};
```

前向声明之后，编译器已经知道 `Process`、`Thread` 是类型名。后续定义补全同一类型，而不是创建另一种恰好同名的类型。参见 [C++ 工作草案：Types](https://eel.is/c++draft/basic.types#general)。

### 14.2 能否使用，取决于当前位置需要什么信息

| 使用方式 | 是否需要所指类 T 已完整定义 | 原因 |
| --- | --- | --- |
| 声明 `T* p`、`T& r` 这样的类型用法 | 通常不需要 | 只需知道类型身份；实际引用变量仍须满足初始化等规则 |
| 类中声明 `T* p` 成员 | 不需要 | 指针的表示不依赖 T 的成员布局 |
| 类中声明 `T value` 按值成员 | 需要 | 包含它的对象需要知道 T 的大小与对齐 |
| `sizeof(T)` | 需要 | 必须知道完整对象大小 |
| 对 T 对象进行成员访问、以 T 为基类、`new T` | 需要 | 需要相应定义及布局信息 |

这里讨论普通类类型及课件中的基本用法；“可以写某种指针声明”不表示对该指针的所有操作都合法。

```cpp
struct Process;

struct Thread {
    int tid;
    Process owner;  // 错误：按值成员的类型还不完整
};
```

### 14.3 为什么不能留给普通链接过程补全

要生成 Thread 对象的存储与访问代码，编译器需要确定：

- `Process` 的大小与对齐，以及它作为成员对 Thread 布局的影响。
- `Thread::owner` 的偏移、`sizeof(Thread)` 和数组元素步长。
- 栈空间分配与字段地址计算等指令所需的常量。

课件强调独立编译与 ABI 的约束：这些布局决定必须在使用它们生成代码时可用。普通链接器处理符号地址与重定位，不会替前端重新完成类定义、类型检查及所有依赖布局的代码生成。

**补充理解：** 即使通过前向登记解决了名字查找，让两个类无限地按值包含彼此仍无法形成有限对象布局。通过指针等间接表示连接对象，才能使每个对象本身的大小可确定。

### 14.4 跨文件使用不必到处包含完整定义

```cpp
// thread.h
#pragma once
struct Process;
struct Thread {
    int tid;
    Process* owner;
};
```

包含 `thread.h` 的文件已经可以定义 `Thread t{1, nullptr};`，因为 Thread 本身的布局完整。只有当代码需要定义 Process 对象或访问其成员时，才需要引入 Process 的完整定义。

## 15. 类型等价、结构兼容与名义身份（p.45–46）

### 15.1 布局相同能否当作同一种类型

课件用下面的语言示意提出问题：

```text
type User  = { id: Int }
type Order = { id: Int }

fun cancel(x: Order) = ...
let u: User = ...
cancel(u)
```

| 规则 | 判断依据 | 在相应规则下的结果 |
| --- | --- | --- |
| 结构等价 | 递归比较语言规定的结构组成 | 两种记录结构相同，可判为等价 |
| 名义等价 | 比较类型声明引入的身份 | 两次独立记录定义可以产生不同类型，拒绝混用 |

**结构比较也不等于只比较字节数。** 字段名字、字段类型、顺序等是否参与，取决于该语言对相应构造器的定义。

### 15.2 结构兼容支持按需求组合

课件的 TypeScript 例子：

```typescript
type Point = { x: number; y: number };
function draw(p: Point) {}

const mouse = { x: 10, y: 20, button: 1 };
draw(mouse); // OK
```

`mouse` 提供了所需的 `x` 和 `y`，所以可以作为参数。这个例子说明的是**赋值／参数兼容性**，并不要求两个类型具有完全相同的字段。TypeScript 对直接出现的对象字面量还可能进行额外属性检查，不能把这个例子推广为任何额外字段写法都允许。见 [TypeScript：Type Compatibility](https://www.typescriptlang.org/docs/handbook/type-compatibility)。

### 15.3 名义身份保留领域区别

课件的 Rust 例子：

```rust
struct UserId(u64);
struct OrderId(u64);
fn cancel(id: OrderId) {}

// 在函数体中：
cancel(UserId(7)); // 类型不匹配
```

两个定义都包装 `u64`，但声明身份不同，因此可以阻止用户编号和订单编号被混用。这里的示例展示类型关系，省略了完整程序的外层结构。

在课件的 Tiger 模型中，两个独立 RECORD 构造创建不同身份，而别名：

```text
type Purchase = Order
```

引用已有类型，不创建一个新的记录身份。判断等价时应沿透明别名找到其所指类型。

### 15.4 Go 不能整体归为“结构等价”（补充澄清）

课件 p.45 将 Go 列在结构等价一侧。更准确的理解是：Go 对不同类型构造使用不同规则，独立定义的命名类型具有不同身份。

```go
type User struct{ ID int }
type Order struct{ ID int }
type Purchase = Order
```

- `User` 与 `Order` 是不同类型，不能仅凭字段相同就当作同一类型传参。
- `Purchase` 是 `Order` 的别名，二者指向同一类型。
- 未命名复合类型有结构化的同一性规则；接口满足关系也需要与命名类型身份分开讨论。

参见 [Go 规范：Type identity](https://go.dev/ref/spec#Type_identity) 和 [Alias declarations](https://go.dev/ref/spec#Alias_declarations)。

### 15.5 三种关系必须区分

| 关系 | 回答的问题 | 重要性质 |
| --- | --- | --- |
| 类型等价 `sameType(S,T)` | S 与 T 是否被语言视为同一类型 | 等价关系通常具有对称性 |
| 赋值／参数兼容 | S 的值能否出现在要求 T 的位置 | 有方向，可能涉及子类型或允许的转换 |
| 转换关系 | 是否存在语言允许的 S → T 转换 | 有方向，并且必须规定转换语义 |

前面的 SimpleType 使用严格同型规则。真实语言加入结构子类型或转换后，这三种判断就不能共用一个简单布尔比较。

## 16. 显式转换：允许转换还要知道怎样转换（p.47–50）

### 16.1 三种层次

```text
程序员自定义函数： intToFloat(3)
语言原生显式转换： 3 as Float
语言原生隐式转换： let rate: Float = 3
```

第一种在类型检查器看来只是普通函数调用。第二种需要语言定义 CastExpr、合法转换规则、转换失败或精度损失的行为，以及如何 lowering。

`Int` 和 `Float` 不因此变成同一种类型：

```text
sameType(Int, Float)                 = false
canConvert(Int, Float, Explicit)     = true  // 在允许此转换的语言中
```

### 16.2 显式转换规则

$$
\frac{\Gamma \vdash e:S\qquad
      \operatorname{canConvert}(S,T,\mathrm{Explicit})}
     {\Gamma \vdash e\ \mathrm{as}\ T:T}
\;[\mathrm{CAST}]
$$

源类型 `S` 来自子表达式，目标类型 `T` 来自显式语法。允许转换后，整个表达式的类型为 `T`。

### 16.3 转换判断应产生实施方案

| 规则类别 | 判断依据 | 后续操作 |
| --- | --- | --- |
| 同型转换 | `sameType(S,T)` | 无需生成改变值的转换代码 |
| 内建数值转换 | 数值类型与当前模式允许 | 插入 NumericCast，生成具体数值转换指令 |
| 用户定义转换 | 查到合适的转换函数 f | 将转换落实为 `f(e)` |
| 受检引用转换 | 允许带运行时检查的引用转换 | 插入运行时检查，并按语言规则处理失败 |
| 无可用规则 | 所有适用性检查失败 | 报告非法转换 |

**补充实现：** 比单纯返回 `true` 更有用的接口是：

```text
findConversion(S, T, mode)
    → ConversionPlan | NoConversion | Ambiguous
```

`ConversionPlan` 可以记录转换种类、需要调用的函数、是否生成运行时检查，以及多步转换的顺序。语义阶段确定这些信息，后续阶段才能生成正确代码。

### 16.4 从 `static_cast` 到 LLVM IR

课件示例：

```cpp
int x = input();
double y = static_cast<double>(x);
```

```text
显式语法：static_cast<double>(x)
    ↓ 根据源类型 int 和目标类型 double 选择规则
Clang AST：CXXStaticCastExpr，转换种类 CK_IntegralToFloating
    ↓ lowering
LLVM IR：%v = sitofp i32 %x to double
```

这里按课件假设目标平台的 `int` 对应 `i32`、`double` 对应 LLVM `double`。`static_cast` 这一个语法形式可触发不同转换，不能把所有 `static_cast` 都生成同一条 IR 指令。

Clang 用 [CXXStaticCastExpr](https://clang.llvm.org/doxygen/classclang_1_1CXXStaticCastExpr.html) 表示这种表达式；LLVM 的 [`sitofp`](https://llvm.org/docs/LangRef.html#sitofp-to-instruction) 把输入按有符号整数解释并转换为浮点数。课件还用 Java 的 `i2d` 和 .NET 的 `conv.r8` 说明相同思想可以落到不同执行平台。

**补充理解：** 示例用运行时取得的 `x`，便于观察转换操作；若直接写常量 `3`，优化可能把它折叠成浮点常量，而不保留显式转换指令。

## 17. 隐式转换：由上下文给出目标类型（p.51）

```text
let rate: Float = 3    // 声明类型给出期望 Float
area(2)               // 形参类型给出期望类型
1 + 2.0               // 运算符规则决定共同的操作类型
```

检查流程是：

```text
1. 检查表达式 e，得到实际类型 S
2. 从上下文获得期望类型 T
3. findConversion(S, T, Implicit)
4. 成功后，自动插入对应的转换节点
```

省略的是源代码中的转换写法，检查规则和实际转换操作仍然存在。

课件采用的设计原则是：

$$
\mathrm{ImplicitEdges} \subseteq \mathrm{ExplicitEdges}
$$

即可以自动使用的转换，应当比需要程序员显式请求的转换更受限制。课件建议优先选择不会丢失信息、不会意外失败、不会带来意外语义或运行开销的转换。

**补充理解：** 这是本讲的设计取向，不能当作所有现有语言对隐式转换的共同保证。是否损失精度还依赖具体数值范围与目标格式，不能只凭 `Int → Float` 两个名字判定。本讲的图模型也不等同于 C++ 等语言完整的重载和转换序列规则。

## 18. 转换图：路径选择必须由语言规定（p.52–53）

### 18.1 Byte 到 Float 可能有两条路径

课件例子假设：`Byte` 是无符号 8 位整数，`Int` 对应有符号 32 位整数，`Float` 对应 LLVM `double`。

```text
let b: Byte = 200
let x: Float = b
```

```text
             直接：uitofp
     Byte ──────────────────→ Float
       │                        ▲
       │ zext                   │ sitofp
       └────────→ Int ──────────┘
```

| 路径 | 对应 LLVM IR | 含义 |
| --- | --- | --- |
| `Byte → Float` | `%x = uitofp i8 %b to double` | 按无符号整数解释，直接转浮点 |
| `Byte → Int → Float` | `%w = zext i8 %b to i32`，再 `%x = sitofp i32 %w to double` | 先零扩展，再把非负 i32 转浮点 |

`i8` 本身不附带“有符号／无符号”的源语言类型属性，解释方式体现在操作中。这里若错误地直接使用 `sitofp i8 %b`，位模式表示的 `200` 会按有符号 i8 解释为 `-56`。参见 LLVM 的 [`uitofp`](https://llvm.org/docs/LangRef.html#uitofp-to-instruction) 和 [`sitofp`](https://llvm.org/docs/LangRef.html#sitofp-to-instruction) 定义。

### 18.2 节点是类型，边是带规则的转换

一条边至少需要说明：

```text
源类型、目标类型
允许的模式：Explicit / Implicit
命中后的动作：转换节点、函数调用或运行时检查
候选比较时使用的优先级
```

一条路径就是按顺序执行的一组转换规则。

```text
findConversion(S, T, mode):
    只考虑当前 mode 允许的边及语言允许的组合
    收集从 S 到 T 的候选路径
    按语言规定的优先级比较候选
    若没有候选，报告无法转换
    若没有唯一最优候选，报告歧义
    返回唯一最优路径对应的 ConversionPlan
```

课件示例采用“直达路径优先于多步链”，因此选 `Byte → Float`。这是一项语言规则，不是只要使用图搜索就自然得到的唯一答案。

**补充实现：** 不能任意把所有转换边做传递闭包。有些语言限制转换次数或转换组合；多条路径还可能在舍入、溢出、副作用、运行时检查等方面不同。最短路径只有在语言明确把它规定为优先级时才适用。

## 19. 设计检查表与易错点（p.54）

为一个新构造设计类型检查时，可以依次明确：

| 设计问题 | 本讲中的具体例子 |
| --- | --- |
| 局部规则的前提与结论是什么 | `[IF]` 的条件、分支类型与结果类型 |
| 环境何时扩展、何时退出 | `[LET]` 在旧环境检查初始化式 |
| 哪些信息必须先登记 | 递归类型先登记全部 NAME 身份 |
| 哪些环应允许，哪些应拒绝 | 允许经 RECORD 的递归，拒绝纯 NAME 环 |
| 类型等价依据是什么 | 结构、声明身份、透明别名 |
| 兼容与转换如何确定 | 模式、ConversionPlan、路径优先级 |
| 出错后如何继续 | ErrorType 传播、兄弟子树继续检查 |

易错点速查：

| 易错点 | 正确理解 |
| --- | --- |
| Parser 接受就代表程序语义合法 | Parser 只完成语法结构检查 |
| `Γ ⊢ e:T` 表示 e 运行后得到 T | T 是静态类型，不是运行结果 |
| 有类型标注就无需检查初始化式或函数体 | 标注是必须验证的约定 |
| `if` 的类型就是条件的类型 | 条件为 Bool，整个表达式取共同分支类型 |
| `let` 的类型就是声明变量的类型 | 它取主体表达式的类型 |
| `[LET]` 中的新变量在初始化式中已可见 | 本讲使用非递归 let，初始化式在旧环境检查 |
| 类型检查必须重新按字符串解析名字 | 可直接复用 Resolver 建立的 Symbol binding |
| 子节点错误后立即返回即可 | 会漏掉兄弟子树中的独立错误 |
| ErrorType 可以当作普通类型使程序通过 | 它只服务于恢复，失败诊断仍须保留 |
| NAME 的 target 为空说明名字未声明 | 名字可以已经登记，只是定义尚未回填 |
| 两遍回填完成就排除了所有非法类型 | 还要检查纯别名环等约束 |
| 递归类型图中的环都应被拒绝 | 纯别名环与经过类型构造器的递归不同 |
| 用新对象替换环境条目就能更新所有引用 | 已保存的旧引用不会自动改变，应回填稳定对象 |
| 名字已知就能按值存储不完整类 | 对象布局仍需完整定义 |
| 相同布局必然意味着同一类型 | 名义类型按声明身份区分 |
| 结构兼容意味着字段集合完全相同 | 兼容规则可能只要求具有目标所需成员 |
| 允许 S 转为 T 就意味着 S 与 T 等价 | 转换有方向，并有具体语义 |
| 隐式转换不需要生成转换操作 | 只是源码省略写法，编译器仍须插入操作 |
| 转换图任取一条可达路径就行 | 必须按语言规则选择唯一最优路径或报告歧义 |
| 所有整数转浮点都用 sitofp | 必须区分有符号与无符号解释 |

## 20. 复习自测与简答

1. **在 `Γ={x:Int}` 下，`if x < 0 then 0 else x + 1` 是什么类型？**  
   条件由 `[LT]` 得 Bool，两分支都是 Int，由 `[IF]` 得整个表达式为 Int。

2. **`if true then 1 else false` 能否通过本讲规则？**  
   不能。即使条件是常量，两个分支仍须具有相同类型，这里分别为 Int 和 Bool。

3. **`let x:Int = 1 in x < 2` 的类型是什么？**  
   Bool。`let` 的类型由主体决定。

4. **空环境下，`let x:Int = x + 1 in x` 错在哪里？**  
   初始化式中的 x 不在旧环境中；当前规则没有递归绑定。

5. **`fun(x:Int)->Bool{x < 0}` 的类型是什么？若把返回标注改为 Int 呢？**  
   原式为 `Int -> Bool`。改为 Int 后，函数体类型与声明返回类型不符。

6. **若 `f : Int -> Bool`，`f(3)` 与 `f(true)` 分别怎样？**  
   `f(3) : Bool`；`f(true)` 因实参类型不匹配被拒绝。

7. **若 `f : Int -> (Int -> Bool)`，`f(1)(2)` 的类型是什么？**  
   第一次调用得 `Int -> Bool`，第二次调用得 Bool；这需要两次应用 `[APP]`。

8. **为什么 Γ 的键适合使用 Symbol？**  
   Symbol 区分同名的不同声明，类型检查可以复用稳定 binding，无须重算遮蔽关系。

9. **检查 `missing + false(42)` 时，为什么不能看到左侧 Error 就返回？**  
   右侧还有独立的“Bool 不可调用”错误，需要继续递归检查。

10. **为什么递归类型需要 NAME 占位对象？**  
    它允许在定义未完成时保存稳定引用，随后通过回填同一个对象补全所有使用者看到的信息。

11. **`A → B → INT`、`A → B → A`、`A → RECORD(next:A)` 哪些通过纯别名环检查？**  
    第一种和第三种通过，因为能到达非 NAME 构造器；第二种重复访问 NAME 对象而被拒绝。

12. **有 `struct T;` 后，为什么 `T* p` 的类型可用，而 `sizeof(T)` 不可用？**  
    前者只需要类型身份及指针表示，后者需要 T 的完整对象布局。

13. **`type Purchase = Order` 在课件的透明别名模型中创建新记录身份吗？**  
    不创建。它引用已有类型，区别于重新写一个独立 RECORD 定义。

14. **`sameType(Int,Float)=false` 是否意味着禁止所有转换？**  
    不是。语言可以另行允许显式或隐式转换，并规定转换的具体语义。

15. **为什么转换查询最好返回 ConversionPlan？**  
    后续阶段需要知道生成数值转换、调用函数还是插入运行时检查，单个布尔值不包含这些信息。

16. **课件中 Byte 值 200 为什么不能直接使用 `sitofp i8`？**  
    它会把该位模式按有符号 i8 解释为 -56。应使用 `uitofp`，或先零扩展到能表示 200 的 Int 再转浮点。

17. **存在两条合法隐式转换路径时怎么办？**  
    按语言规定比较候选，选择唯一最优路径；没有路径或最优候选并列时报告错误。
