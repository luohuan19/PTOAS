# A5 VF 基础能力介绍

[返回 A5 文档目录](README.md)

> 本文整理 VF 硬件背景、Ascend C 示例及 PTOAS VMI/MI 接口。硬件与第三方接口描述沿用原稿来源，本次归档未重新验证；PTOAS 动态偏移、UB alias 和 Unknown 同步限制请参见 [能力指南第 13 节](ptoas-a5-capabilities-guide.html#ub-alias-limits)。

## 1\. VF 的硬件基础

### 1\.1 数据路径与 VF 执行域

传统 Memory 矢量计算的操作数和中间结果位于 UB；Reg 矢量计算引入 SIMD Register File，让连续计算在寄存器内消费中间结果，减少反复读写 UB。一次典型的 VF 数据路径如下：

```Plain Text
GM --外层搬运--> UB --LOAD--> SIMD 寄存器 --VEC--> SIMD 寄存器 --STORE--> UB --外层搬运--> GM
                       SCALAR：循环控制、地址与偏移计算
```

寄存器不能直接访问 GM；进出 GM 必须经由 UB。Ascend C 在 `__aicore__` 执行域通过 `asc_vf_call` 进入 `__simd_vf__` 函数，VF 内调用 `__simd_callee__` 的 Reg 接口。VF 外的标量计算由 Main Scalar 负责，VF 内的标量计算由 Aux Scalar 负责。[来源：Reg 矢量编程模型](https://asc.gitcode.com/guide/programming_guide/programming_model/ai_core_simd_programming/c_pointer_programming/reg_vector_computation.html)。

VF 内的工作可按作用理解为四类：

|类别|执行内容|数据位置与典型 Ascend C 操作|
|---|---|---|
|LOAD|Reg 搬运单元从 UB 读入寄存器。|`asc_loadalign`、`asc_loadunalign` 等。|
|VEC|Reg 矢量执行单元消费/产生寄存器值。|`asc_add`、`asc_mul`、`asc_half2float` 等矢量计算。|
|STORE|Reg 搬运单元将寄存器数据写回 UB。|`asc_storealign`、`asc_storeunalign` 等。|
|SCALAR|Aux Scalar 处理 VF 内循环控制、地址及偏移等标量计算。|循环计数、地址表达式、`addr_reg` 更新等。|

这四类是理解 VF 内部操作和依赖的划分，不表示四条相互独立的硬件流水线。Reg 矢量执行单元、Reg 搬运单元和 Aux Scalar 都归属 `PIPE_V`；无依赖时，搬运与计算可能并行。外层 GM/UB 搬运及其流水线同步属于另一层边界。[来源：硬件原理与执行域](https://asc.gitcode.com/guide/programming_guide/programming_model/ai_core_simd_programming/c_pointer_programming/reg_vector_computation.html)。

### 1\.2 寄存器与 UB 规格

下表是 3510 Reg 矢量 VF 的物理资源规格。Ascend C 代码使用类型化变量，编译器负责分配物理寄存器；下表数量表示同一 VF 的资源上限，不能把源码声明数直接等同于同时占用数。[来源：VF 寄存器资源限制](https://asc.gitcode.com/guide/operator_practice/simd_operator_optimization/vector_compute/vf_optimization/dual_issue_optimization.html)。

|寄存器类别|Ascend C 表达|单个容量|数量|
|---|---|---|---|
|矢量数据寄存器|`vector_*` / `RegTensor`|256 B（1 VL）|32|
|谓词/掩码寄存器|`vector_bool` / `MaskReg`|32 B（VL/8）|8|
|地址寄存器|`addr_reg` / `AddrReg`|32 bit|8|
|搬入非对齐辅助寄存器|`vector_load_unalign` / `UnalignRegForLoad`|32 B（1 DataBlock）|4|
|搬出非对齐辅助寄存器|`vector_store_unalign` / `UnalignRegForStore`|32 B（1 DataBlock）|4|

一个 256 B 数据寄存器可存放 256 个 8\-bit、128 个 16\-bit 或 64 个 32\-bit 元素。掩码寄存器决定哪些元素有效；地址寄存器保存 UB 搬运偏移；两类非对齐辅助寄存器暂存跨 32 B 对齐边界的数据。寄存器按整体使用，不能像 UB 数组一样按索引访问单个 lane。[来源：寄存器类型与内存层级](https://asc.gitcode.com/guide/programming_guide/programming_model/ai_core_simd_programming/c_pointer_programming/reg_vector_computation.html)。

3510 架构单个 AIV 核的 UB 总容量为 **256 KiB**，由 **16 个 16 KiB bank** 组成；SIMD VF 栈默认预留 6 KiB，Ascend C 接口默认预留 2 KiB，因此常规配置下其他用途最多剩余约 **248 KiB**，还要扣除实际 buffer、临时空间与对齐开销。预留与可用容量会受编译选项和运行模式影响。[来源：3510 架构变更](https://asc.gitcode.com/guide/cross_gen_migration_guide/instructions_for_new_features/3510_new_features.html)。

### 1\.3 同一 UB 区域的读写同步

LOAD、VEC、STORE 虽都归属 `PIPE_V`，搬运与计算仍可能并行。**寄存器值的依赖**由硬件保证顺序，例如 LOAD 写入某寄存器后 VEC 读取该寄存器，无须为每条计算手动加屏障。**UB 地址上的冲突**则需要单独判断：不同寄存器访问同一或重叠 UB 区域时，源码先后顺序不一定形成所需的访存依赖。[来源：流水线同步](https://asc.gitcode.com/guide/programming_guide/programming_model/ai_core_simd_programming/c_pointer_programming/reg_vector_computation.html)。

|前一项操作 \-\> 后一项操作|同一或重叠 UB 区域时|Ascend C 同步方向|
|---|---|---|
|STORE \-\> LOAD|后续 LOAD 必须读到前次 STORE 的结果。|`asc_mem_bar(VST_LD)`|
|LOAD \-\> STORE|前次 LOAD 必须完成，才能覆盖其读取的 UB 数据。|`asc_mem_bar(VLD_ST)`|
|STORE \-\> STORE|两次写入必须按预期顺序落地。|`asc_mem_bar(VST_ST)`|
|无重叠的 UB 访问|没有该地址上的读写冲突。|不因 UB 地址而加屏障。|

例如，把一组寄存器写回 UB 后立即由另一组寄存器从同一地址读取，需要明确建立 `STORE -> LOAD` 顺序：

```C++
asc_store(ub_addr, src_reg);
asc_mem_bar(VST_LD);
asc_load(dst_reg, ub_addr);
```

若读写之间存在能够串起操作的寄存器数据依赖，还需按实际指令链判断，不能仅凭“同一地址”机械插入屏障。`asc_vf_call` 返回时，VF 内部 Reg 矢量指令已执行完成；这里的 `asc_mem_bar` 讨论的是**一次 VF 执行期间**对 UB 读写的排序，不等同于 VF 外 GM/UB 搬运的跨流水线同步。PTOAS 是否自动插入屏障、如何暴露对应同步接口，将在第二步按实现分析。[来源：Ascend C 同步规则](https://asc.gitcode.com/guide/programming_guide/programming_model/ai_core_simd_programming/c_pointer_programming/reg_vector_computation.html)。

### 1\.4 带循环的 VF 使用示例：8 × 2048 FP32 Add

下面用 Ascend C 计算 `dst[row, col] = lhs[row, col] + rhs[row, col]`，三个张量的形状均为 `8 × 2048`、类型为 FP32，按行连续存放在 UB 中。一个 VL 为 256 B，可容纳 64 个 FP32 元素；把二维数据视为连续的 16384 个元素后，只需一层 `16384 / 64 = 256` 次的循环。三个张量各占 64 KiB，合计 192 KiB；示例假定三个 UB 区域均满足 32 B 对齐且互不重叠。GM 与 UB 之间的搬运由外层另行完成。

```C++
__simd_vf__ inline void add_vf(__ubuf__ float* dst,
                               __ubuf__ float* lhs,
                               __ubuf__ float* rhs)
{
    vector_float a, b, result;
    vector_bool mask = asc_create_mask_b32(PAT_ALL); // PReg: 64 个 FP32 lane 全有效
    addr_reg offset;                                 // AReg: UB 搬运偏移

    for (uint16_t block = 0; block < 256; ++block) {
        offset = asc_update_addr_reg_b32(64);        // SCALAR: 每轮前进 64 个 FP32 元素
        asc_loadalign(a, lhs, offset);               // LOAD: UB -> 寄存器
        asc_loadalign(b, rhs, offset);               // LOAD: UB -> 寄存器
        asc_add(result, a, b, mask);                // VEC: 寄存器计算
        asc_storealign(dst, result, offset, mask); // STORE: 寄存器 -> UB
    }
}

__aicore__ inline void add_tile(__ubuf__ float* dst,
                                __ubuf__ float* lhs,
                                __ubuf__ float* rhs)
{
    asc_vf_call<add_vf>(dst, lhs, rhs);   // 从外层执行域进入 VF
}
```

循环计数和 AReg 地址更新属于 VF 内的 SCALAR 工作；`b32` 模式的步长 64 以 FP32 元素计，最末次访问覆盖 `16320..16383`，没有越界或尾块。LOAD、VEC、STORE 通过所用寄存器形成数据依赖；由于三个 UB 区域互不重叠，这里不需要 `asc_mem_bar`。若改为复用同一 UB 区域，应按 1\.3 节重新检查访存依赖。`for` 写法符合此处的硬件循环候选形式，但是否实际生成硬件循环仍由编译器决定。[来源：Ascend C VF 加法示例](https://asc.gitcode.com/guide/programming_guide/programming_model/ai_core_simd_programming/c_pointer_programming/reg_vector_computation.html)、[AReg 更新接口](https://asc.gitcode.com/api/SIMD-API/c_api/reg_compute/reg_addr_reg/asc_update_addr_reg.html)。

### 1\.5 PReg、UReg、AReg 在 VF 中的作用

上例的 `vector_bool mask` 是 **PReg**（谓词寄存器）：`asc_create_mask_b32(PAT_ALL)` 使当前 64 个 FP32 元素全部参与 `asc_add` 和 `asc_storealign`。若只处理一个 VL 的前 32 个 FP32 元素，可改用 `asc_create_mask_b32(PAT_VL32)`；这会改变参与计算和写回的 lane，不改变 UB 基地址。PReg 也可承载比较结果。[来源：掩码生成接口](https://asc.gitcode.com/api/SIMD-API/c_api/reg_compute/reg_mask/asc_create_mask.html)。

上例的 `addr_reg offset` 是 **AReg**（地址寄存器）：`asc_update_addr_reg_b32(64)` 在循环迭代中按 64 个 FP32 元素更新偏移，再由支持 AReg 的 `asc_loadalign`/`asc_storealign` 使用。它控制**访问 UB 的位置**，不是数据寄存器，也不充当元素掩码；若只访问一次，也可以直接用 `lhs + 64` 等地址表达式。[来源：AReg 更新接口](https://asc.gitcode.com/api/SIMD-API/c_api/reg_compute/reg_addr_reg/asc_update_addr_reg.html)。

本例所有地址对齐，不需要 **UReg**（非对齐搬运辅助寄存器）。如果改为从 `lhs + 1` 开始读取 FP32，一个起始地址偏离 32 B 边界的 256 B 读取可写成如下片段；调用方必须保证这段访问的整个范围有效：

```C++
vector_load_unalign ureg;
vector_float data;
__ubuf__ float* unaligned_lhs = lhs + 1;
asc_loadunalign_pre(ureg, unaligned_lhs);
asc_loadunalign(data, ureg, unaligned_lhs);
```

搬入 UReg 缓存跨 DataBlock 边界的数据，不能把它当作普通数据寄存器；连续非对齐搬入时通常在循环前初始化一次。非对齐搬出则使用另一类 `vector_store_unalign`，并在结束时调用 `asc_storeunalign_post` 处理暂存尾部。[来源：非对齐搬入接口](https://asc.gitcode.com/api/SIMD-API/c_api/reg_compute/load/asc_loadunalign.html)、[连续非对齐优化](https://asc.gitcode.com/guide/operator_practice/simd_operator_optimization/vector_compute/vf_optimization/continuous_unaligned_optimization.html)。

### 1\.6 掩码、合并模式与位置选择

这几类参数控制不同层面的行为，不能互换：

|类别|常见值|作用及示例|
|---|---|---|
|PReg 创建 pattern|`PAT_ALL`、`PAT_VL32`、`PAT_H`、`PAT_Q`、`PAT_ALLF`|分别选中全部、低 32 个、低一半、低四分之一、零个元素；例如 FP32 的 `asc_create_mask_b32(PAT_VL32)` 只选中前 32 个 lane。`PAT_M3`/`PAT_M4` 还可选中下标为 3/4 倍数的元素。|
|MaskMergeMode|`ZEROING`、`MERGING`|掩码未选中的目的 lane 分别清零、保留原值；例如对部分 lane 做 Add 时，`MERGING` 让其余 lane 保留旧 `dst`。Ascend C C API 的 `asc_add` 对未选中的 lane 置零；基础 API `Reg::Add` 可选 `MaskMergeMode::MERGING`，其实现可能由软件组合。|
|位置选择|`PART_EVEN` / `PART_ODD`；Ascend C C API 对应 `ASC_POSITION_EVEN` / `ASC_POSITION_ODD`|用于宽窄类型转换时选择偶数或奇数位置，不表示计算掩码。例如 FP32 转 FP16，偶数模式把 64 个转换值放到 128\-lane FP16 寄存器的偶数位置，奇数位置清零；反向 FP16 转 FP32 时分别读取偶数、奇数位置，需两个 FP32 寄存器才能覆盖 128 个 FP16 元素。|

`PART_EVEN`/`PART_ODD` 是 TileLang 等接口对位置选择的命名；Ascend C C API 以 `ASC_POSITION_EVEN`/`ASC_POSITION_ODD` 表示。TileLang 的 `MODE_ZEROING`/`MODE_MERGING` 则对应上表的结果写入模式。这些参数并非所有指令都接受；例如 `MaskMergeMode::MERGING` 在基础 API 的 Add 中由软件支持，不能据此推断为单条原生硬件指令。[来源：掩码 pattern](https://asc.gitcode.com/api/SIMD-API/c_api/reg_compute/reg_mask/asc_create_mask.html)、[类型转换的位置选择](https://asc.gitcode.com/api/SIMD-API/c_api/reg_compute/reg_convert/asc_float2half.html)、[合并模式支持列表](https://asc.gitcode.com/api/appendix/reg_vector_compute_interface_mergemode_support_summary.html)。

## A5 SIMD VF 硬件约束与优化指导

### 0\. vector计算单元

> 原稿引用的硬件示意图未随文档提供；以下文字说明保留原整理内容。

AUX SCALAR: 附属Scalar单元，用于Vector Function内Scalar操作的执行

REG执行单元: 计算指令执行ex单元，valu，存在俩，但规格又差异，部分计算指令一个执行单元支持

DMA单元: VLD/VST相关读写UB的能力，At Most: a\. 2 load \+ 1 store / cycle, b\. 1 gather / cycle, c\. 1 scatter / cycle

### 1\. VF 寄存器规格

SIMD Register File 位于 UB 与 Vector 计算单元之间；VF 中的典型路径为 `UB → LOAD → VReg → VEC → VReg → STORE → UB`。VF 内地址、计数和循环控制由 Aux Scalar 参与执行。寄存器不能直接从 GM 读取或直接写回 GM；跨 GM/UB 的搬运位于 VF 外。LOAD、VEC、STORE 和 Aux Scalar 归属 `PIPE_V`，存在数据依赖的操作须按依赖顺序执行，互不依赖的操作才可能并行。[来源：Reg 矢量硬件原理](https://asc.gitcode.com/guide/programming_guide/programming_model/ai_core_simd_programming/c_pointer_programming/reg_vector_computation.html)、[3510 架构规格](https://asc.gitcode.com/guide/programming_guide/advanced_programming/hardware_implementation/architecture_spec/npu_arch_3510.html)。

|资源|单个容量|数量上限|主要约束|
|---|---|---|---|
|向量数据寄存器 VReg|256 B|32|一个满宽寄存器容纳 256 个 b8、128 个 b16 或 64 个 b32 元素；同时存活值过多会增加 spill 风险。|
|谓词寄存器 PReg|32 B|8|一个 bit 对应一个 byte 位置；b16/b32 元素分别按 2/4 bit 分组解释其有效 bit。|
|地址寄存器 AReg|4 B|8|保存 UB 地址偏移；其寄存器个数与最多四层循环寻址是两个不同限制。|
|非对齐搬入辅助寄存器 UReg|32 B|4|连续非对齐读的跨块状态，不能当作普通 VReg。|
|非对齐搬出辅助寄存器 UReg|32 B|4|连续非对齐写的跨块状态，写流结束须收尾。|

这些是**同一 VF 的物理资源上限**，不是 C\+\+ 局部变量个数限制；生命周期结束的寄存器可以复用。向量/掩码/地址寄存器压力过高时，编译器可能换入换出并增加 UB 流量与同步；非对齐辅助寄存器数量超限可导致编译失败。对优化结果应查看目标编译器资源报告与最终指令，不能仅靠源码变量数判断。[来源：3510 架构寄存器表](https://asc.gitcode.com/guide/programming_guide/advanced_programming/hardware_implementation/architecture_spec/npu_arch_3510.html)、[VF 双发与寄存器压力](https://asc.gitcode.com/guide/operator_practice/simd_operator_optimization/vector_compute/vf_optimization/dual_issue_optimization.html)。

### 2\. 256 KiB 与 248 KiB 的区别

**256 KiB 是单个 AIV 的 UB 物理总容量**，不是默认可全部分配给用户 buffer 的空间。默认配置下，VF 栈预留 6 KiB、Ascend C 接口预留 2 KiB，普通用途最多约 **248 KiB**；用户 buffer、编译器临时空间和对齐填充还需从中扣除。在 SIMD\+SIMT 混编时，UB 还可能划出 32–128 KiB 作为 Data Cache。设备与编译选项不同，要重新计算预算。[来源：3510 UB 规格与预留](https://asc.gitcode.com/guide/programming_guide/advanced_programming/hardware_implementation/architecture_spec/npu_arch_3510.html)。

|配置或工作集|容量计算|不能忽略的代价|
|---|---|---|
|默认 SIMD 配置|`256 - 6 - 2 = 248 KiB` 可作为普通 UB 的上界|上界仍不包含其他同时存活的对象、临时区和对齐损耗。|
|释放 VF 栈预留|理论上增加 6 KiB|编译器不能再用该空间缓存寄存器溢出数据，必须确保无 spill。|
|释放 Ascend C 预留|理论上再增加 2 KiB|依赖这 2 KiB 的 Ascend C 接口不再可用。|

两个释放预留空间的选项不能被视为通用优化：前者与寄存器溢出机制互斥，后者会改变可调用接口范围。尤其把 UB 用到接近 248 KiB 时，再增加少量 padding 或临时值也可能超额。[来源：3510 编译选项说明](https://asc.gitcode.com/guide/cross_gen_migration_guide/instructions_for_new_features/3510_new_features.html)。

### 3\. 单发、双发与寄存器溢出

双发（dual issue）要求两条待发指令均已就绪、没有妨碍并行的依赖，且所需执行资源可同时使用；否则只能按硬件允许的节奏发射。LOAD 与 VEC、部分独立计算、不同的对齐搬运之间可能存在并行机会；`load(a) → add(a,b) → store(result)` 的同一条数据依赖链不能通过改写源码顺序变成两条独立工作。手动展开或交错不同迭代可提供独立指令，但也会让更多 VReg/PReg 同时存活。寄存器溢出会引入额外 UB 读写和同步，抵消预期的双发收益。[来源：Reg VF 指令并行](https://asc.gitcode.com/guide/programming_guide/programming_model/ai_core_simd_programming/c_pointer_programming/reg_vector_computation.html)、[双发优化指南](https://asc.gitcode.com/guide/operator_practice/simd_operator_optimization/vector_compute/vf_optimization/dual_issue_optimization.html)。

优化顺序应是：先数清单次迭代的真实数据依赖和同时存活寄存器，再尝试小幅展开/交错；比较最终指令中的 spill、同步和运行时间。不能仅以“源码中相邻两句可并行”或“循环已展开”推断实际双发。VF 融合虽可减少中间 UB 往返和循环启动次数，但融合过大也会提高寄存器压力。[来源：VF 融合与双发](https://asc.gitcode.com/guide/programming_guide/programming_model/ai_core_simd_programming/c_pointer_programming/reg_vector_computation.html)。

### 4\. 四层 Hardware Loop VLoop 与 `uint16_t` 计数

Hardware Loop条件:

为了能让编译器识别并生成Hardware Loop，对应的Loop代码必须符合硬件设计的要求。具体规范如下：

- 迭代变量类型

- VF内所有Loop的迭代变量必须是uint16\_t类型。

- 起始值与步长

- 循环起始值从0开始。

- 每次迭代的步长必须是递增1。

- 循环内不允许跳转指令，比如if/else、三元运算符?:等条件判断跳转。

- VF内if/else在Loop内会阻碍Hardware Loop的生成，编译器虽然会尽可能做if/else消除优化，但是不做完全性保证。

- if控制流可使用if constexpr或者for\(1\)替换。if constexpr在编译时已经完成，无运行时开销，但是需要传入的参数是编译时常量，不能依赖运行时变量；for\(1\)可以触发编译器的循环优化，在当前硬件条件下，Vector侧执行Loop的性能是远远优于条件分支跳转的。

[https://asc.gitcode.com/guide/operator_practice/simd_operator_optimization/vector_compute/vf_optimization/vf_loop_optimization.html#Hardware-Loop%E7%BC%96%E7%A0%81%E8%A7%84%E8%8C%83]()

VF 允许最多 **四层嵌套循环**；满足编译器识别条件的循环才可能成为 Hardware Loop。不满足条件的合法循环通常按 Software Loop 执行，并增加标量计数和跳转开销。Ascend C 指南给出的硬件循环规范包括：迭代变量用 `uint16_t`，从 0 起、每轮加 1，循环执行期间不修改边界，循环体内没有未被消去的运行时 `if/else` 或三元跳转。同层顺序循环与嵌套深度是不同概念；是否真的生成硬件循环须检查最终指令。[来源：VF 循环指南](https://asc.gitcode.com/guide/operator_practice/simd_operator_optimization/vector_compute/vf_optimization/vf_loop_optimization.html)。

```C++
for (uint16_t i = 0; i < repeat_times; ++i) {
    // UB address update, register load, vector compute, register store
}
```

对这种 `i < repeat_times` 写法，`uint16_t` 的取值为 `0..65535`，**安全上界是 ****`repeat_times <= 65535`**；循环体若额外引入分支，则仍可能退化为软件循环。[来源：VF 循环规范](https://asc.gitcode.com/guide/operator_practice/simd_operator_optimization/vector_compute/vf_optimization/vf_loop_optimization.html)。

### 5\. UB Bank 冲突

3510 UB 为 **8 个 bank group，每组 2 个 16 KiB bank**，每个 bank 的行宽 32 B，地址按低位交织。对 32 B 对齐的块起点，官方地址格式 `UB_ADDR[18:0] = {BANK_DEPTH[8:0], BANK[0], BG[2:0], BANK_WIDTH[4:0]}` 给出 `BG=(addr>>5)&7`、`BANK=(addr>>8)&1`。同一 bank 一拍只能完成一读或一写；同组最多两读，或一读一写，不能两写。两次读落在同组**不同** bank 时可并行，落在同一 bank 时冲突；gather/scatter 还需考虑每 bank 的 4 个 8 B subbank。[来源：3510 bank 结构与冲突规则](https://asc.gitcode.com/guide/operator_practice/simd_operator_optimization/memory_access/avoid_ub_bank_conflict/avoid_bank_conflict_npu_arch_3510.html)。

官方例子中，两个独立对象的块起点 `0x0000` 与 `0x4000` 同属 bank 0、group 0，两读同拍会竞争；把第二个块起点移至 `0x4100`，它落在 group 0 的另一个 bank，这一对读可避免该冲突。相同 bank **不等于同一 UB 地址**；bank 冲突是吞吐问题，同一地址的数据依赖是正确性问题。对完整 256 B VL，连续操作会触及多个 32 B bank 行，因此不能仅凭对象起点判定整条指令必定冲突。改变 buffer 起点、行跨度或访问顺序前，应核对容量、32 B 对齐与实际指令时序，并以测量结果判断收益。[来源：3510 bank 地址示例](https://asc.gitcode.com/guide/operator_practice/simd_operator_optimization/memory_access/avoid_ub_bank_conflict/avoid_bank_conflict_npu_arch_3510.html)。

### 6\. 精度标准、饱和与整数溢出

官方[Reg 矢量计算接口精度标准汇总](https://asc.gitcode.com/api/appendix/reg_vector_compute_interface_precision_standard_summary.html)仅适用于 Ascend 950PR/950DT，并把**硬件指令**与**软仿**分列。下表摘出硬件指令列中与常见 VF 计算有关的规则；ULP 是相对接口指定计算结果的误差单位，`0 ULP` 不等于整个算子的数学结果无误差，更不能把多条指令的误差简单相加成统一保证。

|接口族|硬件指令精度标准|对 VF 编写的影响|
|---|---|---|
|`Exp`、`Sqrt`、`Div`、`Ln/Log`|**1 ULP**，**不支持非正规数**（denormal/subnormal）。|涉及极小值时不能直接套用普通输入的误差预期；`Log2/Log10` 在该汇总表硬件列为 `-`，不能从 `Ln/Log` 外推。|
|`Add`、`Adds`、`Mul`、`Muls` 的 half/float 形式|**0 ULP**，**支持非正规数**。|这是单条接口的精度列，不承诺由多个加法构成的链具有与实数计算完全相同的结果。|
|`Add`、`Adds`、`Mul` 的 BF16 形式|先将 BF16 输入转成 FP32 完成该步运算，再以 RINT 舍入回 BF16；`Sub(x,y)=Add(x,Neg(y))`。|BF16 中间值仍会在该步落回 BF16；不能假定整条计算链一直以 FP32 保留精度。|
|`ExpSub`|half 输入先升到 float 后执行减法与 Exp，输出 float；float 输入直接执行减法与 Exp。|复合指令的中间类型是精度契约的一部分，不能简单等同于逐条 half 指令串接。|
|`Axpy`、`MulDstAdd`、`MulAddDst`|乘累加采用高精度计算，结果等效于 FMA。|不能把该融合结果视为分别执行一次 Mul 和一次 Add 的必然相同舍入。|
|`Reduce`、`ReduceDataBlock`|half/float **0 ULP**，采用二叉树结构的成对累加；`PairReduceElem` 也列为 half/float 0 ULP。|改变归约树、分块边界或输入顺序时，可能改变最终数值；应按目标树形比较。|

同页的**软仿列不能替代硬件列**：例如 `Sqrt` 的 float 软仿列给出 0 ULP，而硬件列为 1 ULP；Exp/Div/Log 的非正规数支持情况也可不同。该页还列出低精度接口范围：FP4/FP8/HiF8 主要是 Cast、Duplicate（部分为软仿），不能仅因有这些 dtype 就推断所有 Reg 算术都支持它们。[来源：精度标准及低精度指令清单](https://asc.gitcode.com/api/appendix/reg_vector_compute_interface_precision_standard_summary.html)。

精度标准与**类型转换的舍入/饱和**是两类约束，需分别核查：

|场景|已公布的数值语义|检查点|
|---|---|---|
|`FP32 → FP16 → FP32`|FP32 缩窄会按所选转换模式舍入；升回 FP32 不能恢复缩窄时丢失的尾数。|明确具体转换接口的舍入和 NaN/Inf/超范围行为，不把第二次扩宽当作精度恢复。|
|饱和转换|越界值限制到目标可表示范围；官方示例中 FP32→FP16 的饱和模式把 `70000` 限到 `65504`，并规定特殊值处理。|为量化/缩窄明确饱和模式，检查极值、NaN、Inf。|
|非饱和整型窄化|只保留目标宽度的低位，例如 `int32 511 → int8 -1`。|不把截断误认为饱和。|
|普通整数加减|无进位形式按目标位宽回绕；带进位加法输出独立 carry，低 32 位和仍为截断结果。|需要更宽结果时选更宽计算或显式处理 carry；mask 不改变溢出语义。|

转换和溢出规则分别见[舍入与饱和模式](https://asc.gitcode.com/api/SIMD-API/c_api/reg_compute/reg_convert/rounding_mode.html)、[Reg 加法及进位](https://asc.gitcode.com/api/SIMD-API/c_api/reg_compute/reg_arith/asc_add.html)、[Reg 减法](https://asc.gitcode.com/api/SIMD-API/c_api/reg_compute/reg_arith/asc_sub.html)。任何一行都不能代替具体指令或 dtype 组合的正式接口约束。

## PTOAS VMI 与 MI 当前实现

> 状态：VF 系列第二步，待评审。本文集中整理 PTOAS 的 VMI/MI、布局用法与现有 PyPTO 的接入边界，不提出第三步的 PTO DSL `vecscope` 新 API。硬件背景见\[第一步\]\(10\-pto\-dsl\-vf\-design\.md\)；PyPTO 前端和 TileLang 对照见\[跨仓概览\]\(07\-pypto\-ptoas\-current\-state\.md\)。以下以本地 PTOAS 的 `[VMI 架构概览](../../isa/vmi-isa/00-architecture-overview.md)`、`docs/isa/vmi-isa/10-appendices.md`、`include/PTO/IR/VMIOps.td`、`ptodsl/ptodsl/_vmi_namespace.py` 和 `tools/ptoas/ptoas_pipeline.cpp` 为准。
>
>

参考文章
https://github\.com/hw\-native\-sys/PTOAS/tree/master/docs/isa/vmi\-isa/00\-architecture\-overview\.md

https://github\.com/hw\-native\-sys/PTOAS/tree/master/docs/isa/micro\-isa

### 1\. PTO、VMI、MI 的边界

```Plain Text
PyPTO pl.* ---------------------------> PTO Tile IR (.pto) -> 当前 PyPTO 集成的 C++ 路径
PTODSL pto.tile.* / pto.vmi.* --------> PTO Tile/VMI IR (.pto)
                                        -> PTOAS VMI 校验、掩码粒度与布局分配
                                        -> vmi-to-vpto：逻辑值展开为物理寄存器与 pto.* 微操作
                                        -> VPTO 调度、LLVM/HiVM 发射（A5 vpto 后端）
```

|层|表达内容|对程序员可见的关键点|
|---|---|---|
|Tile PTO|`pto.*` 的 tile/搬运/控制流；Tile 可携带 `valid_shape`、布局等信息|管理 GM/UB 数据和 tile 生命周期，尚未进入寄存器级 VF。|
|VMI（Virtual Micro\-instruction）|`pto.vmi.*`、`!pto.vmi.vreg<LxT>`、`!pto.vmi.mask<L>`|表示逻辑上连续的 `L` 个 lane；通常不指定物理 `part`、交织和寄存器编号。|
|VPTO / MI（物理微操作）|物理宽度的 `pto.*` vector/predicate/地址与同步 op|编码具体搬运模式、谓词粒度、`part` 和数据重排；文档称 `pto.mi` 层，实际 op 拼写多为 `pto.vlds`、`pto.vadd` 等，不存在独立的 `pto.mi.*` 用户命名空间。|

#### 1\.1 逻辑值与物理资源

VMI 以 `!pto.vmi.vreg<LxT>` 表示 **L 个 T 类型的逻辑 lane**，以 `!pto.vmi.mask<Lxpred>` 表示与其逐 lane 对应的谓词。用户描述元素级计算意图，不在 VMI op 上指定物理寄存器编号、`part`、`pack` 或 `dist`；这些由布局分配及物理化决定。架构概览里的 `TileLang T.parallel → pto.vmi → pto.mi` 是分层设计示意，**不能据此推断当前 TileLang 或 PyPTO 已把该路径接通**；本地已有的直接输入是 PTODSL `pto.vmi.*` 或手写 `.pto`。

3510/A5 的一个物理 VReg 为 **256 B（2048 bit）**，32 个 VReg；一个 PReg 为 **256 bit**，8 个 PReg。VReg 内以 **32 B VLane** 为子块，每个 VReg 有 8 个 VLane；一个 VLane 可容纳 8 个 FP32、16 个 FP16/BF16 或 32 个 i8 元素。一个逻辑值的基本承载数为 `K = ceil(L × bitwidth(T) / 2048)`，不足一个物理寄存器仍占一个载体：

|逻辑类型|有效数据量|基本物理承载|说明|
|---|---|---|---|
|`vreg<256xf32>`|1024 B|4 VReg|连续布局下每个载体有 64 个 FP32 lane。|
|`vreg<256xf16>`|512 B|2 VReg|连续布局下每个载体有 128 个 FP16 lane。|
|`vreg<256xi8>`|256 B|1 VReg|满宽 i8 载体。|
|`vreg<64xf16>`|128 B|1 VReg|只有 64 个逻辑 lane 有效，其余物理槽位不可当作有效数据使用。|

上表是**数据量对应的基本承载数**，不是任何布局和任意时刻的实际寄存器占用保证。窄化 `vcvt` 可暂时以带 `lane_stride` 的载体保存结果，后续访存再打包；`vreg` 的 SSA 名称也不是物理编号。一个 VMI `vadd` 可以展开为 `K` 个物理计算，而转换、归约、非连续布局可能额外插入谓词打包、重排或物化。满宽单寄存器的简单运算可接近 1:1，VMI 整体不是 MI 的严格 1:1 别名。来源：PTOAS `[VMI 架构概览](../../isa/vmi-isa/00-architecture-overview.md)`、`include/PTO/Transforms/Passes.td`。

#### 1\.2 布局与掩码如何落到 MI

对同一 `vreg<256xf32>`，**contiguous** 布局把相邻 64 个逻辑 lane 顺序放入一个物理 VReg；**deinterleaved=2** 可把偶数和奇数 lane 分置不同载体；类型宽度改变时还可能出现 EVEN/ODD、子位置或 `lane_stride` 布局。`#pto.vmi.layout<...>` 记录这种映射，不能与 UB 访存参数 `dist_mode` 混同，后者的实际用法见下文“布局：UB 访问模式与寄存器承载”。布局属于 PTOAS 的 IR 决策，不是 `pto.vmi.vreg()` 的用户参数。

逻辑 `pred` 每个 lane 控制一个数据元素；物理 PReg 则按字节位置编码。`vmi-mask-granularity-assignment` 将抽象谓词分配为与数据宽度相配的 `b8/b16/b32` 粒度，布局分配还须让 mask 与数据载体对应。未活动计算 lane 的已支持 `pmode` 为 `"zero"`；`"merge"` 目前被 VMI verifier 拒绝。A5 的 `vload` 没有读 mask，`vstore` 可以带谓词；计算或写回的 mask 不能缩小前面 load 的物理读取范围。逻辑 mask 因而不能直接当作一个固定 PReg 位串。来源：PTOAS `[VMI 架构概览](../../isa/vmi-isa/00-architecture-overview.md)`。

PTOAS 按操作与物理布局的关系区分三类：

|类别|典型行为|物理化影响|
|---|---|---|
|A：布局传递|多数逐元素计算，如 `vadd`。|保持输入的 lane 排列，按物理载体展开；谓词也按载体分配。|
|B：布局可重写|`vcvt`、合适的分组规约或广播。|选择 EVEN/ODD、pack/unpack 等自然布局，并把关系写入结果类型。|
|C：要求连续|无法直接消费当前交错布局的操作，如部分非对齐分组规约。|先插入布局物化，再执行目标操作。|

`group=C` 表示把 `L` 个 lane 分为 **C 组**，不是每组 C 个 lane：例如 `vcadd(..., group=8)` 可将 256 lane 的八组各规约为一个值，得到逻辑 `vreg<8xT>`；结果的物理槽位未必连续，后续 `vbrc(..., group=8)` 从这些槽位广播。分组访存如何指定行跨度见下文“布局：UB 访问模式与寄存器承载”。实际 pipeline 先做 VMI 规范化与校验，再分配 mask 粒度和布局，必要时插入/优化 `ensure_layout` 等内部辅助 op，最后由 `vmi-to-vpto` 展开为物理值与 `pto.*` 微操作。来源：PTOAS `[VMI 架构概览](../../isa/vmi-isa/00-architecture-overview.md)`、`tools/ptoas/ptoas_pipeline.cpp`。

### 2\. VMI  architechture

```Plain Text
TileLang  T.parallel(N) { C[i] = cast<i32>(A[i]) + B[i] }
   │  (direct translation, elementwise semantics preserved)
   ▼
pto.vmi   %w = pto.vmi.vcvt %a; %c = pto.vmi.vadd %w, %b
   │  (pto.as: layout-assignment + lowering)
   ▼
pto.mi    vcvt EVEN/ODD + two-way vadd + vstsx2 INTLV_B32
```

- **Upper → vmi**: `T.parallel`'s logical iteration space translates directly

to `pto.vmi` logical vector ops — elementwise → Category A op, `T.cast` →

a `vcvt` with no explicit `part`, logical length `N` →

`!pto.vmi.vreg<N×T>`, "all active" → auto\-generated tail predicate\.

- **vmi → pto\.mi**: `pto.as` performs layout inference \+ unification \+

materialization, lowering logical vectors to concrete `pto.mi` instructions

\(including `part/pack/interleave/dist`\)\. At `K=1` this degenerates to

zero\-overhead pass\-through\.

---

### Logical vs Physical

A `pto.vmi` value is **logical** — a flat sequence of `L` lanes of type `T`\.

Its physical backing is `K` hardware vector registers \(256B / 2048\-bit each\):

```Plain Text
K = ⌈ L · bitwidth(T) / 2048 ⌉
```

At `K=1` and full\-width \(no partial lanes\), one `pto.vmi.vreg` maps 1:1 to

one `pto.vreg`\. At `K>1`, the logical value fans out across `K` physical

registers with a layout descriptor \(`#pto.vmi.layout`\) tracking the mapping\.

**Physical constants \(A5 vector pipe\):**

```Plain Text
vector register file : 32 architectural vregs, 256 B (2048 bit) each
predicate file       : 8  architectural pregs, 256 bit each, 1 bit controls 1 byte
VLane                : 32 B sub-lane; 8 VLanes per vreg
E_v = 32 / sizeof(T) : lanes per VLane     (f32 → 8, f16/bf16 → 16, i8 → 32)
```

---

### Type System

#### `!pto.vmi.vreg<L×T>`

Logical vector register\. `L` is the logical lane count; `T` is the element type\.

|T|bits|E\_v \(lanes per physical vreg\)|Legal L|
|---|---|---|---|
|`f32` / `i32` / `ui32` / `si32`|32|64|`1, 2, 4, 8, 64, 128, 256`|
|`f16` / `bf16` / `i16` / `ui16` / `si16`|16|128|`1, 2, 4, 8, 64, 128, 256`|
|`i8` / `ui8` / `si8` / `fp8_e4m3` / `fp8_e5m2`|8|256|`1, 2, 4, 8, 64, 128, 256`|

These are the legal lane counts on the formal public VMI/PTODSL surface\.

PTOAS currently accepts additional positive lane counts in internal VMI IR for

lowering intermediates and compatibility tests; those are not public PTODSL

type\-construction choices\.

- **Full vector**: `L · bitwidth(T) == N · 2048` \(integer multiple of 256B\)\.

- **Compact/partial vector**: `L · bitwidth(T) < 2048` — still backed by one

physical vreg \(256B\); only the low `L` logical slots are valid\. Physical

slots outside the logical value are `pad/undef` and must be masked out\.

**Common logical ↔ physical mappings:**

|Logical type|Byte size|K|Physical vregs|Valid slots per vreg|
|---|---|---|---|---|
|`V<256×f32>`|1024B|4|4|64 f32 each, all valid|
|`V<256×f16>`|512B|2|2|128 f16 each, all valid|
|`V<256×i8>`|256B|1|1|256 i8, all valid|
|`V<128×f32>`|512B|2|2|64 f32 each, all valid|
|`V<64×f16>`|128B|1|1|low 64 f16 valid|
|`V<64×i8>`|64B|1|1|low 64 i8 valid|

### 2\. PyPTO 与 PTOAS 当前接点

```Plain Text
PyPTO Python DSL → PyPTO AST/IR → PyPTO PTO codegen → .pto
                                                ↓
                                     PTOAS MLIR 解析/验证
                                                ↓
                                  当前 PyPTO 集成：EmitC/C++

PTOAS 另有：PTODSL pto.vmi / 手写 .pto → A5 VPTO → LLVM/HiVM
```

`.pto` 是 MLIR 风格的文本 IR。PTOAS 已有 `pto.vmi.*` 逻辑向量、

`pto.vecscope` 区域和 A5 VPTO 的物理化路径；它的 PTODSL Python 入口

`pto.vmi.*` 与 PyPTO `pl.*` 是两个独立前端。PyPTO 目前只通过 PTOAS

的 EmitC/C\+\+ 路径构造 kernel wrapper，不会因生成 `.pto` 就自动接入

A5 VPTO。可复用的是 PyPTO 的 Tensor/Tile 外层语法、Vec tile 驻留，

以及 PTOAS 的 VMI 语义；待接通的是 PyPTO 前端 VMI 值和 op 解析、

TileView 到 UB 指针的映射、VPTO 产物与运行时。VMI/MI 的分层、

完整 55\-op 索引、访存布局模式和真实 `.pto` 测试统一见

\[PTOAS VMI 与 MI 当前实现\]\(11\-ptoas\-vmi\-mi\-current\-state\.md\)。

### 3\. 对照：TileLang 的 Ascend SIMD VF 现状

TileLang 使用 `tilelang.ascend.language` 中的 `with T.SimdVF():` 标记

寄存器级 SIMD 区域。它没有 SIMT 线程维度；区域前后通常用 `T.copy`

在 GM 与 UB（`T.alloc_shared`）之间搬运数据，区域内用

`T.simd.vld` 从 UB 读取向量、计算，再用 `T.simd.vsts` 写回 UB。

SIMD VF 本身不能直接访问 GM。参见 TileLang 的

`examples/ascend/example_vecadd.py`、

`tilelang/ascend/analysis/vf_checker.py`。

#### 3\.1 寄存器抽象与硬件的对应关系

|TileLang 表达|生成的 Ascend C/CCE 表达|抽象边界|
|---|---|---|
|`T.simd.vld` 等产生的 `float32x64`、`bfloat16x128` 向量值|`vector_f32`、`vector_bf16` 等|每个值的**类型宽度**对应完整 2048\-bit（256B）向量寄存器|
|`T.simd.pset(32)` 等产生的 `boolx256`|`vector_bool`|对应 256\-bit（32B）谓词寄存器类型；`32` 指按 32\-bit 元素生成 mask，不是 32\-bit 谓词寄存器|
|`T.simd.alloc_var(dtype)` / `alloc_local(shape, dtype)`|可修改的单个向量变量 / 向量变量数组|前者适合累加，后者可按**寄存器槽位**索引；都不表示可按 lane 直接索引物理寄存器|
|`T.alloc_shared`|`__ubuf__` 缓冲区|UB 存储，不是向量寄存器；`vld/vsts` 显式跨越 UB 与寄存器边界|

`T.simd` 按 `2048 / 元素位宽` 推导 lane 数；Codegen 要求向量值

恰好 2048 bit、谓词值为 `boolx256`，再输出上述 CCE 类型。这是

**寄存器类别及单寄存器宽度对齐**，而非指定物理 V/P 寄存器编号。

Python 变量和 TIR `Bind` 是逻辑值；TileLang 不提供将其绑定到某个

物理寄存器编号的接口，最终分配由下游 CCE/BiSheng 编译处理。

参见 `tilelang/ascend/language/simd.py` 的 `_vreg_lanes`、`alloc_var`

和 `alloc_local`，以及 `src/ascend/codegen/codegen_ascend.cc` 的

`ValidateSimdVectorType`、`PrintType` 和 `VisitStmt_(BindNode)`。

#### 3\.2FP32 → FP16 → FP32 转换

显式 SIMD 写法是连续调用两次 `T.simd.vcvt`。若中间 FP16 只供

后续计算使用，可让它保持为寄存器值，无须经过 UB：

```Python
with T.SimdVF():
    x = T.simd.vld(src_ub[0])
    h = T.simd.vcvt(x, "float16", part=0)
    y = T.simd.vcvt(h, "float32", part=0)
    T.simd.vsts(dst_ub[0], y)
```

`x`/`y` 是 `float32x64`，`h` 是 `float16x128`：寄存器仍为2048 bit，但 64 个 FP32 转换后只占 FP16 结果的偶数位置。`part=0` 对应 `PART_EVEN`，第一次选择写入偶数位置，第二次从偶数位置读取；若选择奇数位置，前后也须保持一致。这里的`part` 是寄存器内的位置选择，不是第二个物理寄存器编号。参见 TileLang 的 `tilelang/ascend/language/simd.py` 中 `vcvt` 的`part_x2` 分派，以及昇腾 CCE 的[FP32→FP16](https://www.hiascend.com/document/detail/en/CANNCommunityEdition/910/API/ascendcopapi/docs/en/api/SIMD-API/C-API/reg/reg_vector/asc_float2half.md)和 [FP16→FP32](https://www.hiascend.com/document/detail/en/CANNCommunityEdition/910/API/ascendcopapi/docs/en/api/SIMD-API/C-API/reg/reg_vector/asc_half2float.md)接口说明。

这条路径在 TileLang IR 中保留两次 `tl.simd.vcvt`；Codegen 发出两次 `simd_inst::vcvt` 调用，由 CCE `vcvt` 实现转换。它没有把往返 cast 规定为可消去的恒等操作。第一次转换默认使用`ROUND_R`（最近偶数）和 `sat=True`（`RS_ENABLE`）；FP16舍入后再转回 FP32 不会恢复原始精度。饱和模式还影响溢出、无穷大及 NaN 的结果，分析数值语义时不能只看 dtype 名称。参见 `src/ascend/codegen/codegen_ascend.cc` 的 `simd_vcvt` 分支和`src/tl_templates/ascend/simd_inst.h` 的 `vcvt` 模板。

若中间结果必须写入 UB，可组合仓库两个单向测试的模式：FP32→FP16 后用 `vsts(..., dist="PK_B32")` 压紧存储；反向读取连续 FP16 时用 `vld(..., dist="UNPK_B16")`，再调用`vcvt(..., "float32", part=0)`。这增加一次 UB 往返，且与上述纯寄存器链的布局不同。现有`testing/ascend/language/test_tilelang_ascend_simdvf_cast.py`分别测试了两个方向，没有直接测试同一函数中的往返链。

### 4\. 正式 VMI API 全表

以下 **55 个 ****`pto.vmi.*`**** op** 按 PTOAS `docs/isa/vmi-isa/10-appendices.md` 的正式索引列全；全部由 `ptodsl/tests/test_vmi_isa_inventory.py` 检查 ODS 定义、生成绑定、PTODSL 包装器及发射测试，并标注 `a5-vpto` 能力。表中省略共同前缀 `pto.vmi.`；“PTODSL 入口”与 op 名不同之处单列。此覆盖**名称与路径**，不保证每种 dtype/shape/mode 都已做硬件验证。

|\#|op|作用 / PTODSL 入口差异|
|---|---|---|
|1|`vload`|UB 到逻辑向量；`pto.vmi.vload`。|
|2|`vstore`|逻辑向量到 UB；`pto.vmi.vstore`。|
|3|`vci`|生成 lane 索引向量。|
|4|`vadd`|加法；Python 的向量加标量也经此入口分派到 `vadds`。|
|5|`vsub`|减法。|
|6|`vmul`|乘法；标量形式分派到 `vmuls`。|
|7|`vdiv`|浮点除法。|
|8|`vmax`|逐 lane 最大值；标量形式分派到 `vmaxs`。|
|9|`vmin`|逐 lane 最小值；标量形式分派到 `vmins`。|
|10|`vabs`|绝对值。|
|11|`vneg`|取负。|
|12|`vrelu`|ReLU。|
|13|`vexp`|指数。|
|14|`vln`|自然对数。|
|15|`vsqrt`|平方根。|
|16|`vand`|按位与。|
|17|`vor`|按位或。|
|18|`vxor`|按位异或。|
|19|`vnot`|按位取反。|
|20|`vshl`|左移；标量 rhs 分派到 `vshls`。|
|21|`vshr`|右移；标量 rhs 分派到 `vshrs`。|
|22|`vadds`|向量加标量；PTODSL 同名入口已弃用，改用 `vadd`。|
|23|`vmuls`|向量乘标量；PTODSL 同名入口已弃用，改用 `vmul`。|
|24|`vmaxs`|向量与标量取最大；PTODSL 同名入口已弃用，改用 `vmax`。|
|25|`vmins`|向量与标量取最小；PTODSL 同名入口已弃用，改用 `vmin`。|
|26|`vshls`|向量左移标量位数；PTODSL 同名入口已弃用，改用 `vshl`。|
|27|`vshrs`|向量右移标量位数；PTODSL 同名入口已弃用，改用 `vshr`。|
|28|`vcmp`|向量比较，结果为逻辑 mask。|
|29|`vcmps`|向量与标量比较，结果为逻辑 mask。|
|30|`vsel`|按 mask 选择。|
|31|`vselr`|按动态 lane 索引选择。|
|32|`vbrc`|标量或组内广播。|
|33|`vcadd`|加法规约。|
|34|`vcmax`|最大值规约。|
|35|`vcmin`|最小值规约。|
|36|`vcvt`|数值类型转换。|
|37|`vinterpret_cast`|保持位串、只重解释类型。|
|38|`vexpdif`|`exp(x - max)` 融合计算。|
|39|`vaxpy`|`alpha*x + acc`。|
|40|`vlrelu`|带标量斜率的 Leaky ReLU。|
|41|`vprelu`|参数化 ReLU。|
|42|`vmull`|宽乘，返回低/高部分。|
|43|`vmula`|乘加。|
|44|`vchist`|累积直方图。|
|45|`vdhist`|分布直方图。|
|46|`vgather`|UB 索引读取。|
|47|`vscatter`|UB 索引写入。|
|48|`create_mask`|前缀/尾部 mask；PTODSL `create_mask(active, size=...)`。|
|49|`create_group_mask`|分组 mask；PTODSL 为 `create_mask(active, size=..., group=...)`。|
|50|`vintlv`|两向量交织。|
|51|`vdintlv`|两向量解交织。|
|52|`vaddc`|32 位整数加法及进位，返回 `(value, carry)`。|
|53|`vsubc`|32 位整数减法及未借位标志，返回 `(value, carry)`。|
|54|`vaddcs`|带输入进位的 32 位整数加法。|
|55|`vsubcs`|带输入进位/借位状态的 32 位整数减法。|

完整逐 op 类型、对齐、模式、发射约束见 PTOAS `docs/isa/vmi-isa/01-load-store.md` 至 `10-appendices.md` 和 `ptodsl/docs/user_guide/14-vmi-virtual-instruction-set.md`。PTOAS 的 ODS 还定义 `masked_load`、`channel_split/merge`、`vunzip/vzip`、`ensure_layout` 等内部/扩展操作；它们**不计入**上表的 55 条正式索引。

## 附录：

### AscendC VF 优化指南

https://asc\.gitcode\.com/guide/operator\_practice/simd\_operator\_optimization/vector\_compute/vf\_optimization/vf\_loop\_optimization\.html

### REG 接口边界值

https://asc\.gitcode\.com/api/appendix/reg\_vector\_compute\_interface\_boundary\_value\_summary\.html

### REG MERGEMODE支持情况汇总

https://asc\.gitcode\.com/api/appendix/reg\_vector\_compute\_interface\_mergemode\_support\_summary\.html

### REG 精度支持情况汇总

https://asc\.gitcode\.com/api/appendix/reg\_vector\_compute\_interface\_precision\_standard\_summary\.html

### REG 单双发指令情况汇总

https://asc\.gitcode\.com/api/appendix/reg\_vector\_compute\_interface\_instruction\_single\_dual\_issue\_summary\.html

### Reg矢量计算接口理论性能汇总

https://asc\.gitcode\.com/api/appendix/reg\_vector\_compute\_interface\_theoretical\_perf\_summary\.html
