# LSFG / Hexagon NPU 可行性研究

面向 Snapdragon 8 Elite（SM8750 / Hexagon V79）与 Armada Linux，研究将超分辨率及插帧计算迁移到 Qualcomm NPU 的可行性，并评估跨 Hexagon 平台支持。

资料核查日期：**2026-09-15**。本仓库存放资料与设计分析，不是已经完成的 NPU 插帧实现。核查包括公开文档、固定提交源码和二进制包；尚未进行目标掌机实测。

## 阅读顺序

| 文档 | 内容 |
|---|---|
| [完整可行性报告](HEXAGON-FEASIBILITY.zh-CN.md) | 结论、SDK、Linux 支持、Hexscale 参考价值与验证路线 |
| [Vulkan 扩展迁移专项](VULKAN-NPU-MIGRATION.zh-CN.md) | QCOM/ARM Data Graph、Turnip 支持、张量互通及阶段拆分 |
| [平台证据](research/platform-notes.md) | Armada 内核、固件、FastRPC、Mesa 源码和官方示例 |
| [模型与性能证据](research/model-notes.md) | SR/VFI 模型、基准边界、量化和跨代能力 |
| [LSFG 源码审计](research/repo-audit/SOURCE-AUDIT.zh-CN.md) | 原生实现、模型资产、API 边界和许可 |
| [LSFG Data Graph 适配](research/repo-audit/LSFG-DATA-GRAPH-ADAPTER.zh-CN.md) | 逐阶段资源连接、34/66 次 dispatch、两图方案与时序 |
| [54pkp/hexscale 源码审计](research/repo-audit/HEXSCALE-54PKP-AUDIT.zh-CN.md) | 已实现的 QNN 路径、可复用组件和真机验证缺口 |
| [来源与校验记录](research/SOURCES.md) | 固定提交、官方下载位置、SHA-256 与复核说明 |

## 使用这些结论

- 源码中存在接口、成功交叉编译或模型小图基准，均不等同于 Armada 真机加速证据。
- 优先验证原生 QNN/HTP；Vulkan Data Graph 路径取决于实际驱动暴露的引擎、模型和互通能力。
- GPU/NPU 数据交换、同步、基础游戏 FPS、画质及持续功耗都应纳入性能评价。
- 后续更新应记录设备、镜像、SDK、固件、模型和测试方法，保留原结论的适用版本。

## 资料管理

Git 只跟踪报告、来源链接和校验记录。下载的 SDK wheel、Mesa 发布包、上游源码副本及临时查询输出被 `.gitignore` 排除，保留在研究工作目录中；克隆本仓库不会包含这些缓存。需要复核时按 [来源记录](research/SOURCES.md)重新获取对应版本。

各上游源码、模型、SDK 和固件仍适用其自身许可，本仓库的资料索引不授予这些第三方内容的再分发权。
