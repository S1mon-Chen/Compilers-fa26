# LEC 12–13 · SSA、类型、内存与 LLVM IR

SE3355 编译原理 · 2026 秋　[本讲课件（55 页）](../lectures/LEC-12&13-IR-II.pptx)

沿控制流图（CFG）能找到接下来可能执行的块。优化器还要知道这次加法用的是哪次赋值，以及两次读同一地址能否得到不同结果。静态单赋值（SSA）让每个操作数指向唯一的定义。显式的内存读写和别名分析帮助回答第二个问题。[基本块与控制流图](LEC-11-Intermediate-Repr-I.md)

## 一个值由谁定义、又被谁使用（p.3–15）

```text
a = 4
b = mul a, 2
c = sub a, 3
return b
```

从 `a` 的定义能找到乘法和减法中用它的位置，这叫 **def-use**。从乘法的操作数反查 `a = 4`，方向叫 **use-def**。`add a,a` 有两个使用位置，替换 a 时要改两处。常量传播把 `4` 送到使用处，常量折叠再算出 `4 × 2 = 8`。`c` 没人读，纯减法可以删掉。若结果来自带副作用的调用，还要保留该调用的效果。

普通三地址代码允许同一变量多次赋值。改成 **SSA 形式**时，给每次定义一个新名字，后续指令引用它实际要读的那个版本。

| 普通三地址代码 | SSA 形式 |
| --- | --- |
| `a = 4` | `a1 = 4` |
| `b = mul a, 2` | `b = mul a1, 2` |
| `a = 5` | `a2 = 5` |
| `c = add a, 1` | `c = add a2, 1` |

每个普通使用都要能追到唯一的定义，而且从入口到这个使用点的每条可达路径上，定义都已执行（定义支配使用）。同块内还要先定义后使用。循环中的指令可以动态执行多次，“单次赋值”说的是 IR 里只有一个静态定义位置。[LLVM IR 的良构条件](https://llvm.org/docs/LangRef.html#well-formedness)

## 汇合点怎样选中刚走过那条路的值（p.16–18）

then 算出 `y1 = x + 1`，else 算出 `y2 = x * 2`。在共同后继直接写 `y1`，从 else 来就没有这个值可用。在汇合点写一条 `φ` 指令，让每条前驱边各提供一个输入。

```text
merge:
  y3 = φ [y1, then], [y2, else]
  r = sub y3, 3
  return r
```

`φ` 看的是**实际进入这个块的边**，无需重新求分支条件。输入值只须在对应的边上可用。假设 x 和 y 都在两条分支上重新赋值，汇合处就要同时选出两个值。从 then 来取 `(x1,y1)`，从 else 来取 `(x2,y2)`。用块参数也能表达这种选择，参数值直接写在各条跳转上。

```text
then:
  jump merge(x1, y1)

else:
  jump merge(x2, y2)

merge(x3, y3):
  z = add x3, y3
```

循环头也是汇合点。下面第一次从 entry 来取 0，随后每次从 body 来就取上一轮的 `i_next`。

```text
entry:
  jump test

test:
  i = φ [0, entry], [i_next, body]
  c = less i, n
  branch c, body, exit

body:
  i_next = add i, 1
  jump test

exit:
  return i
```

删除 CFG 边时，也要删除对应的 φ 输入。如果只剩一个输入，就可以直接使用那个值。[LLVM φ 指令](https://llvm.org/docs/LangRef.html#phi-instruction)

## IR 需要保留哪些类型信息（p.19–33）

虎书式 Tree IR 可以靠约定解释 `MEM` 读一个机器字、`BINOP` 做整数运算。一旦既有字节又有浮点数，单写 `MUL(x,2)` 就不够了。用 `i32`、`f32`、`ptr` 标出表示，用 opcode 指定操作。比较两个 `i32` 得到 `i1`，同样的整数位串按有符号或无符号比较，结果可能相反。对无附加标志的固定宽度整数，乘二可改写为左移一位。浮点乘法有自己的舍入和特殊值规则。

类型信息保留到最后一个需要它的阶段。类、数组、闭包可以逐步变成字段布局、地址和环境，透明别名检查完后可以擦除。SSA 名字固定也不代表对象内容固定。给一个指针起了新名字，两个名字仍可指向同一块可变内存。不可变数据方便共享旧版本，若每次修改都要重建大数组，则需考虑复制或结构共享的成本。

## load、store 和别名（p.34–41）

指针标识一个位置（location），`load` 取这个位置的当前内容，`store` 改变内容。可以把内存看成一张随执行更新的表 `M[位置] = 值`。同一指针 p 的 SSA 定义只有一处。下例中地址一直没变，`v1` 读到 1，`v2` 读到 2。

```text
store.i32 1, p
v1:i32 = load.i32 p
store.i32 2, p
v2:i32 = load.i32 p
```

设现在有 `store 1,p; x = load q`。若 p、q 指向同一整数，先读后写就可能读出旧值。单看普通值的 def-use，会漏掉这条经内存传递的依赖。

| 别名查询 | 能得出什么结论 |
| --- | --- |
| NoAlias | 两个被查询的访问范围不重叠，可排除它们之间的读写冲突。 |
| MayAlias | 仍可能重叠。即使分析不够精确，也得保留这条可能的依赖。 |
| MustAlias | 两次访问总从同一位置开始。要复用或转发具体内容，还要看宽度、类型与中间的写入。 |

**指向集合** `Pt(p)` 记录 p 可能指向哪些位置。例如 `p = &x; q = p; if (c) q = &y;` 执行到汇合点时，`Pt(p) = {x}`，`Pt(q) = {x,y}`。c 为假时两者都指向 x，为真时指向不同对象，所以查得 MayAlias。若集合覆盖所有可能位置，且 x、y 这类抽象位置彼此独立，集合不相交即可排除别名。实际访问还要看宽度，两个起始地址不同的访问可以部分重叠。未知调用也可能改写内存。[LLVM 别名分析](https://llvm.org/docs/AliasAnalysis.html#the-alias-method)

## 同一个 adjust，用课程 IR 和 LLVM IR 各写一遍（p.42–50）

```c
int adjust(int x, int *out) {
    int y;
    if (x < 10)
        y = x + 1;
    else
        y = x * 2;
    *out = y;
    return y;
}
```

```text
function adjust(x:i32, out:ptr) -> i32 {
entry:
  c:i1 = less.i32 x, 10
  branch c, then, else

then:
  y1:i32 = add.i32 x, 1
  jump merge

else:
  y2:i32 = mul.i32 x, 2
  jump merge

merge:
  y3:i32 = φ.i32 [y1, then], [y2, else]
  store.i32 y3, out
  return y3
}
```

若 `out` 指向有效的可写整数，`x = 5` 会走 then，写入并返回 6。`x = 10` 会走 else，写入并返回 20。`y3` 有两个使用位置，一个给 `store`，一个给 `return`。调用者能读到 `*out`，所以即使它忽略返回值，写入也可能有用。

对应的 LLVM IR 如下。`%` 命名局部值，`@` 命名函数和全局对象。`icmp slt` 做有符号整数比较，`br i1` 跳转，`phi` 汇合结果。`ptr` 表示指针，`store i32` 说明这次写的类型。[LLVM 标识符](https://llvm.org/docs/LangRef.html#identifiers) · [LLVM 指针类型](https://llvm.org/docs/OpaquePointers.html#the-opaque-pointer-type)

```llvm
define i32 @adjust(i32 %x, ptr %out) {
entry:
  %c = icmp slt i32 %x, 10
  br i1 %c, label %then, label %else

then:
  %y1 = add i32 %x, 1
  br label %merge

else:
  %y2 = mul i32 %x, 2
  br label %merge

merge:
  %y3 = phi i32 [ %y1, %then ], [ %y2, %else ]
  store i32 %y3, ptr %out
  ret i32 %y3
}
```

LLVM 最初是 UIUC 的编译器研究项目。Clang 等前端可生成 LLVM IR，优化 pass 在其上分析和改写，目标后端做指令选择、寄存器分配和代码生成。同一项目还提供 LLD 链接器和 LLDB 调试器。

## Module、地址计算与溢出承诺（p.51–55）

一个 module 可以同时包含全局变量、外部声明（只有接口）和函数定义（有函数体）。

```llvm
@counter = global i32 0

declare i32 @external(i32)

define i32 @get_counter() {
entry:
  %v = load i32, ptr @counter
  ret i32 %v
}
```

**结构体字段。** 令 `Point = { i32 x, i32 y }`。下面的 GEP（`getelementptr`）先按 `Point` 布局选择第 0 个对象，再选从 0 编号的字段 1，得到 `y` 的地址。下一条 `load` 才真正读出数值。[LLVM 对 GEP 的说明](https://llvm.org/docs/GetElementPtr.html)

```llvm
%Point = type { i32, i32 }

define i32 @read_y(ptr %p) {
entry:
  %field = getelementptr %Point, ptr %p, i32 0, i32 1
  %y = load i32, ptr %field
  ret i32 %y
}
```

**整数提升。** 在课件假定的 8 位 `unsigned char`、32 位 `int` 平台上，C 的 `int inc(unsigned char x) { return x + 1; }` 先把 x 提升为 int，再做加法。

```llvm
define i32 @inc(i8 %x) {
entry:
  %x32 = zext i8 %x to i32
  %sum = add nsw i32 %x32, 1
  ret i32 %sum
}
```

取 `x = 255`。零扩展后加一得 256，若先在 8 位里加一再扩展则得到 0。`nsw` 表示这次加法承诺无有符号溢出，此例的 32 位结果始终在 1 到 256 之间。若带 `nsw` 的加法实际发生有符号溢出，LLVM 将结果视为 poison，不会自动插入运行时检查。[LLVM 整数加法](https://llvm.org/docs/LangRef.html#add-instruction)

**自测。** `store.i32 7,p` 后紧跟 `v = load.i32 q`，能否改成 `v = 7`？若已证明这两次普通访问指向同一完整的有效整数位置、访问宽度相同，且没有中间修改，就可以。只有 MayAlias 时证据不足。这个区别决定了 [公共子表达式消除](LEC-15-Global-Optim-II.md) 在内存操作前何时必须停下。
