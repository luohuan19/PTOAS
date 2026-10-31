# A5 能力与编程指南

本目录集中整理 PTOAS 的 A5 能力、编程接口和使用限制。

| 文档 | 内容与阅读顺序 |
| --- | --- |
| [PTOAS A5 能力图解手册](ptoas-a5-capabilities-guide.html) | 主入口：从 Tile、VMI、micro、SIMT 到搬运、同步与编译产物；包含 UB alias、动态偏移及 Unknown 依赖的分析限制。 |
| [3510 新特性参考稿](3510-new-features.md) | 导入的硬件与软件状态整理，保留原核查日期及版本；部分历史软件结论已在 HTML 指南中补充核对。 |
| [A5 VF 基础能力介绍](a5-vf-basics.md) | VF 执行模型、寄存器与 UB、硬件约束，以及 VMI/MI 对照。 |
| [VPTO 当前能力梳理](vpto-current-capabilities.md) | 架构覆盖、混合后端、Bridge、优化及已知缺口的源码核查。 |
| [VPTO C++ Bridge 使用示例](vpto-cpp-bridge-usage.md) | Pipe/Cube/Print 的策略、编译、调用与验证方式，以及 EmitC 的使用边界。 |

HTML 指南可在本地浏览器中直接打开；在线代码托管页面可能只展示源码，下载或克隆仓库后打开即可使用交互功能。保留仓库目录结构可以使用本地源码链接，固定提交链接需要网络。

指南区分原有验证记录与后续静态核查。本次文档整理未重新执行编译、模拟器或 NPU 验证；接口存在、测试文件存在和实际运行通过分别判断。

建议先读 HTML 指南建立整体模型，再按需要查阅 VF 基础、VPTO 能力和 Bridge 示例。3510 稿作为来源参考阅读，不作为后续源码版本的完整支持承诺。
