# Ghidra 研究 DOS 版主程序的通用心得

本文记录第一次探索《大航海时代2》DOS 版 `MAIN.EXE` 时得到的通用经验。重点不在某一段交易逻辑，而在 16-bit DOS 程序和 Ghidra 反编译结果的阅读方法。

## 1. Decompiler 不是源码

Ghidra 右侧 Decompiler 输出的是从机器码恢复出的伪 C 代码，不是原始源码。它适合快速看控制流和大致数据流，但不能直接当作事实。

常见现象：

- 同一个临时变量可能在不同阶段代表不同业务含义。
- 函数参数个数可能不完整。
- 寄存器传参可能显示成 `in_AX`、`in_AL`、`in_DX`。
- 全局变量可能显示成裸地址或很绕的指针表达式。
- 常量可能被折叠、换算或隐藏在表达式中。

结论：关键公式、参数来源、溢出行为必须回到 Listing 汇编或 p-code 校验。

## 2. 16-bit 程序里地址通常不是完整地址

8086/80286 风格程序大量使用 `segment:offset`。Ghidra 里看到：

```c
*(undefined2 *)0xc2dc
```

不能简单理解为绝对地址 `0xc2dc`。汇编中的：

```asm
MOV BX, word ptr [0xc2dc]
```

真实含义通常是：

```asm
MOV BX, word ptr DS:[0xc2dc]
```

因此需要先确认当时 `DS` 的值。比如 Memory Map 中如果数据段是：

```text
DATA 4815:baf1 - 4815:faf0
```

那么 `DS:0xc2dc` 对应的是：

```text
4815:c2dc
```

也可换算为线性地址：

```text
0x48150 + 0xc2dc = 0x5442c
```

这类段寄存器上下文如果没有设置，Decompiler 会继续显示裸偏移，已经命名的 label 也可能不会出现在伪 C 中。

## 3. 设置 DS 比只改 label 更重要

如果已经在 `4815:c2dc` 命名了 label，但 Decompiler 仍显示 `0xc2dc`，通常说明 Ghidra 没有把当前函数的 `DS` 解析到 `0x4815`。

处理方式：

1. 在 Listing 中选中相关函数或代码范围。
2. 使用 `Set Register Values...`。
3. 设置 `DS = 0x4815`。
4. 范围尽量覆盖整个函数或至少覆盖相关 basic block。
5. 重新 decompile。

判断 `DS` 是否可覆盖整个函数时，要检查函数内部是否出现修改段寄存器的指令，例如：

```asm
MOV DS,...
POP DS
LDS ...
```

如果没有修改 `DS`，同一函数内的 `[offset]` 访问通常都可按同一个 `DS` 理解。

## 4. `undefined2` 是 Ghidra 的占位类型

`undefined2` 表示：

```text
Ghidra 知道这里是 2 字节，但还不知道具体语义。
```

它可能是：

- `ushort`
- `short`
- 16-bit `int`
- near pointer
- offset
- 结构体字段

因此看到：

```c
*(undefined2 *)0xc2dc
```

不要急着赋予业务含义。它只说明“从某地址读 2 字节”。后续需要结合汇编、访问方式和结构体偏移来决定真实类型。

## 5. `in_AX`、`in_AL`、`in_DX` 通常表示隐式寄存器输入

Decompiler 中出现：

```c
byte in_AL;
uint in_DX;
```

通常说明函数在使用入口寄存器的值，但 Ghidra 没把它恢复成正式参数。

例如函数内部使用 `AL` 判断商品类型，那么更合理的建模可能是：

```c
int get_index_of_good_type(byte goods_id);
```

并设置 custom storage：

```text
goods_id -> AL
return   -> AX
```

寄存器传参函数如果不修签名，调用点经常会丢失数据流，导致前面的计算被 Decompiler 认为“无用”。

## 6. Custom Storage 用来描述寄存器传参

16-bit DOS 程序里并非所有参数都通过栈传递。常见模式是调用前直接设置寄存器：

```asm
MOV DX, value
MOV AX, limit
CALLF some_function
```

如果函数实际读取 `AX` 和 `DX`，Ghidra 默认的 `cdecl`/`stdcall` 栈参数模型就不够。

此时应在函数签名中启用 `Use Custom Storage`，手动指定：

```text
param1 -> AX
param2 -> DX
return -> AX
```

这能显著改善调用点反编译结果。

## 7. near pointer 和 far pointer 必须分清

如果汇编是：

```asm
MOV BX, word ptr [some_global]
```

说明从全局变量中只读了 2 字节到 `BX`，这通常是 16-bit near pointer / offset。

因此这个全局变量应占 2 字节。不要贸然建成 4 字节 far pointer。

判断方式：

- `word ptr` 读 2 字节，通常对应 near pointer 或 16-bit 值。
- `dword ptr` 或显式段:偏移组合才可能是 far pointer。

Ghidra 中把 near pointer 建成结构体指针后，有时会出现很绕的表达式：

```c
(*(MarketData **)&current_market_ptr)->field
```

这通常可读作：

```c
current_market_ptr->field
```

前者只是 Ghidra 为了表达“全局变量里存着一个结构体指针”而生成的形式。

## 8. 结构体建模应从确定偏移开始

不要一次性猜完整结构体。应先从确定访问建立字段。

例如看到：

```asm
MOV CX, [BX]
MOV DL, [BX + DI + 0xe]
MOV [BX + DI + 0xe], AL
```

可以先建：

```c
struct MarketData {
    ushort commerce_value;          // +0x00
    undefined1 unknown_02[0x0c];    // +0x02
    uchar price_fluctuation[10];    // +0x0e
};
```

这里 `commerce_value` 和 `price_fluctuation` 是由访问模式支撑的字段，未知区域保留为 `unknown_*`。

同理，船舱类结构可从数组访问推出：

```c
struct ShipCargoLike {
    undefined1 unknown_00[0x0c];
    ushort goods_qty[5];  // +0x0c
    char goods_id[5];     // +0x16
};
```

字段名应随证据逐步修正。

## 9. `&` 和 `*` 在反编译代码里要按 C 语义读

常见表达式：

```c
*(undefined2 *)0xc2dc
```

拆开是：

```c
(undefined2 *)0xc2dc  // 把数值 0xc2dc 当成指向 2 字节值的指针
* (...)               // 读取这个指针指向的 2 字节
```

另一个常见表达式：

```c
(*(MarketData **)&current_market_ptr)->commerce_value
```

可拆成：

```c
&current_market_ptr                 // current_market_ptr 变量自己的地址
(MarketData **)&current_market_ptr  // 该地址里存放的是 MarketData*
*(MarketData **)&current_market_ptr // 读出 MarketData*
->commerce_value                    // 访问结构体字段
```

业务阅读时可简化为：

```c
current_market_ptr->commerce_value
```

## 10. 连续复用同一临时变量不代表赋值无效

Decompiler 会复用临时变量名，例如：

```c
uVar7 = calc_delta();
uVar7 = clamp(old_value + uVar7);
uVar7 = calc_global_delta();
```

第一句不是无效赋值，因为第二句右侧使用了上一句的 `uVar7`。

判断一次赋值是否有效，要看下一次覆盖之前是否被读取。

如果变量生命周期不同，建议使用注释说明阶段含义，或在 Ghidra 中尝试 split variable 后分别命名。

## 11. 16-bit 乘法需要特别注意结果宽度

8086 的 `MUL` 常见形式是：

```asm
MUL word ptr [...]
```

它会产生 32 位结果：

```text
DX:AX = AX * operand
```

如果后续只使用 `AX`，或执行：

```asm
SUB DX, DX
DIV CX
```

就表示高 16 位被丢弃，后续只用低 16 位结果。这类行为可能是业务 bug 的来源。

与此相对，运行库中的 long 乘法 helper 可能返回 32 位结果。它看起来能处理 32-bit 操作数，但通常只返回低 32 位：

```text
mul_u32_low32(a, b) = low32(a * b)
```

这符合 `long * long -> long` 的运行库语义，不必自动视为 bug。

## 12. `CONCAT22` 表示拼接两个 16 位值

Ghidra 的：

```c
CONCAT22(high, low)
```

意思是：

```text
(high << 16) | low
```

也就是把两个 2 字节值拼成一个 4 字节值。

看到：

```c
(ulong)value >> 0x10
```

通常是在取 32 位值的高 16 位。

看到：

```c
(uint)value
```

在 16-bit 程序中通常表示截取低 16 位，但要结合 Ghidra 当前 `uint` 宽度确认。

## 13. 32 位加减法常拆成高低 word

16-bit 程序中 32 位金额或计数常拆成两个 word 存储：

```text
money_low
money_high
```

减法常见形式：

```c
old_low = money_low;
money_low -= low16(value);
money_high -= high16(value) + (old_low < low16(value));
```

其中：

```c
old_low < low16(value)
```

是低 16 位减法的借位判断。这个表达式会生成 `0` 或 `1`，用于从高 16 位继续扣除。

如果某个全局变量只是金额低 16 位，不应命名为 `money_ptr`。只有变量内容本身是地址时才使用 `_ptr`。

## 14. 函数命名优先表达行为，不急着表达业务

对于明显像运行库 helper 的函数，优先使用行为名：

```text
mul_u32_low32
clamp_u16_to_max
```

等确认它是编译器运行库函数后，再考虑更短的运行库语义名：

```text
long_mul
```

对于业务函数，如果当前只知道局部行为，名字应保守：

```text
get_index_of_good_type
```

不要过早命名成更高层概念，避免后续发现语义不一致。

## 15. 推荐工作流程

1. 先读 Listing 汇编，确认真实寄存器和内存访问。
2. 设置正确的段寄存器，尤其是 `DS`。
3. 给确定的数据地址加 label。
4. 从访问偏移建立最小结构体。
5. 修复明显的寄存器传参函数签名。
6. 重新 decompile，观察常量和字段是否回到数据流。
7. 对短生命周期变量谨慎命名，必要时先加注释。
8. 所有结论都用汇编片段反查一次。

## 16. 记录结论时区分事实和推断

建议笔记中使用三类措辞：

- “确定”：由汇编直接支持，例如某字段位于 `+0x0e`，数组项 1 字节。
- “大概率”：由多处访问和业务上下文支持，但还缺少完整交叉引用。
- “待确认”：仅由命名、流程或单点观察推断。

这能避免后续研究被早期命名误导。

