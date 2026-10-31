# VPTO C++ Bridge 使用示例与 EmitC 后端限制

[返回 A5 文档目录](README.md)

本文面向需要在 VPTO kernel 中使用矩阵乘、跨核 Pipe 和设备打印的开发者，基于当前仓库的实际用例给出配置、编译、host 调用和结果检查方法。示例目标为 A5。

> 本文命令按仓库根目录执行；本次归档修复文档路径与策略示例格式，未重新执行编译、模拟器或 NPU 验证。

## 1. 使用范围与后端选择

VPTO C++ Bridge 允许一部分 PTO 操作在 VPTO 编译过程中调用编译器生成的设备侧 C++ wrapper，复用 PTO-ISA/CANN 提供的实现。用户编写 PTO IR 和路由策略；wrapper 的生成、编译和链接由 `ptoas` 完成。

普通 VPTO 操作与 bridge 操作可以出现在同一 kernel 中。当前可选择的操作如下：

| Family | 操作 | 不指定策略文件时 | 典型用途 |
|---|---|---|---|
| `pipe` | `pto.initialize_l2l_pipe`、`pto.tpush`、`pto.tpop`、`pto.tfree` | 启用 | Cube 与 Vector 之间传递 tile |
| `cube` | `pto.tmatmul`、`pto.tmatmul.acc` | 关闭 | 通过 C++ wrapper 执行矩阵乘 |
| `print` | `pto.print`、`pto.tprint` | 启用 | 标量诊断与 tile 内容打印 |

编译时必须选中 VPTO 后端：

```bash
ptoas --pto-arch=a5 --pto-backend=vpto input.pto -o kernel.fatobj.o
```

也可以在所属 `module` 上设置 `pto.backend = "vpto"`。显式 CLI 选项优先于模块属性；两处都不指定时，默认后端是 `emitc`。

Bridge 与 EmitC 是不同的编译能力：bridge 在 VPTO 编译中为选中的操作提供 C++ 调用，EmitC 后端则输出整份 kernel 的 CCE C++。策略文件不能插入任意 C++ 源码、指定自定义 include 或声明任意外部 ABI。

## 2. 环境准备

| 依赖 | 作用 |
|---|---|
| 当前 checkout 对应的 PTOAS | 解析 PTO IR，生成 VPTO 或对象文件 |
| CANN/BiSheng 与配套 `llvm-link` | 编译设备 C++ wrapper，并完成设备代码链接 |
| 兼容的 PTO-ISA 头文件 | 提供 Cube/Pipe 的 C++ 实现 |
| CANN DebugTunnel 头文件和运行库 | Print 的编译与日志解码 |
| A5 模拟器或实际 NPU runtime | 执行 kernel；只导出 IR 时不需要设备 |
| Python 与 NumPy | 仓库 host 验证用例的数据生成和比较 |

PTO-ISA 路径也可以通过 `PTO_ISA_PATH` 指定；两者同时存在时，`PTO_ISA_PATH` 优先。应使用与当前 PTOAS/CANN 环境兼容的 PTO-ISA 版本。

## 3. 路由策略的完整写法

将以下内容保存为 `bridge-policy.yaml`，以启用 Pipe、两种 Cube 操作和两种 Print 操作：

```yaml
version: 1
families:
  pipe:
    enabled: true
  cube:
    enabled_ops:
      - pto.tmatmul
      - pto.tmatmul.acc
  print:
    enabled_ops:
      - pto.print
      - pto.tprint
```

通过下列选项使用它：

```bash
--vpto-bridge-policy=bridge-policy.yaml
```

需要遵守以下规则：

- `version` 当前只能是 `1`，`families` 必须存在。
- `pipe` 使用 `enabled: true|false`，整体控制其四个生命周期操作。
- `cube`、`print` 使用 `enabled_ops` 逐项选择操作，不能用 `enabled: true` 开启整个 family。
- 操作名必须属于对应 family；未知字段、未知操作和重复操作会报错。
- **自定义文件替换整份内置策略。** 省略的 family 为关闭或空列表，不会继承内置默认值。例如只写 `cube` 时，Pipe 和 Print 均不启用。
- 未被选中的 Cube 操作继续使用常规 VPTO 实现；被选中但不满足 bridge 约束的操作会直接报错，不会自动回退。
- VPTO 中的 `pto.print`、`pto.tprint` 没有非 bridge 实现。输入包含它们时，策略必须保留相应操作。

内置默认策略等价于：

```yaml
version: 1
families:
  pipe: { enabled: true }
  cube: { enabled_ops: [] }
  print: { enabled_ops: [pto.print, pto.tprint] }
```

## 4. 示例一：Cube 矩阵乘，经 Pipe 交给 Vector 写回

### 4.1 仓库用例与计算结果

完整运行用例是 [`cube-tmatmul-bridge`](../../../test/vpto/cases/kernels/cube-tmatmul-bridge)。它执行以下工作：

1. 将两个 `16×16` 的 f16 矩阵从 GM 搬入本地矩阵存储。
2. 通过 bridged `pto.tmatmul` 计算 f32 结果。
3. Cube 将 `16×16` Acc tile 推入 Pipe，按上下两半分给 Vector。
4. Vector 从弹出的 tile 地址写回前 `8×16` 个 f32，随后释放槽位。

该用例的预期值为：

```text
A = eye(16, dtype=f16)
B = arange(256, dtype=f16).reshape(16, 16)
C = matmul(float32(A), float32(B))
output = C[0:8, 0:16]               # 128 个 f32，共 512 字节
output[0] = [0, 1, ..., 15]
output[7] = [112, 113, ..., 127]
```

输出仅从 `tpop` 返回的地址写回，因此结果比较同时覆盖矩阵乘和 Pipe 数据消费。

该目录通过 `harness.dir` 引用兄弟目录 [`fifo-tile-data-consume`](../../../test/vpto/cases/kernels/fifo-tile-data-consume) 的 `launch.cpp`、`main.cpp`、`golden.py`、`compare.py`。复制用例时应保留这两个目录的相对关系，或一并复制被引用的 harness。

### 4.2 完整 PTO IR

以下代码对应仓库的 `cube-tmatmul-bridge/kernel.pto`，可以保存为 `$BRIDGE_WORK_DIR/matmul-pipe.pto`。本例显式编排搬运和同步，保留默认 `level2`，由编译器为 `alloc_tile` 规划本地地址。

```mlir
module attributes {pto.target_arch = "a5"} {
  func.func @fifo_tile_data_consume_kernel(%a: !pto.ptr<f16, gm>,
                                          %b: !pto.ptr<f16, gm>,
                                          %out: !pto.ptr<f32, gm>)
      attributes {pto.entry} {
    pto.section.cube {
      %c0 = arith.constant 0 : i64
      %c1 = arith.constant 1 : i64
      %c16 = arith.constant 16 : i64
      %c512 = arith.constant 512 : i64
      %fifo_addr = arith.constant 65536 : i32
      %pipe = pto.initialize_l2l_pipe{dir_mask = 1, slot_size = 1024, slot_num = 8, flag_base = 0, nosplit = false} (%fifo_addr : i32) -> !pto.pipe

      %l1a = pto.castptr %c0 : i64 -> !pto.ptr<f16, l1>
      %l1b = pto.castptr %c512 : i64 -> !pto.ptr<f16, l1>
      %left = pto.alloc_tile
        : !pto.tile_buf<loc=left, dtype=f16, rows=16, cols=16, v_row=16, v_col=16, blayout=col_major, slayout=row_major, fractal=512, pad=0>
      %right = pto.alloc_tile
        : !pto.tile_buf<loc=right, dtype=f16, rows=16, cols=16, v_row=16, v_col=16, blayout=row_major, slayout=col_major, fractal=512, pad=0>
      %l0a = pto.tile_buf_addr %left
        : !pto.tile_buf<loc=left, dtype=f16, rows=16, cols=16, v_row=16, v_col=16, blayout=col_major, slayout=row_major, fractal=512, pad=0>
       -> !pto.ptr<f16, l0a>
      %l0b = pto.tile_buf_addr %right
        : !pto.tile_buf<loc=right, dtype=f16, rows=16, cols=16, v_row=16, v_col=16, blayout=row_major, slayout=col_major, fractal=512, pad=0>
       -> !pto.ptr<f16, l0b>
      %acc = pto.alloc_tile
        : !pto.tile_buf<loc=acc, dtype=f32, rows=16, cols=16, v_row=16, v_col=16, blayout=col_major, slayout=row_major, fractal=1024, pad=0>

      pto.mte_gm_l1 %a, %l1a, %c512 nburst(%c1, %c0, %c0) loop(%c1, %c0, %c0)
        : !pto.ptr<f16, gm>, !pto.ptr<f16, l1>, i64, i64, i64, i64, loop i64, i64, i64
      pto.mte_gm_l1 %b, %l1b, %c512 nburst(%c1, %c0, %c0) loop(%c1, %c0, %c0)
        : !pto.ptr<f16, gm>, !pto.ptr<f16, l1>, i64, i64, i64, i64, loop i64, i64, i64
      pto.set_flag["PIPE_MTE2", "PIPE_MTE1", "EVENT_ID0"]
      pto.wait_flag["PIPE_MTE2", "PIPE_MTE1", "EVENT_ID0"]
      pto.mte_l1_l0a %l1a, %l0a, %c16, %c16, %c0, %c0
        : !pto.ptr<f16, l1>, !pto.ptr<f16, l0a>, i64, i64, i64, i64
      pto.mte_l1_l0b %l1b, %l0b, %c16, %c16, %c0, %c0 {transpose = true}
        : !pto.ptr<f16, l1>, !pto.ptr<f16, l0b>, i64, i64, i64, i64
      pto.set_flag["PIPE_MTE1", "PIPE_M", "EVENT_ID0"]
      pto.wait_flag["PIPE_MTE1", "PIPE_M", "EVENT_ID0"]

      pto.tmatmul ins(%left, %right : !pto.tile_buf<loc=left, dtype=f16, rows=16, cols=16, v_row=16, v_col=16, blayout=col_major, slayout=row_major, fractal=512, pad=0>, !pto.tile_buf<loc=right, dtype=f16, rows=16, cols=16, v_row=16, v_col=16, blayout=row_major, slayout=col_major, fractal=512, pad=0>)
                  outs(%acc : !pto.tile_buf<loc=acc, dtype=f32, rows=16, cols=16, v_row=16, v_col=16, blayout=col_major, slayout=row_major, fractal=1024, pad=0>)
      pto.set_flag["PIPE_M", "PIPE_FIX", "EVENT_ID1"]
      pto.wait_flag["PIPE_M", "PIPE_FIX", "EVENT_ID1"]
      pto.tpush(%acc, %pipe : !pto.tile_buf<loc=acc, dtype=f32, rows=16, cols=16, v_row=16, v_col=16, blayout=col_major, slayout=row_major, fractal=1024, pad=0>, !pto.pipe) {split = 1}
    }

    pto.section.vector {
      %c0 = arith.constant 0 : i64
      %c1 = arith.constant 1 : i64
      %c512 = arith.constant 512 : i64
      %fifo_addr = arith.constant 65536 : i32
      %pipe = pto.initialize_l2l_pipe{dir_mask = 1, slot_size = 1024, slot_num = 8, flag_base = 0, nosplit = false} (%fifo_addr : i32) -> !pto.pipe
      %tile = pto.declare_tile -> !pto.tile_buf<loc=vec, dtype=f32, rows=8, cols=16, v_row=8, v_col=16, blayout=row_major, slayout=none_box, fractal=512, pad=0>
      pto.tpop(%tile, %pipe : !pto.tile_buf<loc=vec, dtype=f32, rows=8, cols=16, v_row=8, v_col=16, blayout=row_major, slayout=none_box, fractal=512, pad=0>, !pto.pipe) {split = 1}
      %ub = pto.tile_buf_addr %tile
        : !pto.tile_buf<loc=vec, dtype=f32, rows=8, cols=16, v_row=8, v_col=16, blayout=row_major, slayout=none_box, fractal=512, pad=0>
       -> !pto.ptr<f32, ub>
      %subblock = pto.get_subblock_idx
      %is0 = arith.cmpi eq, %subblock, %c0 : i64
      scf.if %is0 {
        pto.mte_ub_gm %ub, %out, %c512 nburst(%c1, %c0, %c0)
          : !pto.ptr<f32, ub>, !pto.ptr<f32, gm>, i64, i64, i64, i64
      }
      pto.tfree(%pipe : !pto.pipe) {split = 1}
      pto.barrier #pto.pipe<PIPE_ALL>
    }
    return
  }
}
```

### 4.3 参数和生命周期说明

| 参数/数值 | 单位与含义 |
|---|---|
| `dir_mask = 1` | Cube → Vector；`2` 表示 Vector → Cube |
| `slot_size = 1024` | 每槽字节数；本例生产者 tile 为 `16×16×4 = 1024` 字节 |
| `slot_num = 8` | FIFO 槽数 |
| `flag_base = 0` | 通信同步资源的起始编号；不同 Pipe 的资源分配应避免冲突 |
| `nosplit = false` | 本例使用拆分模式；省略该属性也按 `false` 处理 |
| `%fifo_addr = 65536 : i32` | FIFO 的本地字节地址/偏移；本例放在普通 tile 使用区之外，生产消费两侧一致 |
| `split = 1` | 沿行方向分成上下两半；本例 Vector 每份 tile 为 `8×16` |
| `%c512` | GM/L1 输入各 512 字节，Vector 回写也是 512 字节 |

`split` 的语法值为 `0`（不拆分）、`1`（上下）、`2`（左右）、`3`（上下奇数拆分）、`4`（左右奇数拆分）；具体方向、形状和架构仍受操作校验及 PTO-ISA 约束。同一 Pipe 对象的 push/pop/free 必须使用一致的 split。

生命周期顺序为 `initialize → push → pop → 使用弹出数据 → free`。消费端的 `declare_tile` 不单独分配数据空间，`tpop` 将其绑定到 FIFO 槽位。释放之前应完成对槽位数据的访问，并满足相关流水线的同步要求。

### 4.4 导出 IR 与生成对象

以下命令直接使用仓库完整输入，无需先手动保存上一节代码：

```bash
export BRIDGE_CASE_DIR="$PTO_SOURCE_DIR/test/vpto/cases/kernels/cube-tmatmul-bridge"

"$PTOAS_BIN" --pto-arch=a5 --pto-backend=vpto \
  --vpto-bridge-policy="$BRIDGE_CASE_DIR/bridge-policy.yaml" \
  --emit-vpto "$BRIDGE_CASE_DIR/kernel.pto" \
  -o "$BRIDGE_WORK_DIR/matmul-pipe.vpto.mlir"

"$PTOAS_BIN" --pto-arch=a5 --pto-backend=vpto \
  --vpto-bridge-policy="$BRIDGE_CASE_DIR/bridge-policy.yaml" \
  --emit-vpto-llvm-ir "$BRIDGE_CASE_DIR/kernel.pto" \
  -o "$BRIDGE_WORK_DIR/matmul-pipe.ll"

"$PTOAS_BIN" --pto-arch=a5 --pto-backend=vpto \
  --vpto-bridge-policy="$BRIDGE_CASE_DIR/bridge-policy.yaml" \
  "$BRIDGE_CASE_DIR/kernel.pto" \
  -o "$BRIDGE_WORK_DIR/kernel.fatobj.o"
```

三种输出的区别：

| 命令形式 | 输出 | 验证范围 |
|---|---|---|
| `--emit-vpto` | 编译后的 VPTO 文本，包含 bridge 信息和生成的 wrapper 源码属性 | 检查路由、类型、地址与 wrapper 生成 |
| `--emit-vpto-llvm-ir` | 设备 LLVM IR 文本 | 检查调用声明和调用参数；文本中允许保留待链接的 wrapper 声明 |
| 不加上述导出选项 | host 可链接的 fatobj | 实际编译并链接 wrapper；需要完整 CANN/PTO-ISA 工具链 |

导出 `.ll` 成功不能替代 wrapper 编译验证。`--vpto-print-ir` 用于向 stderr 打印调试信息；要保存 VPTO 文件应使用 `--emit-vpto`。

### 4.5 编译、运行和比较

使用仓库驱动脚本可完成 fatobj、launch object、kernel 动态库、host 可执行文件、golden 和 compare 的整条流程：

```bash
WORK_SPACE="$BRIDGE_WORK_DIR/runtime" \
CASE_NAME=kernels/cube-tmatmul-bridge \
DEVICE=SIM \
PTOAS_BIN="$PTOAS_BIN" \
PTOAS_FLAGS='--pto-arch=a5 --pto-backend=vpto' \
  bash "$PTO_SOURCE_DIR/test/vpto/scripts/run_host_vpto_validation.sh"
```

脚本自动读取该 case 的 `bridge-policy.yaml`。模拟器库默认从 CANN 中查找 `dav_3510/lib`，必要时通过 `SIM_LIB_DIR` 指定。

输出目录为 `$BRIDGE_WORK_DIR/runtime/kernels_cube-tmatmul-bridge/`，主要文件包括：

```text
kernel.fatobj.o
launch.o
libkernels_cube-tmatmul-bridge_kernel.so
kernels_cube-tmatmul-bridge
v1.bin / v2.bin / v3.bin / golden_v3.bin
validation.log
```

成功标志是 `compare passed` 和 `All 1 VPTO case(s) passed`。本例比较容差为 `atol=1e-2, rtol=1e-2`。

只编译到 kernel 动态库、不执行设备时，在上述命令中增加 `COMPILE_ONLY=1`；运行真实 NPU 时使用 `DEVICE=NPU` 并准备可用的设备 runtime。脚本会重新创建当前 case 的输出子目录，因此应使用专用 `WORK_SPACE`。

### 4.6 C++ host 如何调用

host 调用的是 kernel launch 接口。下面是仓库 `launch.cpp` 中的核心调用形式，需放入具有 CANN 头文件和类型环境的完整源文件中编译：

```cpp
extern "C" __global__ [aicore] void fifo_tile_data_consume_kernel(
    __gm__ __fp16 *a, __gm__ __fp16 *b, __gm__ float *c);

void LaunchFifo_tile_data_consume_kernel(
    __fp16 *a, __fp16 *b, float *c, void *stream) {
  fifo_tile_data_consume_kernel<<<1, nullptr, stream>>>(
      (__gm__ __fp16 *)a, (__gm__ __fp16 *)b, (__gm__ float *)c);
}
```

host 需要为输入分别准备 512 字节设备内存，为输出准备 512 字节；完成 H2D 拷贝后调用该接口，同步 stream，再 D2H 拷回比较。完整资源申请、错误处理和清理代码使用上述共享 harness 的 `main.cpp`。

## 5. 示例二：带已有累加值的矩阵乘

`pto.tmatmul.acc` 的逻辑结果为 `dst = acc_in + lhs × rhs`。它与 `pto.tmatmul` 是独立的路由项，策略需要包含 `pto.tmatmul.acc`。

完整编译用例见 [`cube_tmatmul_acc.pto`](../../../test/lit/vpto/bridge/cube_tmatmul_acc.pto)。其关键调用如下，`%acc`、`%dst` 均为已分配的 Acc tile：

```mlir
pto.tmatmul.acc
  ins(%acc, %lhs, %rhs : !pto.tile_buf<loc=acc, dtype=f32, rows=16, cols=16, v_row=16, v_col=16, blayout=col_major, slayout=row_major, fractal=1024, pad=0>, !pto.tile_buf<loc=left, dtype=f16, rows=16, cols=16, v_row=16, v_col=16, blayout=col_major, slayout=row_major, fractal=512, pad=0>, !pto.tile_buf<loc=right, dtype=f16, rows=16, cols=16, v_row=16, v_col=16, blayout=row_major, slayout=col_major, fractal=512, pad=0>)
  outs(%dst : !pto.tile_buf<loc=acc, dtype=f32, rows=16, cols=16, v_row=16, v_col=16, blayout=col_major, slayout=row_major, fractal=1024, pad=0>)
  {accPhase = #pto<acc_phase final>}
```

```bash
"$PTOAS_BIN" --pto-arch=a5 --pto-backend=vpto \
  --vpto-bridge-policy="$BRIDGE_WORK_DIR/bridge-policy.yaml" \
  --emit-vpto "$PTO_SOURCE_DIR/test/lit/vpto/bridge/cube_tmatmul_acc.pto" \
  -o "$BRIDGE_WORK_DIR/matmul-acc.vpto.mlir"
```

该 lit 用例验证编译与 wrapper 参数，未初始化矩阵数据，不能直接作为数值运行样例。实际运行前应装载有效的 lhs、rhs 和 acc_in。`accPhase` 可省略（默认 `unspecified`），也可显式设置 `unspecified`、`partial`、`final`，应按实际矩阵乘分段流程设置；示例一使用省略形式，独立的 phase 编译用例见 [`cube_tmatmul_e2e.pto`](../../../test/lit/vpto/bridge/cube_tmatmul_e2e.pto)。

## 6. 示例三：标量打印和 tile 打印

### 6.1 最小标量打印 kernel

保存为 `$BRIDGE_WORK_DIR/print.pto`：

```mlir
module attributes {
  pto.target_arch = "a5",
  pto.kernel_kind = #pto.kernel_kind<vector>
} {
  func.func @print_kernel(%dt: !pto.ptr<i8, gm>) attributes {pto.kernel} {
    %offset = arith.constant 1.5 : f32
    %n = arith.constant 64 : i32
    pto.print ins("offset = %+08.3f\n", %offset : f32)
    pto.print ins("n = %d\n", %n : i32)
    return
  }
}
```

```bash
"$PTOAS_BIN" --pto-arch=a5 --pto-backend=vpto \
  --vpto-print-dtdata=explicit --emit-vpto \
  "$BRIDGE_WORK_DIR/print.pto" -o "$BRIDGE_WORK_DIR/print.vpto.mlir"

"$PTOAS_BIN" --pto-arch=a5 --pto-backend=vpto \
  --vpto-print-dtdata=explicit \
  "$BRIDGE_WORK_DIR/print.pto" -o "$BRIDGE_WORK_DIR/print.fatobj.o"
```

不传策略文件时 Print 默认启用。即使 kernel 只有打印、没有 tile 计算，也可以编译。此最小示例的运行仍需 host 打开 DebugTunnel 并传入 `%dt`；完整 host 流程见 6.3。

### 6.2 格式串、类型和 `tprint` 限制

每条 `pto.print` 接受一个标量，格式串必须恰好包含一个支持的转换项；`%%` 是普通百分号，不计入转换项数。可以使用 `+ - 0 #` 和空格标志、数字宽度及数字精度，不支持 `*` 动态宽度/精度。

| 标量类型 | 格式示例 | 行为 |
|---|---|---|
| f16、bf16、f32 | `%f`、`%e`、`%g`、`%a` 及对应大写形式 | f16/bf16 扩展后输出 |
| f64 | `%f` 等浮点转换 | 最终按 f32 精度记录，不保留 f64 精度 |
| 窄整数、i32 | `%d`、`%i` | 按有符号值扩展 |
| 窄整数、i32 | `%u`、`%x`、`%X`、`%o` | 按无符号值扩展 |
| i64 | `%lld`、`%llu`、`%llx` 等 | 必须带 64 位长度修饰；实现接受 `ll`、`l`、`z`，建议显式使用 `ll` |

例如，`pto.print ins("n = %d", %value : i64)` 会报缺少 64 位长度修饰；`%f %f`、纯字面量、`%s`、`%p` 均不能作为单标量 Print bridge 的格式串。

`pto.tprint` 示例：

```mlir
%address = arith.constant 1024 : i64
%tile = pto.alloc_tile addr = %address
  : !pto.tile_buf<loc=vec, dtype=f32, rows=8, cols=8, v_row=2, v_col=3, blayout=row_major, slayout=none_box, fractal=512, pad=0>
pto.tprint ins(%tile : !pto.tile_buf<loc=vec, dtype=f32, rows=8, cols=8, v_row=2, v_col=3, blayout=row_major, slayout=none_box, fractal=512, pad=0>)
```

这是需要嵌入 Vector kernel 的片段；执行前必须向该 UB 地址写入数据并完成相应同步。输出遍历有效区域 `[0:2, 0:3]`，相邻两行的物理间隔仍为 8 个元素。

当前 VPTO `tprint` 要求：

- 二维 Vec tile，ND row-major，即 `blayout=row_major, slayout=none_box`。
- 源直接来自已有地址的 `alloc_tile`，有效行列数均为静态值。
- 元素类型为 f16、bf16、f32 或 signless i8/i16/i32；这些整数按有符号值输出，不能据此推断 unsigned tile 打印已支持。
- 不支持 `tmp` 操作数和显式 `printFormat`。浮点元素当前固定使用 `%6.2f`，整数使用 `%6d`。

### 6.3 DebugTunnel 与完整运行用例

`--vpto-print-dtdata` 显式选择参数约定，编译器不会根据函数签名自动猜测：

| 值 | 默认 | host 约定 |
|---|---|---|
| `implicit` | 是 | 由 CCE launch/runtime 提供隐藏的 DebugTunnel 参数；面向真实 NPU 的约定 |
| `explicit` | 否 | host 自行打开 DebugTunnel，并把指针作为 kernel 的第一个用户参数；仓库模拟器用例使用此方式 |

两种模式都会保留设备编译需要的隐藏尾参数，用户手写的 launch 声明只表达用户参数，隐藏参数由 CCE 编译流程处理。`explicit` 模式的第一个用户参数必须是 GM 指针，建议声明为 `!pto.ptr<i8, gm>`。

完整用例为 [`print-bridge`](../../../test/vpto/cases/kernels/print-bridge)，它同时执行：64 个 f32 的 GM→UB→GM 复制、两条标量打印、两个静态有效区域的 `tprint`。其 `ptoas.flags` 已包含：

```text
--pto-arch a5 --pto-backend=vpto --pto-level=level3 --vpto-print-dtdata=explicit
```

该用例所有相关地址均显式给出，因此使用 `level3`。`ptoas.flags` 存在时会替换驱动脚本的全局 `PTOAS_FLAGS`，随后再追加 case 的 `bridge-policy.yaml`（若存在）。

```bash
WORK_SPACE="$BRIDGE_WORK_DIR/runtime" \
CASE_NAME=kernels/print-bridge \
DEVICE=SIM \
PTOAS_BIN="$PTOAS_BIN" \
  bash "$PTO_SOURCE_DIR/test/vpto/scripts/run_host_vpto_validation.sh"
```

host 的关键顺序是：

```text
dtData = __DebugTunnel_Open(1)
检查 dtData 非空
LaunchPrint_bridge_kernel_mix_aiv(dtData, input, output, 1.5f, 64, stream)
aclrtSynchronizeStream(stream)
__DebugTunnel_Close(dtData, stream)
拷回 output 并比较
```

单独编译含打印 kernel 的 `launch.cpp` 时，还需要 `--cce-enable-print -D__CCE_ENABLE_PRINT_FOUND_CANN__`；仓库驱动脚本会自动添加。完整 host 代码位于该用例目录中。

`compare.py` 同时检查复制结果和 `validation.log` 中解码出的打印文本，包括：

```text
offset = +001.500
n = 64
=== [TPRINT Tile] Data Type: float32, Layout: ND, TileType: Vec ===
  Shape: [8, 8], Valid Shape: [2, 3]
  Shape: [4, 8], Valid Shape: [2, 3]
```

它还检查两个 tile 有效区域中的实际数值，不能仅凭 host 复制正确就认定 Print 成功。输出目录为 `$BRIDGE_WORK_DIR/runtime/kernels_print-bridge/`。

## 7. Bridge 当前支持边界

| 范围 | 约束 |
|---|---|
| Cube core | `tmatmul`、`tmatmul.acc` 必须处于 Cube section 或具有 Cube kernel kind 的上下文 |
| Cube 操作数 | 采用 `ins/outs` 形式且无 tensor SSA 返回值；dst/lhs/rhs/acc_in 都须直接来自带 i64 地址的 `alloc_tile` |
| Cube 形状 | 二维静态正形状及静态有效形状，满足 `M×K` 与 `K×N`；输出为 Acc、左输入为 Left、右输入为 Right |
| Cube 类型 | bridge tile 描述接受 f16/bf16/f32 和 8/16/32 位整数；具体乘加类型组合、布局仍须满足操作及 PTO-ISA 约束 |
| Cube 变体 | 仅上述两个已注册操作；未提供 MX scale、FP4/FP8 或其他 matmul 变体的直接 bridge 路由 |
| Cube view | 不接受仍以 view/subview 表示的输入；Pipe 弹出地址不能直接当作当前 bridged matmul 的 Left/Right 输入 |
| Pipe 方向 | 当前 bridge 仅支持 local Pipe，`dir_mask=1`（Acc→Vec）或 `2`（Vec→Mat），不支持 `acc_push_epilogue` |
| Pipe 地址 | 初始化时本地地址类型为 i32；push 的 tile 必须有可解析地址；槽位大小、数量应与 tile 和实际存储匹配 |
| Pipe pop/free | 每个声明的 tile 最多一个 `tpop`，且先 pop 再使用；`tfree` 使用仅 pipe 参数的形式 |
| Pipe 本地槽数 | 当前 bridge 使用 2 个本地槽，不应依赖自定义 `local_slot_num` 改变这一行为 |
| Print | 受第 6 节的格式、类型、布局和 DTData 参数约定限制 |
| 工具链 | 文本 IR 验证不保证 wrapper 能通过实际 CANN/PTO-ISA 编译，版本必须匹配 |
| 运行覆盖 | 本文使用 A5 模拟器验证；真实 NPU，尤其隐式 DTData 路径，需在目标环境单独验证 |

## 8. EmitC 后端的限制与使用区别

### 8.1 输出形式与 bridge 选项

| 项目 | VPTO + C++ Bridge | 单一 EmitC 后端 |
|---|---|---|
| 选择方式 | `--pto-backend=vpto` | `--pto-backend=emitc` |
| 正常输出 | fatobj，需要显式输出文件 | CCE C++ 源码文本，通常保存为 `.cpp` |
| `--vpto-bridge-policy` | 决定已注册操作是否使用 bridge | 不启用 VPTO bridge，也不会补齐 EmitC 缺少的操作 |
| `--emit-vpto` / `--emit-vpto-llvm-ir` | 导出对应文本 | 与 EmitC 组合会报 VPTO 专属选项错误 |
| `--vpto-print-dtdata` | 控制 VPTO Print 的 DTData 投递方式 | 不能代替 EmitC/CANN 自身的 Print 配置 |
| host 集成 | 链接 fatobj 与 launch object | 还需另行用 CANN 编译生成的 C++ 并链接 |

例如，下面的 EmitC 命令输出 C++ 文本；即使把输出名改成 `.o`，也不会因此变为对象文件：

```bash
"$PTOAS_BIN" --pto-arch=a5 --pto-backend=emitc \
  "$PTO_SOURCE_DIR/test/lit/vpto/bridge/cube_tmatmul_e2e.pto" \
  -o "$BRIDGE_WORK_DIR/matmul.emitc.cpp"
```

这里使用纯 tile 计算的编译用例。第 4 节完整用例还包含 VPTO 微指令，不能直接套用此命令，原因见下一节。

### 8.2 不能直接使用完整 VPTO 微指令用例

本文 `cube-tmatmul-bridge/kernel.pto` 包含 `pto.mte_gm_l1`、`pto.mte_l1_l0a`、`pto.mte_l1_l0b` 等设备操作。当前 EmitC 路径没有覆盖本例所需的全部转换；直接切换后端会在首个不支持的操作处失败。实际命令与诊断为：

```bash
"$PTOAS_BIN" --pto-arch=a5 --pto-backend=emitc \
  "$PTO_SOURCE_DIR/test/vpto/cases/kernels/cube-tmatmul-bridge/kernel.pto" \
  -o "$BRIDGE_WORK_DIR/unsupported.emitc.cpp"
```

```text
error: failed to legalize operation 'pto.mte_gm_l1' that was explicitly marked illegal
```

为 EmitC 编写程序时，应采用它已支持的 tile 级加载、计算和存储操作，并核对目标架构约束。路由 YAML 只能选择 VPTO bridge，不能把上述微指令自动变为 EmitC 可用代码。

### 8.3 EmitC 仍支持的能力与其他约束

- EmitC 自身已有 `tmatmul`、Pipe、`print`、`tprint` 等操作的 C++ 输出支持；不能将“不能启用 VPTO bridge”理解为这些语义在 EmitC 中全部不可用。
- EmitC 支持 Vector/Cube section、普通条件和循环，不能笼统表述为“不支持混合核或控制流”。
- EmitC 的普通函数调用/返回转换不支持直接携带多个结果；应改成单结果或输出参数形式。编译器也会拒绝递归或循环调用图。
- `pto.tdump` 当前仅支持 VPTO；EmitC 会给出要求使用 VPTO 的诊断。
- EmitC `tprint` 有独立的 `tmp`、`printFormat` 处理，不能将第 6 节 VPTO bridge 的限制原样推广到 EmitC；最终可编译类型与布局以对应 PTO-ISA/CANN 支持为准。
- EmitC C++ 文本生成成功不代表设备 C++ 已编译、已链接或运行正确，还需要单独完成工具链验证。

### 8.4 含 EmitC 与 VPTO 子模块的混合后端输入

仓库另有以子 `module` 为边界的混合后端 fatobj 流程；这是独立的组合机制。VPTO 子模块可以使用 bridge，EmitC 子模块按自己的能力编译。

混合后端输入需要显式 `-o` 文件路径，且不支持混合 fatobj 模式下直接导出 `--emit-vpto` 等调试 IR。调试时应分别导出待检查的单后端编译单元。不要在希望保留混合后端时传全局 `--pto-backend=vpto`，因为显式 CLI 会覆盖模块选择。

进一步说明见 [Mix-Kernel and Mix-Backend Compile Flow](../../designs/mix-kernel-mix-backend-compile-flow.md)。

## 9. 常见报错与处理

| 诊断/现象 | 原因与处理 |
|---|---|
| `VPTO bridge policy: cannot read` | 策略路径错误；检查文件存在且可读 |
| `pipe uses enabled; cube and print use enabled_ops` | family 配置方式错误；按第 3 节重写 |
| `invalid or duplicate ... op` | 操作不在该 family 或名称重复 |
| `the route policy excludes ... which has no non-bridge VPTO lowering` | 自定义策略漏掉了输入中的 Print 操作 |
| `matmul bridge ... requires an alloc_tile with a planned i64 address` | tile 来源或地址不满足要求；恢复内存规划或为支持的 `alloc_tile` 显式提供地址，检查是否误用了无完整地址的 `level3` |
| `requires core 'cube'` | Cube 操作缺少 Cube 上下文或放在 Vector 中 |
| `tprint bridge requires static valid dimensions` | 有效行列为动态值；调试时使用静态有效区域 |
| `explicit print DTData delivery needs a global-memory pointer as the kernel's first argument` | 显式打印模式的签名缺少首个 GM 指针参数 |
| 打印无输出 | 检查 DTData 模式、首参数传值、launch 编译打印开关及 host 的 tunnel open/close 与 stream 同步 |
| `bridge wrapper bitcode compilation` 失败 | 查看编译诊断，检查 CANN/PTO-ISA 版本、头文件路径、目标类型/布局 |
| `bridge wrapper bitcode linking` 失败 | 检查使用的配套 `llvm-link`、工具链版本及设备符号 |
| `ASCEND_HOME_PATH is required for VPTO fatobj emission` | 对象输出需要完整 CANN 路径；仅观察 IR 时使用文本导出命令 |
| `failed to legalize operation 'pto.mte_gm_l1'` | 本文 VPTO 微指令用例误用了 EmitC；按第 8 节选择对应输入和后端 |
