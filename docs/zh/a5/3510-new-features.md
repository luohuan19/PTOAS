<!-- markdownlint-disable MD013 MD060 -->
# 3510 新增硬件能力、架构变更与软件现状

> 本文为导入的参考整理稿，保留原核查日期、版本与判断口径。部分软件结论与本仓库后续核查不同；PTOAS 能力及限制请参见 [A5 能力图解手册](ptoas-a5-capabilities-guide.html)。原稿引用但本仓库未收录的材料以文件名标识，不能作为本地可打开的文档。

本文依次介绍：① 新增硬件能力及场景；② 相对 2201 的能力增强、移除与行为变化；③ PTOAS/PyPTO 当前开放与使用情况。硬件能力和软件接口覆盖分别判断。

原文总览中的 19 项及另外两项编程模型共用编号 1–21，本文保留这些编号作为跨表索引，不按编号决定章节顺序。原文归入“新特性”的增强项与布局变化放在第二章；SIMT 等编程模型放在第一章，并明确其属于执行模型。

## 一、3510 新增硬件能力

本章先介绍硬件提供了什么及其用途；已有能力的增强与限制见第二章，软件开放程度见第三章。具体算子例子是对原文能力的应用归纳。

### 1.1 新增数据通路与搬运能力

| 原文编号 | 新增能力 | 能力说明 | 使用场景 |
| --- | --- | --- | --- |
| 1 | UB→L1 | Vector 数据直接送入 Cube 侧 L1，省去 GM 中转 | 量化、激活或预处理后接 GEMM |
| 2 | L1→UB | L1 驻留数据直接搬到 Vector 侧 UB | L1 数据的向量处理 |
| 3 | L0C→UB | Cube 累加结果直接送入 Vector，单向通路 | GEMM 后接归一化、激活、Softmax |
| 4 | ND-DMA | 配置多维 shape、stride；在适用参数组合下完成复杂 GM→UB 搬入 | Slice、非连续搬入、Padding、Transpose、Broadcast |
| 5 | DMA loop 模式 | 一次配置多次规则搬运，减少逐条下发 | 等间隔块、交错数据、分块矩阵 |
| 6 | MX 数据及 scale 搬运与计算 | 同步搬运低比特数据及分块 scale，执行 MX 矩阵乘加 | MXFP8/MXFP4 GEMM |
| 8 | Fixpipe NZ2DN | L0C→GM/L1/UB 搬出时随路将 NZ 转为列主序 DN | 直接生成后续算子需要的列主序输出 |
| 10 | Stride 卷积矩阵搬运 | 配置输出矩阵 K 轴方向偏移及相关步长 | 卷积矩阵块的搬入 |

### 1.2 新增执行、存储与同步机制

| 原文编号 | 新增能力 | 能力说明 | 使用场景 |
| --- | --- | --- | --- |
| 14 | Regbase | AIV 直接操作向量寄存器，中间结果可驻留寄存器 | GELU、Softmax 等融合表达式，减少 UB 往返 |
| 16 | DCI | 使数据缓存失效而不回写脏行；须满足对应内存顺序及使用条件 | 减少算子退出时的 Clean 开销 |
| 17 | SSBuffer | 独立地址空间、总容量 3 KB，32 字节对齐及访问粒度 | 同一任务 AIC/AIV 之间传递控制、地址及少量标量 |
| 18 | Mutex | 锁定/释放流水建立依赖，提供不同于事件配对的同步机制 | MTE2/Vector/MTE3 异步协作及双缓冲 |
| 19 | 核间同步 mode 4 | AIC 可与单个 AIV 同步 | AIV0/AIV1 分别触发 AIC，减少冗余等待 |
| 20 | SIMT 执行模型 | 线程粒度计算、分支、索引及原子操作 | 不规则访问和控制流 |
| 21 | SIMD/SIMT 混合模型 | 同一 Kernel 协同使用两种计算方式 | 规则部分 SIMD、不规则部分 SIMT |
| 补充 | NDDMA Cache | 为 ND-DMA 提供对应的新缓存类型 | 多维搬运的数据复用及一致性管理 |

## 二、相对 2201 的修改点

本章综合新特性说明与其引用的《2201到3510架构变更》。增强、硬件移除、性能风险与参数语义变化分别列出；不能把所有变化统一视为性能提升或降低。

### 2.1 既有能力增强

| 原文编号/类别 | 2201 / 原有方式 | 3510 变化 | 场景与迁移关注点 |
| --- | --- | --- | --- |
| 7：2D 随路转置 | 以拼接后的方块矩阵配置，较难描述分形级间隔 | 按 512 字节分形配置位置、源 stride、`srcFracGap` | 非连续分形 B 搬入 L0B；重新映射旧参数 |
| 9：2D 矩阵搬运 | 原有 2D 搬运能力与参数 | 2D V2 提供更灵活搬运与更广类型覆盖 | 重新检查起点、step、stride 和对齐；并非全部参数保持兼容 |
| 11：矩阵输入类型 | 原有矩阵类型范围 | 扩展 FP8、HiF8 及带 scale 的 MX 格式 | 低精度 GEMM；矩阵数据、scale 与累加类型须分别核对 |
| 13：基础向量 API | 原有张量/标量形式及运算族 | 扩展 LocalTensor 单元素标量、Select/Neg/Prelu、Mull、AbsSub/ExpSub、MulsCast、Truncate、交织、张量移位量、Ands/Ors | 动态标量、宽乘双结果、重排与复合表达式；API 扩展不等于每项对应一条新硬件指令 |
| 15：原子接口 | 原有原子操作接口 | Add/Max/Min/Exch/CAS 标量原子读改写，可返回旧值 | 并发计数、直方图、索引分配和共享更新 |
| 架构规格 | 原有 AI Core 数量；UB 为 16 group × 3 bank × 4 KB，即 192 KB | AI Core 数量增加；UB 总容量 256 KB，8 group × 2 bank × 16 KB | 产品实际核数需匹配；原有 bank 避冲突布局需重评估 |
| 数据类型 | 原有类型集合 | 新增/扩展 FP4 E2M1/E1M2 打包类型、HiF8、FP8 E4M3/E5M2/E8M0 | 存储、转换、矩阵计算支持范围分别检查 |
| DCache 控制 | 原有 DCCI 接口 | 新增 `CACHELINE_ATOMIC` 参数 | 原子访问期间 DCache/GM 一致性 |
| UB 预留 | Ascend C 接口预留 8 KB | 3510 Ascend C 接口预留 2 KB，提供空间控制选项 | 总容量、DCache 划分、接口与 VF 栈预留分别计算，不直接当作可分配容量 |

### 2.2 能力移除、收缩与性能风险

以下变化来自伴随的架构变更文档；不属于“PyPTO 尚未开放”造成的限制。

| 类别 | 3510 移除或限制 | 影响 | 原文迁移方向或注意事项 |
| --- | --- | --- | --- |
| 数据通路 | 删除 L1→GM 直达通路 | DataCopy/DumpTensor 不能继续直接搬出 | 原文给出单位矩阵 MMAD→L0C→Fixpipe→GM 的替代方式；不能当作等价开销 |
| 数据通路 | 删除 GM→L0A/L0B 直达通路 | 旧 LoadData 直达路径失效 | 拆为 GM→L1→L0A/L0B |
| 转置 | L1→L0A 不再支持原 transpose 能力 | 原有 A 矩阵转置搬入需调整 | 结合 L0A NZ 布局重新设计，不将 L0B 转置增强推广到 L0A |
| 初始化 | 删除 L0A/L0B 初始化相关硬件指令 | Fill 不能直接初始化这两个位置 | 调整初始化位置及搬运流程 |
| Cube 类型 | 不支持 s4 矩阵计算 | 原 int4b_t MMAD 不能直接执行 | 原文要求先转 int8；不能与浮点 MXFP4 混同 |
| 稀疏计算 | 不支持原 4:2 稀疏矩阵计算 | 旧 LoadDataWithSparse/MmadWithSparse 路径不适用 | 重新设计表示与计算；Vector 数据处理不等于恢复硬件稀疏计算 |
| 浮点数值 | 硬件不支持 Subnormal | 依赖软件仿真/config 或 FTZ 策略 | Ln/Sqrt/Rsqrt/Div/Reciprocal/Exp 等核对精度与开销；`--cce-ftz` 默认为 true |
| Vector 性能 | Membase→Regbase 后，部分基础 API 高维切分场景性能降低 | 沿用旧 API/分块方式不保证提速 | 重评估寄存器驻留、切分及融合；不是所有向量运算都退化 |
| L1 边界 | 删除边界值设定寄存器 | 不支持 `SetLoadDataBoundary` | 显式处理边界和填充 |
| 图像预处理 | 删除 AIPP 硬件指令，改为软仿 | AIPP 接口可能降速 | 核查 `SetAippFunctions` / `LoadImageToLocal` 路径 |
| 调试 | 移除 UB 异常调试接口 | `CheckLocalMemoryIA` 不可用 | 调整调试方式；不是计算功能削弱 |
| 归约辅助 | 移除所有 repeat 最值/索引的寄存器读取能力 | `GetReduceRepeatMaxMinSpr` 不可用 | 可用 Regbase Reduce 手动实现 |

### 2.3 布局、访问语义与编译配置变化

| 编号/类别 | 变化或条件 | 迁移关注点 |
| --- | --- | --- |
| 12：L0A 布局 | 分形排布由 ZZ 改为 NZ；原 ZZ→ZN 分形变化不再支持 | 重新计算 L0A 切分地址、布局及 K 对齐，不属于可直接复用旧地址公式的增强 |
| SSBuffer | 非 MIX 各核独立占用 1 KB；MIX 共享 3 KB；任务、使能及 32 字节访问约束 | 通信双方显式初始化并约定布局；不能作为任意跨任务缓存 |
| DCI | 与 DCCI 不同，不回写脏行；接口方式调用前需 DSB_ALL | 仅在满足使用条件时替换；编译选项为 `--cce-no-dcache-flush` |
| Host 类型 | 部分新增低精度类型、int4x2_t、bfloat16_t 仅支持以 C++ 模板形式定义/声明 | 不直接沿用普通 Host 变量写法 |
| Ascend C 预留 UB | `--cce-disable-asc-reserved-ubuf` 可释放预留空间 | 依赖该空间的接口将不可用 |
| VF 栈预留 UB | `--cce-disable-vf-stack-reserved-ubuf` 可释放 6 KB | 必须保证寄存器不溢出；不能无条件释放 |
| 多 repeat 同步 | `NO_OVERLAP_IN_MULTI_REPEAT` 可消除不必要同步 | 前提是相应内存区域无地址重叠 |

## 三、PTOAS / PyPTO 当前现状

核查日期：2026-10-21。PyPTO 基线：`4b3f0e9576e39f78d52485065432bd4c8c932808`；PTOAS 基线：`1b36b6f58e6a9cf4db6e76b98206411a8f9d1f80`。

完整能力与生成路径核查（原稿路径：`ptoas-capability-audit.md`，本仓库未收录）提供 703 个 PTOAS 操作定义、逐操作 PyPTO 生成依据与 A5 接入细节。当前 153 项是源码生成候选，不是运行验证覆盖率。950 默认选择 A5，但使用 EmitC，未启用 VPTO、PTOAS Tile fusion 或 buffer-ID 同步。

当前环境的 PTOAS 版本探测因缺少 TileOps 资源失败，本次未完成编译、仿真或真机验证。以下接口结论以该审计报告的证据及边界为准。

第二章新增列出的 s4、稀疏、Subnormal、AIPP 等迁移项，目前只确认原文所述架构变化；尚未逐项重新追踪 PTOAS/PyPTO 的拒绝诊断、仿真实现或替代 lowering，不能按硬件变更直接推定其软件行为。

### 逐项开放情况


| 编号 / 能力 | PTOAS | 当前 PyPTO |
| --- | --- | --- |
| 1：UB→L1 | `tmov`、`mte_ub_l1` | 已有 A5 V2C、`tpush_to_aic` 路径 |
| 2：L1→UB | 底层 `mte_l1_ub` | 未见完整公开路径；950 Mat 搬运图没有 Mat→Vec |
| 3：L0C→UB | Acc→Vec `tmov`、`mte_l0c_ub` | 已有 Acc→Vec、C2V 及后处理路径 |
| 4：ND-DMA | 未见专用 ND-DMA 接口；DMA loops 不能直接等同于 ND-DMA | 未见专用接口和 lowering |
| 5：DMA loop | `mte_load/store(..., loops=...)`；A5 前两层使用硬件 loop | 未显式开放 loop 参数；后端内部使用需单独核查 |
| 6：MX | MX 搬运、`tmatmul.mx`；校验接受 FP8×FP8 和 FP4×FP4 | 部分：MXFP8；原生 FP4×FP4 未开放 |
| 7：2D 随路转置增强 | transpose、起点、step、stride；完整 `srcFracGap` 对应未确认 | 常规转置搬入；未显式开放完整分形间隔参数 |
| 8：Fixpipe NZ2DN | Fixpipe NZ2DN，GM/L1/UB 底层接口 | 有 DN view/store 表达；全部目的空间和参数组合未确认 |
| 9：2D 搬运增强 | `mte_l1_l0a/b`、`mte_gm_l1_frac` 等参数 | load/extract/move 覆盖常见搬运，未暴露全部硬件参数 |
| 10：Stride 卷积搬运 | 未见专用 3D/Stride 卷积搬运接口 | 未见对应专用接口 |
| 11：低精度矩阵类型 | FP8、HiF8、MX FP4 类型与矩阵操作 | 有 FP8/HF8 类型及生成链路，具体算子约束不同 |
| 12：L0A NZ 布局 | A5 verifier 检查 NZ 对应布局 | 950 后端已适配 Left/跨核 NZ 布局 |
| 13：基础向量 API 扩展 | Tile、VPTO、VMI 多层接口，不能整体与 Ascend C API 一一等同 | 部分，见下表 |
| 14：Regbase | 显式 SIMD 寄存器、VPTO/VMI | Tile→EmitC 主链路；未选择 VPTO backend，未见公开 Reg/VF 编程 API |
| 15：原子接口扩展 | SIMT Add/Max/Min/Exch/CAS；Scalar 同名 API 完整对应未确认 | Tile store atomic-add；未见完整返回旧值接口 |
| 16：DCI | 未见专用 DCI 接口及编译开关集成 | `cacheinvalid` 对应 DCCI，不是 DCI |
| 17：SSBuffer | 未见独立 SSBuffer 地址空间及读写接口 | 未见类型、接口及 lowering |
| 18：Mutex | 有 `get_buf/rls_buf`；与 Mutex 完整语义等价未确认 | 未见公开 Mutex/Lock/Unlock API |
| 19：核间 mode 4 | `set_intra_block/wait_intra_block` 低层原语 | 未直接开放；`sync_set` 的 FFTS mode 仅 0/1/2 |
| 20：SIMT | `@pto.simt`、launch、线程/warp/原子接口 | 未见原生 SIMT DSL API |
| 21：SIMD/SIMT 混编 | SIMT helper/region 与 SIMD Kernel 协作 | 未见原生 DSL 混编 API |

### 矢量 API 细分

| 原文能力 | PTOAS | PyPTO |
| --- | --- | --- |
| Adds/Muls/Subs/Divs/Maxs/Mins/Compares 的 LocalTensor 单元素标量形式 | Tile/寄存器标量操作，需逐项映射 | 可 `tile.read` 后传入标量；参数形式不同 |
| Select、Neg、Prelu | 对应能力 | `select`、`neg`、`prelu` |
| Mull 高低双结果 | `vmull` | 未见对应双结果公开接口 |
| AbsSub、ExpSub | 基础或相关复合操作可组合 | 可组合表达式，未见完全对应独立 API |
| MulsCast | 乘法和转换可组合 | `muls` 后 `cast`，不保证单条融合指令 |
| Truncate | 带舍入模式的 `vtrc` | 未见同名独立 API，需核对舍入语义 |
| Interleave/DeInterleave | Tile/寄存器接口 | 未见对应公开接口 |
| 张量移位量 ShiftLeft/ShiftRight | 相关接口 | `shl/shr`；已有状态表注明部分 ISA/语义仍需复核 |
| Ands/Ors | 对应能力 | `ands/ors` |

### MX 的具体边界

`pl.matmul_mx` 的实际输入数据要求为 `FP8E4M3FN`，scale 为 `FP8E8M0`。FP4 lhs 必须先显式转换为 FP8；这不是原生 MXFP4。原生 FP4×FP4、MX E5M2 输入及 MXFP4 动态量化未开放。已有 MX matmul 和 quant 系统测试文件，但本次未重跑。详见 MX 约束（原稿路径：`ir/05-operators.md`，本仓库未收录）。

### 补充能力的软件接入情况

| 增强点 | 场景 | 核查结果 |
| --- | --- | --- |
| 核数增加 | 更大并行规模 | 已有 A5/950 核与 cluster 建模；具体产品核数需匹配环境 |
| UB 总容量 256 KB 与 bank 组织 | 更大分块、减少 bank 冲突 | PyPTO 当前按 240 KB 安全可用容量建模；未见公开 bank 布局控制 |
| 数据类型扩展 | 低精度存储、转换及矩阵计算 | PTOAS 更广；PyPTO 有 FP8/HF8/FP4 E2M1X2，未见独立 FP4 E1M2 |
| NDDMA Cache | ND-DMA 复用及一致性 | 两者均未见专用控制接口 |
| `CACHELINE_ATOMIC` | 原子操作的 cache 一致性 | 未见两者显式开放参数 |
| `--cce-disable-asc-reserved-ubuf` | 释放 Ascend C 预留 UB | 未见专用配置入口 |
| `--cce-disable-vf-stack-reserved-ubuf` | 无溢出时释放 VF 栈预留 UB | 未见专用配置入口 |
| `NO_OVERLAP_IN_MULTI_REPEAT` | 无地址重叠时减少同步 | 未见正式配置入口 |

### 代码依据

- PyPTO 950 资源与搬运图（原稿路径：`../../../src/backend/common/soc.cpp`，本仓库未收录）、950 布局（原稿路径：`../../../src/backend/950/backend_950_handler.cpp`，本仓库未收录）。
- PyPTO Tile API（原稿路径：`../../../python/pypto/language/op/tile_ops.py`，本仓库未收录）、同步 API（原稿路径：`../../../python/pypto/language/op/system_ops.py`，本仓库未收录）。
- [PTOAS 搬运接口](https://github.com/hw-native-sys/PTOAS/blob/1b36b6f58e6a9cf4db6e76b98206411a8f9d1f80/ptodsl/ptodsl/_ops_mte.py)。
- [PTOAS 操作定义及 DCCI 语义](https://github.com/hw-native-sys/PTOAS/blob/1b36b6f58e6a9cf4db6e76b98206411a8f9d1f80/include/PTO/IR/PTOOps.td)。
- [PTOAS MX 类型校验](https://github.com/hw-native-sys/PTOAS/blob/1b36b6f58e6a9cf4db6e76b98206411a8f9d1f80/lib/PTO/IR/PTOTile/PTOInsertAndFillPad.cpp)。
- 跨核通信（原稿路径：`../reference/pto-isa/01-tpush_tpop.md`，本仓库未收录）、已有 PTOAS 状态矩阵（原稿路径：`ptoas-op-status.md`，本仓库未收录）。

## 来源与判断口径

来源为 [asc-devkit 的 3510 新特性说明](https://gitcode.com/cann/asc-devkit/blob/master/docs/zh/guide/cross_gen_migration_guide/instructions_for_new_features/3510_new_features.md)。线上页面访问受限，实际读取本机已有 `asc-devkit-master.zip` 内的同路径文件；归档条目时间为 2026-10-14 14:53:34，内容 SHA256 为 `1a7124b4836d55bf9c091521bb077ab8c45aaa99e4072071bb7770218c964c6a`。未确认线上 master 此后变化。

第二章另参考同一归档中的 [2201到3510架构变更](https://gitcode.com/cann/asc-devkit/blob/master/docs/zh/guide/cross_gen_migration_guide/3510_arch_migration/2201_to_3510_arch_changes.md)，条目时间同为 2026-10-14 14:53:34，内容 SHA256 为 `c5afa7a7f8c64dc0d92b955904fb560027eb7d75be205bf1b9e5861de83d4b94`。该文档补充了原“新增特性”说明未完整列出的能力移除及性能风险。

“开放”指静态源码中存在接口和实现依据，不表示本次完成编译、仿真或上板验证。“部分”表示类型、参数或执行路径覆盖不完整；“未见”表示未找到对应公开路径。底层操作定义不等于所有后端、产品及类型均可执行。PTOAS 本地源码版本与 PyPTO 固定依赖版本必须分别检查。
