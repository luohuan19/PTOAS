# VPTO 当前能力梳理

[返回 A5 文档目录](README.md)

- 核查日期：2026-10-27。
- 源码基线：`235f381f31ac`。
- 核查方式：阅读当前源码、正向与负向回归用例，以及已有验证记录；本次未重新执行编译、模拟器或 NPU 测试。

## 1. 能力定位与判定口径

当前 VPTO 已具备 A5 上的 AIV、AIC、混合 kernel 编译链路，以及部分 A2/A3 向量能力。当前源码还支持 EmitC/VPTO 混合后端编译、Pipe/Cube/Print 的 C++ Bridge，以及低层 GM↔GM SDMA 操作；功能覆盖尚未与 EmitC 完全对齐。

对于“大部分逻辑使用 EmitC、少量计算使用 VPTO”的需求，可采用混合后端：将主体和局部计算组织成不同的子模块，通过设备函数调用连接，分别编译后链接成一个 fatobj。第 4 节给出具体用例。

对于同一目标架构、同一套合法 PTO 输入，覆盖 EmitC 已支持的 kernel 场景可以作为 VPTO 的功能完整性目标。本文记录当前实现状态，不将这一目标视为已经完成的能力。

本文区分以下证据：

| 证据 | 能说明什么 | 不能直接推出什么 |
|---|---|---|
| 源码实现 | 存在对应表示、转换或产物生成路径 | 所有输入组合均可编译或运行正确 |
| 编译回归用例 | 仓库对指定输入的展开、LLVM 发射或诊断有明确预期 | 该用例已完成设备运行验证 |
| 模拟器或硬件验证用例 | 已具备执行和数值比对入口 | 本次核查已执行或通过这些用例 |
| 历史通过记录 | 特定环境和代码状态下曾验证通过 | 当前基线和其他环境必然全部通过 |

### 1.1 三种组合方式

| 方式 | 划分边界 | 解决的问题 |
|---|---|---|
| 混合 kernel（Mix-kernel） | 同一逻辑 kernel 的 Cube/Vector section | 组织 AIC 和 AIV 的物理执行分工 |
| 混合后端（Mix-backend） | 不同 `pto.backend` 的子 `module` | EmitC 编译主体，VPTO 编译选定函数，再链接成统一产物 |
| VPTO C++ Bridge | VPTO 模块中已注册、被策略选中的操作 | 在 VPTO 编译中生成设备侧 C++ wrapper，复用 PTO-ISA/CANN 实现 |

混合后端不受 C++ Bridge 的操作列表限制。加法等普通操作可以放入 VPTO 子模块；Bridge 策略只决定指定操作是否改走 C++ wrapper。

## 2. 架构覆盖

| 目标架构 | AIV / Vector | AIC / Cube | 当前边界 |
|---|---|---|---|
| A5 | 已有 SIMD、SIMT 及相应访存实现 | 已有 MAD、MAD_MX、MTE、矩阵结果写回及部分 TileOp 展开 | 支持纯 Vector、纯 Cube 和 Cube/Vector 混合 kernel；仍需按算子、dtype、shape、layout 核对 |
| A2/A3 | 共用 C220 UB 路径，覆盖部分搬运和向量 TileOp | 已有 C220 Cube 目标选择基础设施 | 常规专用 TileOp 路径尚未形成完整 AIC 矩阵计算覆盖；A5 Bridge 验证记录也不能直接推广到 A2/A3 |

A2/A3 的专用 TileOp 路径执行 UB 转换后返回，不进入 A5 的后续 TileOp 模板展开。当前已确认的能力主要集中在 AIV。

## 3. 计算、访存和 kernel 表达能力

除单独注明外，下表主要描述 A5 的实现和回归覆盖。

| 能力 | 当前具备的内容 | 范围与边界 |
|---|---|---|
| 输入与编程形式 | PTO 文本、PTOBC；TileOp、VMI、低层 MI，以及混合表达 | 具体组合需满足目标架构、执行域和 IR 校验规则 |
| 标量与标准向量运算 | `pto.addi`、`pto.addf` 等算术操作可转换为标准算术和 LLVM IR | 普通加法不需要 C++ Bridge；具体名称和类型以操作定义为准 |
| AIV SIMD 计算 | 算术、比较、选择、位运算、移位、归约、广播、类型转换、部分排序指令 | 各指令的 dtype、布局、掩码和数值模式支持不同 |
| AIV SIMD 访存 | UB 与寄存器间搬运、对齐和非对齐访问、Gather/Scatter、谓词访存、地址后更新 | 部分访问模式仍有类型、对齐和硬件支持限制 |
| SIMT 计算与访存 | 线程和执行配置、数学运算、GM/UB 访存、原子操作、Shuffle/Vote/Redux、线程同步 | 以具体操作的类型和执行域约束为准 |
| SIMT 状态与分段执行 | keep/resume、persistent fragment 的分析和物化 | 跨 entry 的 keep/resume 状态不能由当前模拟器完整建模，需要硬件验证 |
| AIC Cube 计算 | MAD、MAD_MX、累加与偏置、部分低精度矩阵计算和 MX scale 处理 | 已有 `TMATMUL` 展开与 Cube LLVM 发射回归；不代表所有矩阵类型与布局组合均已验证 |
| 数据搬运 | GM↔UB、GM→L1、L1→L0A/L0B、L0C 结果写回，以及部分格式转换和填充 | 各路径分别约束类型、形状、步长、地址空间和对齐 |
| 同步 | Event set/wait、动态事件、buffer-id、Barrier、MemBar；可选自动同步插入 | 支持范围依架构、流水线、执行域和所选同步模式而定；不等于自动补全所有跨核协议 |
| 本地跨核 Pipe | `initialize_l2l_pipe`、`tpush`、`tpop`、`tfree` 可通过默认启用的 Bridge 实现 | Bridge 当前覆盖 local Pipe 的 Acc→Vec 和 Vec→Mat 方向，详见第 5 节 |
| 设备打印 | `pto.print`、`pto.tprint` 通过 Print Bridge 输出标量和 Tile 文本 | 默认启用；依赖 DebugTunnel，受格式、Tile 布局与参数约定限制 |
| 条件断言与二进制转储 | `pto.assert` 请求条件 trap；`pto.tdump` 将 Tile 元数据和有效数据写入调用方提供的 GM | `tdump` 为 VPTO 能力；不能把 assertion/trap 视为可靠终止或阻止后续写入的保证 |
| 低层 GM↔GM SDMA | `pto.session_init` 从 Host 配置构造 session，`pto.sdma_gm_gm` 发起连续字节拷贝并返回完成记录地址 | 存在 A5、A2/A3 对应处理；要求普通 AICore kernel 上下文，不能放在 SIMT 域内；高层 `pto.comm.*` 的状态见第 8 节 |
| 混合 kernel | 一个 kernel 包含 Cube/Vector section，拆分成 AIC/AIV 子模块并生成产物 | 已有拆分及规范化回归；跨核通信仍需正确表达 |
| 控制流与函数 | 循环、条件分支、标量计算、局部数组与结构体、函数内联及外部符号调用 | 调用边界需满足 ABI 和类型约束 |

A2/A3 当前专用向量路径覆盖的主要算子族如下：

| 算子族 | 示例 |
|---|---|
| GM↔UB 搬运 | `tload`、`tstore` |
| 双输入元素运算 | `tadd`、`tsub`、`tmul`、`tdiv`、`tmax`、`tmin`、`taddrelu` |
| 一元运算 | `tabs`、`trelu`、`tneg`、`texp`、`tlog`、`tsqrt`、`trsqrt`、`trecip` |
| 标量运算 | `tadds`、`tmuls`、`tmaxs`、`tmins` |
| 位运算和移位 | `tand`、`tor`、`txor`、`tnot`、`tshls`、`tshrs` |
| Gather | `tgather`、`tgatherb` |

上述列表表示算子族已有实现，不表示任意 dtype、动态形状和步长均被支持。A2/A3 UB 路径要求静态 valid shape；DMA 等路径还有各自的步长约束。

## 4. EmitC 主体调用少量 VPTO 函数

### 4.1 组织与编译方式

将主要逻辑放在 `pto.backend = "emitc"` 子模块，将需要 VPTO 的计算段提取为函数，放在 `pto.backend = "vpto"` 子模块。当前选择后端的边界是子模块，而不是在同一函数中给单条操作设置后端。

| 位置 | 函数形式 | 作用 |
|---|---|---|
| EmitC 调用方 | 没有函数体的 `func.func private @helper(...)` | 声明外部设备函数 |
| EmitC 调用方 | `func.call @helper(...)` | 从主体调用局部计算 |
| VPTO 被调用方 | 有函数体的 `func.func public @helper(...)` | 提供需要导出的设备函数实现 |

PTODSL 也可以通过 `@pto.jit(backend="emitc")` 入口调用 `@pto.jit(entry=False, backend="vpto")` 编译单元来表达这种结构。当前 PTODSL 跨子模块调用接口限定于兼容 C ABI 的 GM 指针和标量，不直接传递 Tile 对象。

编译时不传全局 `--pto-backend`，让 PTOAS 使用子模块的 `pto.backend` 属性：

```bash
ptoas mixed.pto --pto-arch=a5 -o mixed.fatobj.o
```

显式的全局 `--pto-backend=emitc` 或 `--pto-backend=vpto` 会覆盖子模块选择。混合后端要求显式输出文件；不能在生成混合 fatobj 的同一次调用中使用 `--emit-vpto` 等调试 IR 输出模式，调试时应分别检查各编译单元。

编译过程为：

```text
EmitC 子模块 → CCE C++ → Bisheng → 子 fatobj
VPTO 子模块  → LLVM IR → Bisheng → 子 fatobj
                                ↓
                    统一链接并解析设备函数符号
                                ↓
                          最终 fatobj
```

两个后端不会跨子模块内联或统一优化函数体。函数接口、执行域、同步和共享存储的使用仍需满足各自的约束。

### 4.2 已有向量加法用例

完整输入见 [mixed-external-vadd/kernel.pto](../../../test/vpto/cases/micro-op/backend/mixed-external-vadd/kernel.pto)。

| 阶段 | 用例中的行为 |
|---|---|
| Host | 只启动一次 `mixed_external_vadd_kernel`，传入两个输入和一个输出的 GM 指针 |
| EmitC 入口 | 声明并调用 `external_vadd` |
| VPTO 函数 | 将输入搬到 UB，循环执行向量加法，再将结果搬回 GM；搬运、同步和计算均在该函数中 |
| 数据规模 | 1024 个 FP32 元素，每轮处理 64 个，共 16 轮 |
| 结果检查 | Host 同步后拷回，与 NumPy 的 `lhs + rhs` 比较，`atol` 和 `rtol` 均为 `1e-4` |

这是验证跨后端调用链的最小用例。它的 EmitC 入口只有一次函数调用，不能据此声称该用例的大部分工作量由 EmitC 完成；实际应用可以在 EmitC 调用方保留更多主体逻辑，只提取选定计算函数。

在已配置的编译环境中，可用以下命令编译原始用例：

```bash
ptoas --pto-arch=a5 \
  test/vpto/cases/micro-op/backend/mixed-external-vadd/kernel.pto \
  -o /tmp/mixed-vadd.fatobj.o
```

该 case 的 [ptoas.flags](../../../test/vpto/cases/micro-op/backend/mixed-external-vadd/ptoas.flags) 仅指定 `--pto-arch a5`，避免驱动脚本默认的全局后端参数覆盖子模块选择。

## 5. VPTO C++ Bridge

### 5.1 当前可路由的操作

Bridge 在 VPTO 模块内部按操作选择 C++ wrapper；EmitC/VPTO 混合后端按子模块选择编译器路径。两者可以组合，但用途和边界不同。

| Family | PTO 操作名 | 内置默认策略 |
|---|---|---|
| Pipe | `pto.initialize_l2l_pipe`、`pto.tpush`、`pto.tpop`、`pto.tfree` | 整个 family 开启 |
| Cube | `pto.tmatmul`、`pto.tmatmul.acc` | 均关闭，需要逐项选择 |
| Print | `pto.print`、`pto.tprint` | 均开启 |

以上是 Bridge 的 8 个可选择操作名，不是 VPTO 的全部操作列表。`pto.addi`、`pto.addf`、`pto.tadd`、`pto.vadd` 等加法操作可以继续使用普通 VPTO 路径，无需加入 Bridge 策略。

### 5.2 策略与编译

以下策略启用 Pipe、两种 Cube 操作和两种 Print 操作，可保存为 `bridge-policy.yaml`：

```yaml
version: 1
families:
  pipe:
    enabled: true
  cube:
    enabled_ops: [pto.tmatmul, pto.tmatmul.acc]
  print:
    enabled_ops: [pto.print, pto.tprint]
```

```bash
ptoas kernel.pto --pto-arch=a5 --pto-backend=vpto \
  --vpto-bridge-policy=bridge-policy.yaml -o kernel.fatobj.o
```

- 不指定文件时使用内置默认策略；自定义文件替换整份默认策略，省略的 family 不继承默认启用状态。
- `pipe` 使用 `enabled` 整体控制；`cube`、`print` 使用 `enabled_ops` 逐项选择。未知字段、操作和重复项会报错。
- 未选中的 Cube 操作继续使用常规 VPTO 展开；选中后不满足 Bridge 约束会报错，不自动回退。
- `pto.print`、`pto.tprint` 在 VPTO 中没有非 Bridge 实现，输入包含打印时必须在策略中保留对应操作。
- 策略只能选择已注册的操作，不能指定任意 C++ 源码、include 或外部 ABI。

Wrapper 由 PTOAS 生成，使用 CANN/Bisheng 和兼容的 PTO-ISA 头文件编译，再通过配套 `llvm-link` 等步骤合入设备代码。单独导出 LLVM 文本时可以保留待链接的 wrapper 声明，因此文本导出成功不能替代 wrapper 编译和链接验证。

### 5.3 主要约束与验证范围

| Family | 当前主要约束 |
|---|---|
| Cube | 要求 Cube 执行上下文、二维静态形状，以及直接来自已规划地址 `alloc_tile` 的操作数；view/subview、MX scale、FP4/FP8 等不属于当前 Cube Bridge 的直接路由范围 |
| Pipe | 仅覆盖 local Pipe 的 `dir_mask=1`（Acc→Vec）或 `2`（Vec→Mat），不支持 `acc_push_epilogue`；push/pop/free 的 split、地址和生命周期必须一致 |
| Print | 标量格式必须匹配类型；`tprint` 要求二维 ND row-major Vec tile、已规划地址和静态有效区域，接受 f16/bf16/f32 和 signless i8/i16/i32，不支持 `tmp` 或显式 `printFormat` |

`--vpto-print-dtdata=implicit` 为默认约定，由运行时投递隐藏的 DebugTunnel 参数；`explicit` 使用 Host 传入的首个 GM 指针，仓库模拟器用例采用此方式。

仓库 Bridge 设计文档记录了 A5 模拟器上 Pipe 消费、Cube 矩阵乘和 Print 用例通过 compare 的结果；这属于已有验证记录。本次未重跑这些用例，真实 NPU，尤其隐式 DebugTunnel 参数路径，仍需单独验证。

## 6. 编译优化能力

| 方面 | 当前能力 | 开关或范围 |
|---|---|---|
| VMI 转换 | 布局与掩码粒度分配、布局约束和代价规划、布局转换消除、重物化、VMI→VPTO、访存流融合 | 进入 VPTO 的统一语义处理流程；支持情况以具体操作和布局为准 |
| 通用优化 | 常量传播、CSE、循环不变量外提、循环展开、地址和掩码简化、转换和归约优化 | 是否应用取决于输入形式、优化前提和配置 |
| Tile 融合 | 融合区域、循环融合、冗余读写消除、可选 cost model | A5；`--enable-op-fusion=true` 显式开启，默认关闭 |
| 指令调度 | 依赖分析、调度、寄存器压力分析、有界重物化和再次调度 | 当前调度 pass 运行于 A5 Vector，跳过规范化后的 Cube 子模块 |
| 调度分析输出 | `off`、`analyze`、`on` 三种模式及 trace | A5 默认 `on`；默认随之启用有界重物化，可分别关闭 |
| 内存与地址处理 | 局部内存规划、地址空间规范化、地址后更新优化 | 受构建 level 和输入地址配置约束 |
| VecScope 内存屏障 | 对 `vecscope` / `strict_vecscope` 内的 UB 向量和标量访问分析依赖，插入并消除冗余 MemBar | `--enable-vecscope-mem-bar` 默认开启；可选 `--enable-vecscope-mem-bar-all` 保守模式默认关闭；不承担 GM 或 sibling VecScope 间同步 |

VPTO 调度 pass 与 Bisheng 自身的向量调度策略是两个不同阶段。后者可通过 `--bisheng-vec-misched=auto|on|off` 控制；默认 `auto` 会在适用条件下重试并比较 VF 栈使用情况。

VMI 布局规划具有搜索预算，可通过 `--vmi-layout-max-frontier-entries` 和 `--vmi-layout-max-transitions` 调整；两者均要求正值。VecScope 的自动屏障分析有地址和依赖可建模范围，不能据此承诺为任意未知地址或全部同步场景自动兜底。 当前普通模式最终得到 `Unknown` 时不会仅因此补屏障；动态循环边界另有专门处理，详见 [UB alias 与动态偏移限制](ptoas-a5-capabilities-guide.html#ub-alias-limits)。

## 7. 编译链路与工程能力

VPTO 的常规设备编译路径为：

```text
PTO 文本 / PTOBC / 前端生成的 PTO IR
    → 前端处理与局部内存规划（按输入需要）
    → 按策略生成 Bridge 调用与 C++ wrapper（如有）
    → 常规 TileOp 展开及 VMI/MI 处理
    → VPTO 优化与合法性校验
    → 按 AIC/AIV 生成 LLVM 模块
    → 编译并合入设备 wrapper（如有）
    → Bisheng 编译设备目标文件
    → 合并设备对象并打包 Host stub
    → 可供 Host 链接的 fatobj
```

不同输入可跳过不需要的前端或展开步骤。A2/A3 使用其专用 UB 转换路径，不能据此编译链路图推定具有 A5 的全部能力。

| 方面 | 当前能力 |
|---|---|
| IR 输出 | `--emit-vpto` 输出优化后的 VPTO IR；`--emit-vpto-llvm-ir` 输出 LLVM IR |
| 设备产物 | 常规 VPTO 模式调用 Bisheng 并输出 fatobj，要求显式提供 `-o` 文件路径 |
| Host 接入 | 生成与设备入口对应的 Host stub，供后续 Host 链接和调用 |
| 混合后端 | EmitC/VPTO 子模块分别编译后链接成统一 fatobj；这与 AIC/AIV 混合 kernel 是两个维度 |
| C++ Bridge | Pipe/Cube/Print 按策略生成设备侧 wrapper，编译和链接由 PTOAS 完成 |
| 诊断 | 输入与发射阶段校验、IR dump、调度分析和 trace |
| 工具链适配 | Bisheng 调度策略选择、VF_SIMT size 校验与修复 |

以已安装 PTOAS 并加载兼容 CANN 环境后的 A5 调用为例：

```bash
# 编译为可供 Host 链接的 fatobj
ptoas kernel.pto --pto-arch=a5 --pto-backend=vpto -o kernel.fatobj.o

# 输出 VPTO IR
ptoas kernel.pto --pto-arch=a5 --pto-backend=vpto --emit-vpto -o kernel.vpto

# 输出 LLVM IR
ptoas kernel.pto --pto-arch=a5 --pto-backend=vpto --emit-vpto-llvm-ir -o kernel.ll
```

CLI 在未通过参数或模块属性选择后端时，默认仍为 EmitC。当前 A5 LLVM 发射入口要求 CANN 9.0.0-beta.2 或更新版本。

## 8. 已知缺口与验证边界

| 项目 | 当前状态 |
|---|---|
| EmitC 功能对齐 | 尚未完成，不能将“支持 VPTO 后端”理解为全部 EmitC kernel 均可直接切换 |
| A2/A3 AIC TileOp | 常规专用路径尚未形成完整矩阵计算覆盖；Cube 目标选择和 A5 Bridge 验证记录不能替代 A2/A3 端到端覆盖 |
| 混合后端粒度 | 支持子模块和函数调用边界；同一函数内逐操作切换后端需要先提取计算段，没有任意操作自动切换的路由策略 |
| Bridge 覆盖 | 仅支持注册并选中的操作；Cube Bridge 的类型与布局边界独立于原生 MAD/MAD_MX 的支持范围 |
| 高层异步 session | `pto.comm.build_async_session`、`tput_async`、`tget_async`、异步事件等待与查询仍被 VPTO 发射校验拒绝，目前要求 EmitC；这不否定低层 `session_init` / `sdma_gm_gm` 已有实现 |
| 低层 SDMA 完成与可见性 | 普通 post 返回完成记录地址，kernel 返回不等于拷贝完成；本操作组尚无消费该记录的 wait op。A5 `{soft_put}` 路径在操作返回前完成拷贝并返回空记录，A2/A3 忽略此属性并保持 post 行为 |
| 部分高层 CMO/fence | `pto.cmo.cacheinvalid`、`pto.fence.barrier_all` 尚不能直接发射，需要先转换为支持的低层操作 |
| dtype、shape、layout 组合 | 需要按具体算子核对；类型存在或单条编译回归存在，均不代表所有组合已验证 |
| 模拟器覆盖 | 部分行为需要实机验证，例如跨 SIMT entry 的 keep/resume 状态 |
| Debug 与 Print | assertion/trap 不承诺可靠终止或阻止后续写入；dump 不保证异常退出后仍完整，Print 的真实 NPU 参数投递仍需目标环境验证 |
| 当前验证结论 | 本次仅源码与用例核查，没有新的构建、模拟器运行或 NPU 数值验证结果 |

仓库保留了 A3 上 392 项元素级硬件端到端验证通过的历史记录。A2 与 A3 共用转换实现，但这份 A3 记录不能替代独立 A2 实机验证，也不能代表当前基线已重新通过全部用例。

## 9. 源码与测试依据

| 内容 | 主要依据 |
|---|---|
| 输入加载、后端选择与编译任务 | [driver.cpp](../../../tools/ptoas/driver.cpp)、[后端与混合产物说明](../../designs/module-backend-attr-and-mixed-fatobj.md) |
| EmitC 主体与 VPTO 函数组合 | [混合编译流程](../../designs/mix-kernel-mix-backend-compile-flow.md)、[向量加法输入](../../../test/vpto/cases/micro-op/backend/mixed-external-vadd/kernel.pto)、[Host 启动](../../../test/vpto/cases/micro-op/backend/mixed-external-vadd/launch.cpp)、[数值比较](../../../test/vpto/cases/micro-op/backend/mixed-external-vadd/compare.py) |
| TileOp、VMI、优化和编译输出 | [ptoas_pipeline.cpp](../../../tools/ptoas/ptoas_pipeline.cpp) |
| A2/A3 专用向量路径 | [LowerPTOToUBufOps.cpp](../../../lib/PTO/Transforms/Lowering/LowerPTOToUBufOps.cpp)、[算子覆盖说明](../../designs/a2a3-vpto-elementwise-impl.md) |
| AIC/AIV LLVM 目标选择 | [VPTOCANN900LLVMEmitterPipeline.cpp](../../../lib/PTO/Transforms/VPTO/VPTOCANN900LLVMEmitterPipeline.cpp) |
| A5 Cube TileOp 展开 | [TMATMUL 回归](../../../test/lit/vpto/cube/expand_tile_op_tilelang_tmatmul.pto) |
| Cube 低精度路径 | [MAD_MX 编译回归](../../../test/lit/vpto/cube/mad_mx_low_precision_vpto_llvm.pto)、[MX scale 运行验证用例](../../../test/vpto/cases/micro-op/cube-matmul/mad_mx_e8m0_stage_abi/kernel.py) |
| AIC/AIV 混合 kernel | [混合 section 回归](../../../test/lit/vpto/section_sugar_mixed.pto) |
| SIMT 与持久状态 | [SIMT 编译回归](../../../test/lit/vpto/simt_lowlevel_intrinsics_vpto_llvm.pto)、[keep/resume 硬件验证说明](../../../test/vpto/cases/onboard-only/simt-keep-resume-register-swap/README.md) |
| 普通加法编译路径 | [标量与标准向量回归](../../../test/lit/vpto/scalar_frontend_to_standard.pto)、[寄存器加法混合调用用例](../../../test/vpto/cases/micro-op/backend/mixed-external-vadd/kernel.pto) |
| Bridge 范围、默认策略和验证记录 | [技术设计](../../designs/vpto-bridge-reimplementation.md)、[注册表](../../../lib/PTO/IR/VPTOBridgeRegistry.cpp)、[策略解析](../../../lib/PTO/Transforms/VPTOBridgeWhitelist.cpp)、[类型与操作转换](../../../lib/PTO/Transforms/PTOLowerBridgeOps.cpp) |
| Bridge 运行验证用例 | [Cube 矩阵乘](../../../test/vpto/cases/kernels/cube-tmatmul-bridge/kernel.pto)、[Print](../../../test/vpto/cases/kernels/print-bridge/kernel.pto)、[Pipe 生命周期](../../../test/lit/vpto/bridge/route_policy_cli.pto) |
| VecScope 自动屏障 | [分析范围与限制](../../designs/vecscope-membar-design.md)、[分析实现](../../../lib/PTO/Transforms/VecScopeMemBar/VecScopeMemBarAnalysis.cpp)、[插入流程](../../../tools/ptoas/ptoas_pipeline.cpp) |
| 调度范围与默认行为 | [VPTOSchedulerPass.cpp](../../../lib/PTO/Transforms/VPTOScheduler/VPTOSchedulerPass.cpp)、[CLI 调度回归](../../../test/lit/vpto/vpto_scheduler_cli.pto) |
| 目标文件、Host stub 与工具链策略 | [ObjectEmission.cpp](../../../tools/ptoas/ObjectEmission.cpp)、[VPTOHostStubEmission.cpp](../../../tools/ptoas/VPTOHostStubEmission.cpp)、[BishengScheduler.cpp](../../../tools/ptoas/BishengScheduler.cpp) |
| CANN 发射版本约束 | [VPTOLLVMEmitterDispatcher.cpp](../../../lib/PTO/Transforms/VPTO/VPTOLLVMEmitterDispatcher.cpp) |
| 高层异步通信缺口 | [VPTO 发射校验](../../../lib/PTO/Transforms/Passes/PTOValidateVPTOIR.cpp)、[async session 负向回归](../../../test/lit/vpto/async_session_unsupported.pto) |
| 低层 SDMA | [语义与约束](../../isa/micro-isa/19-async-comm.md)、[展开实现](../../../lib/PTO/Transforms/VPTO/VPTOExpandAsyncComm.cpp)、[A5 soft_put 回归](../../../test/lit/vpto/async_sdma_gm_gm_a5_soft_put_session.pto)、[运行用例](../../../test/vpto/cases/async-comm/sdma_gm_gm.py) |
| 断言与二进制转储 | [接口与终止限制](../../designs/debug-ops-scope.md)、[转储实现](../../../lib/PTO/Transforms/Lowering/LowerPTOTensorDump.cpp)、[运行验证与失败探针](../../../test/vpto/cases/kernels/tdump-vec/README.md) |
| CMO/fence 缺口 | [CMO 负向回归](../../../test/lit/vpto/memory_consistency_cmo_unsupported.pto)、[fence 负向回归](../../../test/lit/vpto/memory_consistency_fence_unsupported.pto) |
| A3 历史硬件通过记录 | [元素级验证记录](../../designs/a2a3-vpto-elementwise-pr-summary.md) |
