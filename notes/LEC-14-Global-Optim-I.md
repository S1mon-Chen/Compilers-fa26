# LEC 14 · 常量传播、活跃性与死代码删除

SE3355 编译原理 · 2026 秋　[本讲课件（84 页）](../lectures/LEC-14-Global-Optimization-I.pptx)

`x = 4` 之后遇到 `y = x + 1`，可以向前传播事实“x 等于 4”，把 y 算成 5。反过来，要判断 `x = 4` 能否删除，得看它**之后**还有没有路径会读这个值。两个问题看的方向相反，也需要不同的数据流方程。

## 常量传播在汇合处怎样算（p.3–26）

**常量传播**把使用处的名字替换为已知常量，**常量折叠**在编译时依照目标 IR 的语义算出结果。代数化简也要看类型和语义，例如普通整数加零可以省去，严格浮点的 `x * 0.0` 却可能因 NaN、无穷大或零的符号而无法替换为 `+0.0`。变换要先保持语义，再看是否值得做。

下面的例子有 7 个基本块。先保留所有边，不管入口条件 `%cond` 的值是什么。

```llvm
define i32 @example(i1 %cond) {
entry:
  %a = add i32 2, 3
  br i1 %cond, label %left, label %right

left:
  %x1 = add i32 %a, 1
  br label %merge

right:
  %x2 = mul i32 2, 3
  br label %merge

merge:
  %x = phi i32 [ %x1, %left ], [ %x2, %right ]
  %same = icmp eq i32 %x, 6
  br i1 %same, label %fast, label %slow

fast:
  %r1 = add i32 %x, 0
  br label %done

slow:
  %r2 = mul i32 %x, 100
  br label %done

done:
  %r = phi i32 [ %r1, %fast ], [ %r2, %slow ]
  ret i32 %r
}
```

给每个 SSA 值记录一个事实。课件把尚未收到任何值信息记为 `⊥`，已知唯一常量记为 `Const(c)`，无法确定单一常量记为 `⊤`。字面量从自身的值开始，未知的函数参数从 `⊤` 开始，待处理的内部结果从 `⊥` 开始。这个格（lattice）的偏序是 `⊥ ≤ Const(c) ≤ ⊤`。两个不同的常量在格中不可比，例如 2 和 3 合并就得到 `⊤`。表中的 c 代表 `Const(c)`，a 代表任意一个事实。`⊥` 表示尚未收到信息，仅凭它无法判断基本块是否可达。

| 输入 | 结果 | 原因 |
| --- | --- | --- |
| `⊥ ⊔ a` | `a` | 空集不增加可能值 |
| `c ⊔ c` | `c` | 两边给出同一个常量 |
| `c₁ ⊔ c₂`，且 `c₁ ≠ c₂` | `⊤` | 本抽象域不表示多个不同常量的集合 |
| `⊤ ⊔ a` | `⊤` | 合并后仍不能确定唯一常量 |

这里的 `⊔`（join）要同时覆盖各个来源。对 `φ`，运行时按到达的边选一个值，分析时合并仍需考虑的各条边。对普通算术指令，若两个操作数都是常量，按 IR 语义折叠。有操作数尚未收到信息时可以等待更新。如果已知输入可能产生多个不同结果，就记为 `⊤`。

| 处理阶段 | `%cond` | `%a` | `%x1` | `%x2` | `%x` | `%same` |
| --- | --- | --- | --- | --- | --- | --- |
| 初始化 | `⊤` | `⊥` | `⊥` | `⊥` | `⊥` | `⊥` |
| entry | `⊤` | `5` | `⊥` | `⊥` | `⊥` | `⊥` |
| left、right | `⊤` | `5` | `6` | `6` | `⊥` | `⊥` |
| merge 的 φ | `⊤` | `5` | `6` | `6` | `6` | `⊥` |
| merge 的比较 | `⊤` | `5` | `6` | `6` | `6` | `true` |

`%x1` 和 `%x2` 都是 6，所以 `%x` 等于 6，`%same` 恒为真。此时若仍把 slow 边也算在内，`%r1` 是 6，`%r2` 是 600，`%r` 只能得到 `⊤`。**要得到最终的 `return 6`，还必须把恒真的分支用于改写 CFG。**

## 回边、固定点与工作队列（p.27–36）

```text
entry:
    br loop

loop:
    %i = phi [ 0, entry ], [ %next, body ]
    %go = icmp slt i32 %i, 3
    br %go, body, exit

body:
    %next = add i32 %i, 1
    br loop

exit:
    ret i32 %i
```

`%i` 决定 `%next`，而回边又用 `%next` 更新 `%i`。第一遍只能算出 `0 ⊔ ⊥ = 0`。这时回边还没送来 `%next` 的值，把 0 代入循环会算错。

| 阶段 | `%i` | `%go` | `%next` | 依据 |
| --- | --- | --- | --- | --- |
| 初始化 | `⊥` | `⊥` | `⊥` | 尚未传播 |
| 首次处理 φ | `0` | `⊥` | `⊥` | `0 ⊔ ⊥ = 0` |
| 继续处理比较和 body | `0` | `true` | `1` | `0 < 3`，`0 + 1` |
| 回边使 φ 重算，再传播 | `⊤` | `⊤` | `⊤` | `0 ⊔ 1 = ⊤` |
| 再次传播 | `⊤` | `⊤` | `⊤` | 不再变化，达到固定点 |

对同一个 SSA 值，这里只保留一个覆盖所有迭代的摘要。`⊤` 表示这个分析证明不了 `%i` 恒等于某个数，无法仅凭它推出退出时 i 的确切值。用单调的传递规则反复更新，直到所有结果再算一次都不变，这就是**固定点**。每个值最多变化两次，从 `⊥` 到某个常量，再到 `⊤`。函数里的定义有限，迭代会停止。

可以每轮扫描整个函数，也可以用 worklist 只重算依赖变化的节点。在 SSA 常量传播中，`affectedBy(n)` 是读取 n 结果的指令。初始队列放入所有待计算的定义。

```text
facts = initialize()
worklist = allNodes()

while worklist is not empty:
    n = worklist.pop()
    newFact = transfer(n, facts)

    if newFact != facts[n]:
        facts[n] = newFact
        worklist.addAll(affectedBy(n))
```

## 怎样把 7 个块化成 1 个（p.37–48）

1. 在各个使用处代入已经稳定的常量，再折叠计算。`%same` 变为 true，把 merge 末尾的条件分支改成直接去 fast。
2. 从 entry 沿 CFG 做可达性遍历，slow 已不可达。删除 slow 及其边，并从 done 的 φ 去掉来自 slow 的输入。现在 φ 只可能取 `%r1 = 6`。
3. 删掉已无用途且能安全移除的计算。left、right 等块只剩跳转，可以绕过空中转块。若前块只有一个后继、后块也只有这一个前驱，可合并直线块。重接边时要保留各 φ 输入和来源的对应关系。
4. 入口的两条路径最终都返回 6，整个函数只剩一个块。

```llvm
define i32 @example(i1 %cond) {
entry:
  ret i32 6
}
```

## 活跃性从后往前算（p.49–67）

从某个程序点出发，**只要存在一条路径能在下一次重定义 x 前读到当前 x**，x 在这个点就活跃。程序可能往任一后继走，因此后继的需求取并集。`IN` 指执行指令前，`OUT` 指执行后。

```text
OUT[s] = ⋃ IN[t]，其中 t 是 s 的后继
IN[s]  = USE[s] ∪ (OUT[s] − DEF[s])
```

`USE` 产生对旧值的需求，`DEF` 切断后面对旧值的需求。`x = x + 1` 同时使用旧 x、定义新 x，旧 x 在入口总是活跃，能否删掉该赋值要看**新 x 的出口活跃性**和指令本身的效果。对块使用同样的方程时，`USE[B]` 只收集在块内首次定义**以前**读到的变量。例如 `x = 1; y = x` 不要求块入口提供旧 x。

在 SSA 中要把 φ 的操作数算在**各自的入边**上。例如 `v = φ [a,L],[b,R]` 让 a 在 `L → merge` 上活跃，b 在 `R → merge` 上活跃。若把 a、b 都算作 merge 块的普通使用，就会误以为两条路都需要两个值。

```text
B1:
    x = 1
    br c, B2, B3

B2:
    y = x + 1
    br B4

B3:
    x = 2
    y = 0
    br B4

B4:
    return y
```

只追踪 x。B2 的加法要读 x，B3 会先覆盖 x，故 B2 入口为 live、B3 入口为 dead。在 B1 的分支处，只要有一个后继需要 x，它就活跃。

| 块 | 指令 s | `L_in(x,s)` | `L_out(x,s)` |
| --- | --- | --- | --- |
| B1 | `x = 1` | dead | live |
| B1 | `br c, B2, B3` | live | live |
| B2 | `y = x + 1` | live | dead |
| B2 | `br B4` | dead | dead |
| B3 | `x = 2` | dead | dead |
| B3 | `y = 0` | dead | dead |
| B3 | `br B4` | dead | dead |
| B4 | `return y` | dead | dead |

两次赋值的入口都可以是 dead，但 B1 的 `x = 1` 必须为 B2 保留。B3 的 `x = 2` 之后无人再读，如果 x 是普通局部变量，就可以删掉这次赋值。

## 循环里过早删代码会怎样（p.68–77）

```text
B1:
 s1  x = 0
 s2  sum = 0
 s3  br B2

B2:
 s4  br c, B3, B4

B3:
 s5  sum = sum + x
 s6  x = x + 1
 s7  br B2

B4:
 s8  return sum
```

按课件给定的顺序 `s8, s4, s7, s6, s5, s3, s2, s1` 扫描，所有事实从 dead 开始。第一次看 s6，它使用旧 x，故入口为 live，可出口仍暂记 dead。直到 s5 发现自己需要 x，这个需求才沿 s4、s7 的回边传回 s6 的出口。若在第一遍就删掉 s6，循环执行两轮后，原本等于 1 的 sum 会变成 0。

| 指令 | 最终 `L_in(x,s)` | 最终 `L_out(x,s)` |
| --- | --- | --- |
| s1 `x = 0` | dead | live |
| s2 `sum = 0` | live | live |
| s3 `br B2` | live | live |
| s4 `br c, B3, B4` | live | live |
| s5 `sum = sum + x` | live | live |
| s6 `x = x + 1` | live | live |
| s7 `br B2` | live | live |
| s8 `return sum` | dead | dead |

工作队列初始放入所有指令，每次重算并保存 OUT、IN。**当 IN 变化时把前驱重新入队**，因为前驱读取的是这条指令的 IN。即使这次 IN 没变，OUT 的新值仍要保存，s6 正是这种情况。对本讲的局部变量，在函数出口没有额外使用时，出口边界为 dead。若返回指令是 `return x`，它本身读取 x，所以 x 在这条指令的入口活跃。

## SSA 上怎样做死代码删除（p.78–84）

SSA 中每个操作数都对应唯一的定义。课件先保留返回、分支、store、有副作用的调用，以及其他不能安全删除的指令。再从这些指令沿操作数反向标记所有依赖的定义，最后删除未标记的纯计算。

```text
marked = {}
worklist = rootInstructions()

while worklist is not empty:
    inst = worklist.pop()
    if inst in marked:
        continue
    marked.add(inst)

    for operand in inst.operands:
        if operand is defined by an instruction:
            worklist.add(definitionOf(operand))

dead = {inst for inst in allInstructions() if inst not in marked}
for inst in dead:
    dropOperandReferences(inst)
for inst in dead:
    erase(inst)
```

例如 `x1 = 3; x2 = 4; y1 = x2; return y1` 中，反向追到 `y1`、`x2` 就结束，`x1` 可以删掉。用这种方式还可发现互相引用、对外却无用途的纯计算环。先统一断开待删指令的操作数，再统一删除，就不会因为环里还有指令引用彼此而删不下去。

这两类数据流分析的更新方向不同。常量传播看定义给出了什么值，定义的事实一变，就要通知读取它的指令。活跃性看将来会不会读到当前值，某条指令入口的需求一变，就要通知它的前驱。两者都反复更新到固定点后再改程序。

**自测。** 在 `x = y + 1` 后活跃集合是 `{x,z}`，入口是什么？先去掉这条指令定义的 x，再加上它读取的 y，得到 `{y,z}`。若算出 `{x,y,z}`，就把出口的新 x 错当成了入口也需要的旧 x。
