# LEC 11 · 从 AST 生成三地址代码与控制流图

SE3355 编译原理 · 2026 秋　[本讲课件（45 页）](../lectures/LEC-11-IR-I.pptx)

读 `return (x + 1) * (x - 2);` 时，AST 能告诉我们乘法的两个孩子是谁。要生成可执行代码，还要决定先求哪一项、把结果放在哪里，以及哪些指令只在某个分支上执行。本讲先用**三地址指令**记录计算，用**基本块和跳转**记录控制流。

## AST 到机器指令之间差了什么（p.4–18）

```c
int adjust(int x) {
    int y = x + 1;
    if (y < 10)
        return y;
    return y * 2;
}
```

`adjust` 的名字和类型已经查清。条件为真就返回 `y`，为假才执行最后的乘法。课件把这个函数译成了几条 x86-64 指令。

```asm
adjust:
    lea eax, [rdi + 1]
    cmp eax, 10
    jl  .return
    add eax, eax
.return:
    ret
```

`lea` 算出 `x + 1`，`cmp` 和 `jl` 一起决定是否跳过加倍操作。这里已经选定指令、寄存器和标签。若把这些决定全放进 AST 遍历，换一门源语言或换一个目标机，都要重新处理许多相同问题。共享一套中间表示（IR）后，N 门语言各自提供前端，M 种机器各自提供后端，中间的公共优化只需实现一次。这样可以按 N + M 个接口组织工作。各前端仍要保留源语言语义，各后端仍要遵守目标机器的约定。

| 形态 | 主要显式表达的信息 | 常见用途 | 课件中的系统示例 |
| --- | --- | --- | --- |
| Tree／高层 IR | 语言构造及嵌套关系 | 去糖、类型检查等 | Rust HIR／THIR，GCC GENERIC |
| 三地址指令 + CFG | 计算步骤与控制转移 | 数据流分析、优化 | LLVM IR，GCC GIMPLE，Rust MIR |
| Node Graph | 定义—使用、数据依赖等关系 | 图上的优化 | HotSpot C2 Ideal Graph，GraalVM IR |
| Stack／Bytecode | 栈式操作或紧凑执行指令 | 验证、解释、分发 | Java bytecode，WebAssembly，CPython bytecode |

不同编译阶段（pass）约定好输入、输出和必须保持的性质。例如，优化器可以假定基本块结尾写明了控制流，后端可以等到指令选择以后再为临时值分配物理寄存器。[GCC 的 GIMPLE 表示](https://gcc.gnu.org/onlinedocs/gccint/GIMPLE.html)

## 表达式翻译要做两件事（p.19–29）

翻译（lowering）一个表达式时，把计算它的指令追加到当前块，并向调用方**返回代表结果的 Value**。编译 `x + 1` 时通常还不知道 x 的数值，返回的是临时结果的引用。例如，按从左到右的顺序翻译 `(x + 1) * (x - 2)` 会得到三个临时值。

```text
t0 = add x, 1
t1 = sub x, 2
t2 = mul t0, t1

result = t2
```

这种形式叫三地址代码，典型形状是 `result = op arg1, arg2`，也可存成四元组 `(op,arg1,arg2,result)`。名字代表中间结果，之后的每条指令都可以引用它。取负、调用、跳转和返回所需的操作数数量各不相同。

```text
lowerExpr(Int(n)):
    return Const(n)

lowerExpr(Var(symbol)):
    return valueOf(symbol)

lowerExpr(Binary(op, left, right)):
    a = lowerExpr(left)
    b = lowerExpr(right)
    return emit(op, a, b)
```

`valueOf` 表示读取变量。下一讲会区分直接引用一个 SSA 值和从内存取值。上述递归只适用于按该顺序求值的运算。**写在左侧的参数一定传给左边的形参，但不一定先执行。** 在 C 的 `use(f(),g())` 中，f 和 g 的执行次序可以不同，传给 use 的两个位置仍然固定。Haskell 的 `first x _ = x` 甚至无须为求得结果而计算第二个参数。[C11 草案第 6.5.2.2 节](https://www.open-std.org/jtc1/sc22/wg14/www/docs/n1570.pdf) · [Haskell 的非严格求值](https://www.haskell.org/onlinereport/haskell2010/haskellch6.html)

短路运算要单独翻译。例如 `p != null && p.ready` 先判断 p 是否为空，编译器把读取 `p.ready` 的指令放进非空分支。运行时若 p 为空，就跳过这次读取。

## 条件分支怎样变成基本块（p.30–41）

一个基本块从入口开始顺序执行，只有最后一条终结指令（terminator）决定随后去哪里。条件分支有两个目标，无条件跳转有一个，`return` 没有函数内后继。把基本块画成节点、可能的控制转移画成有向边，就得到了控制流图。这里的 CFG 指 control-flow graph。语法分析章节里的同一缩写指上下文无关文法（context-free grammar）。[LLVM 终结指令](https://llvm.org/docs/LangRef.html#terminator-instructions)

在 `adjust` 的教学 IR 中，入口块计算条件，两条返回路径各占一块。

```text
entry:
    y = add x, 1
    c = less y, 10
    branch c, thenBB, elseBB
thenBB:
    return y
elseBB:
    r = mul y, 2
    return r
```

两边都已返回，无需汇合。若两边只是给 x 赋值、后面还有 `return x`，则由汇合块接着执行。

```text
entry:
    c = less y, 10
    branch c, thenBB, elseBB

thenBB:
    x = 1
    branch mergeBB

elseBB:
    x = 2
    branch mergeBB

mergeBB:
    return x
```

构造器用 `currentBlock` 记录下一条指令写到哪里。先建两个分支块，翻译条件并生成条件跳转。然后分别进入两侧，递归翻译语句。每侧结束时都要**检查当前块还能否继续**。例如 then 内又有一个 if，内层分支可能已占用 then 的终结指令。外层通向汇合点的跳转应写在内层分支留下的出口。

| 两侧的出口 | 共同后继怎么接 |
| --- | --- |
| 两侧都能继续 | 两侧都跳到汇合块。 |
| 一侧已返回 | 只接另一侧。没有 else 时，条件为假可直接到汇合块。 |
| 两侧都已返回 | 后面的语句从这里不可达。若实现预建了汇合块，应清理它或单独处理不可达代码。 |

若要自己写构造器，可以约定 `lowerArm` 返回该分支还能继续执行的出口块。若分支已经返回，它就返回 `none`。外层只连接实际存在的出口。

```text
lowerIfElse(stmt):
    c = lowerExpr(stmt.cond)
    t = newBlock("then")
    f = newBlock("else")
    emit(branch c, t, f)

    tExit = lowerArm(t, stmt.thenBranch)
    fExit = lowerArm(f, stmt.elseBranch)
    exits = [b for b in (tExit, fExit) if b != none]
    if exits is empty:
        return none

    merge = newBlock("merge")
    for b in exits:
        switchTo(b)
        emit(branch merge)
    switchTo(merge)
    return merge
```

汇合块只说明两条路接到同一处。上例用的是可变名字 x。换成 SSA 名字时，还要记录每条边带来哪个 x。[φ 与基本块参数](LEC-12&13-Intermediate-Repr-II.md)

## while 的回边为什么指向条件块（p.42–45）

`while (x < 10) x = x + 1;` 要在第一次进循环前检查条件，每次更新后还要重查。把那条比较指令放在 `test` 块里，编译器只生成一次，程序每次走到该块都会执行它。

```text
entry:
    branch test

test:
    c = less x, 10
    branch c, body, exit

body:
    x = add x, 1
    branch test

exit:
    return x
```

初值为 10 时直接退出，为 8 时依次得到 9、10 再退出。若循环体已返回，它的这个出口就没有回边。对这种 while，`break` 跳到 exit，`continue` 跳到 test。每层嵌套循环各自保存这两个目标。

```text
lowerWhile(stmt):
    testBB = newBlock("test")
    bodyBB = newBlock("body")
    exitBB = newBlock("exit")
    emit(branch testBB)

    switchTo(testBB)
    c = lowerExpr(stmt.condition)
    emit(branch c, bodyBB, exitBB)

    switchTo(bodyBB)
    lowerStmt(stmt.body)
    if not isTerminated():
        emit(branch testBB)

    switchTo(exitBB)
```

把 CFG 输出成文本时，从入口遍历并记录已访问块，每个可达块只写一次。先标记再访问后继，遇回边就能停下来。只要保留标签和显式跳转，then、else 两块谁先打印都不改变执行。这里的 *flatten* 仍输出 IR，指令选择与寄存器分配是后端的工作。

**自测。** 给 `if (c) return 1; return 2;` 画图。入口的真边去 `return 1`，假边去 `return 2`，两个返回块都没有后继。如果从第一个返回块又画了一条到第二个返回块的边，就给已结束的路径添了一次跳转。
