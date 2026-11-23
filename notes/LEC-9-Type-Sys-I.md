# LEC 9 · 类型检查、类型定义与转换

SE3355 编译原理 · 2026 秋　[本讲课件（54 页）](../lectures/LEC-9-Type-System-I.pptx)

`1 + true` 能解析成一棵加法树，到类型检查时才出错，因为课程语言的加法要求两个整数。类型检查器沿 AST 检查每条规则的前提，把推得的类型附回节点。名字解析解决“这个 `x` 是谁”，类型检查才问“这个 `x` 能做什么”。

## 类型也能检查接口（p.2–10）

静态检查根据代码和声明在执行前判断类型，动态检查根据执行时的值判断。两者可同时存在。例如 Java 允许把静态类型为 `Object` 的引用显式转成 `String`，若实际对象与目标类型不兼容，执行转换时会抛出异常。若引用为 `null`，转换成功，结果仍为 `null`。[Java 受检引用转换](https://docs.oracle.com/javase/specs/jls/se25/html/jls-5.html#jls-5.1.6.3)

课件里的 RT 用正则语言描述 Shell 命令允许接收、可能输出的字节流。`find . | grep -E 'book[0-9]+[.]txt' | xargs cat` 看似接得通，但路径 `./ book0.txt` 含空格，默认的 `xargs` 会把它拆成两个参数。若要保证管道兼容，上游可能输出的每一行都必须是下游允许的输入，即 `L(out) ⊆ L(in)`。也可以检查 `L(out) ∩ complement(L(in))` 是否为空。上述路径就在这个差集里。[RT 论文](https://atlas.cs.brown.edu/pdf/rt:osdi:2026.pdf)

## SimpleType 的判断规则（p.12–23）

`Γ ⊢ e : T` 读作“在类型环境 Γ 中，表达式 e 有类型 T”。Γ 记录当前可见绑定的类型。推导树的子节点给出前提，前提全成立，才可在父节点写出结论。这里的类型只有 `Int`、`Bool` 和函数类型 `A → B`，运算符不做隐式转换。

| 表达式 | 检查什么 | 得到的类型 |
| --- | --- | --- |
| 整数、`true` 或 `false` | 字面量直接给出类型 | `Int` 或 `Bool` |
| 变量 `x` | 查询已绑定声明在 Γ 中的类型 | `Γ(x)` |
| `a + b`、`a < b` | 两个操作数都是 `Int` | 分别为 `Int`、`Bool` |
| `if c then a else b` | c 为 `Bool`，两个分支同型 T | T |
| `let x:T = init in body` | 在旧 Γ 中检查 init 得 T，再用 `Γ[x↦T]` 检查 body | body 的类型 |
| `fun (x:A) -> B { body }` | 在 `Γ[x↦A]` 中检查 body 得 B | `A → B` |
| `f(a)` | f 的类型为 `A → B`，a 的类型为 A | B |

写成推理规则时，横线上面是前提，下面是结论。`[LET]` 的两个前提用到的环境不同，直接决定了新名字从哪里开始可见。

```text
Γ ⊢ c : Bool       Γ ⊢ a : T       Γ ⊢ b : T
──────────────────────────────────────────── [IF]
          Γ ⊢ if c then a else b : T

Γ ⊢ init : T       Γ[x ↦ T] ⊢ body : U
──────────────────────────────────────────── [LET]
     Γ ⊢ let x:T = init in body : U

Γ ⊢ f : A → B      Γ ⊢ arg : A
─────────────────────────────── [APP]
         Γ ⊢ f(arg) : B
```

**手算条件式。** 给定 `Γ = {x:Int}`，要检查 `if x < 0 then 0 else x + 1`，先分别为条件和两条分支推导类型。

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

条件是 `Bool`，两个分支都是 `Int`，整棵 `if` 的类型为 `Int`。若把 then 分支改成 `true`，条件检查仍通过，但分支类型成了 `Bool` 和 `Int`，无法应用 `[IF]`。

**再加一层绑定。** 在下面的表达式里，`x` 只在函数体可见，`inc` 只在 `in` 后的主体可见。留意检查初始化式时，环境里还没有 inc。

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

## 规则怎样变成检查器（p.25–33）

对 `check(e,Γ)` 按节点种类分支，递归检查孩子，再核对上表的前提。名字节点已经保存 Symbol，所以 Γ 可以用声明身份作键。扩展环境只影响对应的子树。函数体检查完毕后，参数不会漏进外层。比较函数类型时要分别比较参数和结果。

加法的检查器先访问左右两棵子树，即使一边出错，另一边的独立错误也能查出来。

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

检查 `missing + false(42)` 会发现两个独立错误。左边名字未声明，右边把布尔值当函数调用。即使左边已经产生 `ErrorType`，也要检查右边。父节点可以沿用错误标记，避免再报一次缺乏依据的加法错误。语法缺口、名字查找失败和类型检查失败分别由 `ErrorExpr`、`ErrorSymbol` 和 `ErrorType` 表示。

## 递归类型怎样登记（p.35–44）

值的类型从 Γ 查，类型名的定义从另一张表 Δ 查。下面的 Tiger 声明组定义了进程和线程。读到进程的 `main` 字段时，后面的线程类型还没有展开。

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

1. 第一遍只登记名字。在 Δ 中放入 `P = NameTy(process, 未填)`、`T = NameTy(thread, 未填)`，让两者先取得稳定身份。
2. 第二遍读取字段，得到 `P.target = Record{pid:Int, main:T}` 与 `T.target = Record{tid:Int, owner:P}`。必须回填原来的占位对象，先前存下的引用才能看到补全的定义。
3. 从每个名字沿 `NAME.target` 追踪。到基本类型、记录或数组等真正的构造时结束。一路只经过名字却第二次见到同一对象，说明是纯别名环，例如 `type A = B; type B = A`。查不到目标名字则另报未定义。

判断别名环时只沿名字链走。在记录字段里重新遇见原名，可以表示合法递归。具体的内存布局还取决于语言如何表示记录。C++ 中 `struct Process;` 足以声明 `Process* p`，因为指针大小已知。按值嵌入 `Process`、取 `sizeof(Process)` 或访问其成员，则需要完整定义。把两个类彼此按值嵌入会使大小无法确定，改用指针可截断这种布局依赖。[C++ 不完整类型](https://eel.is/c++draft/basic.types#general)

## 同一个类型、能当参数、能转换（p.45–53）

| 问题 | 判断依据与例子 |
| --- | --- |
| 两个类型是否相同？ | 名义类型看声明身份。两个分别定义的 Rust 包装类型 `UserId(u64)`、`OrderId(u64)` 即使布局相同，也不能直接互换。结构类型按语言规定的字段、组成类型等比较。 |
| S 的值能否传给需要 T 的函数？ | 方向是 S 到 T。TypeScript 中，一个已有变量若提供 `Point` 需要的 `x`、`y`，即使还有其他字段，也可能传给接收 `Point` 的函数。兼容并不要求字段完全相同。 |
| S 的值怎样转成 T？ | 先查语言允许的规则，再保留转换步骤。结果可能是数值转换、调用用户函数或带运行时检查的引用转换。 |

Go 的 `type Purchase = Order` 给已有类型起别名。`type User struct{ID int}` 和 `type Order struct{ID int}` 各自定义新的命名类型。前者共享身份，后两者即使字段相同仍是不同类型。[Go 类型同一性](https://go.dev/ref/spec#Type_identity) · [Go 别名](https://go.dev/ref/spec#Alias_declarations) · [TypeScript 类型兼容性](https://www.typescriptlang.org/docs/handbook/type-compatibility)

显式的 `static_cast<double>(x)` 写明了目标。若 `x` 是有符号 32 位整数，前端选中整数到浮点的转换，并可生成 `sitofp i32 %x to double`。隐式转换由上下文给出目标，例如 `let rate: Float = 3`。类型检查器也要把选中的转换记进 AST 或 IR，后端才知道该发什么指令。[Clang 转换节点](https://clang.llvm.org/doxygen/classclang_1_1CXXStaticCastExpr.html) · [LLVM 的 sitofp](https://llvm.org/docs/LangRef.html#sitofp-to-instruction)

**验算转换路径。** 课件假定 `Byte` 为无符号 8 位，`Int` 为有符号 32 位，`Float` 用 `double` 表示。对于 `Byte(200)`，下表中的两条路径都会保留它的数值。

| 路径 | 对应 LLVM IR | 含义 |
| --- | --- | --- |
| `Byte → Float` | `%x = uitofp i8 %b to double` | 按无符号整数解释，直接转浮点 |
| `Byte → Int → Float` | `%w = zext i8 %b to i32`，再 `%x = sitofp i32 %w to double` | 先零扩展，再把非负 i32 转浮点 |

两条路径都得 `200.0`。若直接用 `sitofp i8`，同一位模式会按有符号数解释，得到 `−56.0`。所以转换图的每条边都要标明适用条件和具体操作。若有多条可行路径，由语言规定优先级或报告歧义，不能随遍历顺序挑选。[LLVM 的 uitofp](https://llvm.org/docs/LangRef.html#uitofp-to-instruction)

**自测。** `let x:Int = x + 1 in x < 10` 中，两处读取 x 分别指向哪个声明？按本讲的非递归 `let`，初始化式里的 `x` 查外层 Γ，找不到就报未声明。主体里的 `x` 绑定到新声明。若外层 `x:Int` 存在，整个表达式的类型是 `Bool`。省略标注时，需要根据使用方式反推类型。[第 10 讲 · 类型推导与合一](LEC-10-Type-Sys-II.md)
