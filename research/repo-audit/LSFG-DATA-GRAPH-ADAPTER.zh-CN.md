# lsfg-vk v3.1 管线的 data graph 适配边界

此笔记仅回答阶段划分、适配点和多倍率时序；不再重复平台可用性判断。源码基线为 `0e7a3898c1285b13df8596f2bd2cbb8f85b4383b`。

## 公开源码允许确认什么

阶段来源：[v3_1.cpp](https://git.lsfg-vk.dev/lsfg-vk/tree/lsfg-vk-pipeline/src/pipelines/v3_1.cpp?id=0e7a3898c1285b13df8596f2bd2cbb8f85b4383b)。以下次数是按 `appendPass` 和循环推导的逻辑 shader dispatch 数，不是耗时占比。`PipelineSignatureBuilder` 会在依赖允许范围内重排、按 shader 聚合；实际 command recording 仍对每个 subiteration 执行一次 compute dispatch。

| 组 | 已验证的数据连接及次数 | 从公开源码不能确定的内容 | data graph 迁移建议 |
|---|---|---|---|
| 输入历史/拷贝 | 两层 RGBA8 输入；历史交替、恢复时重置；不属于下列模型 pass | 外部模型需要何种颜色/张量布局 | 保留 Vulkan 捕获与调度；另建 image/tensor 互通 |
| `mipmaps` | pre-pass，1 次；读两帧输入图像数组，写带 Mipmaps 标记的 7 个 R8 级别；第 23–49 行 | 具体滤波核、通道处理、归一化，不可只凭名称判定 | 初版可保留 GPU；若模型图能精确表达再融合进去 |
| `alpha0..3` | pre-pass，7 组，每组 4 次，共 28 次；每组读对应 mipmap，链式写 RGBA8 中间图，末端 alpha 结果 pinned；第 51–138 行 | 是否卷积、网络层结构/权重、打包通道语义、算子精度；不能把 Greek 名称直接等同 CNN | 候选计算簇，须获得/重建精确图后再判定；优先连同相邻计算融合 |
| `beta0..4` | pre-pass，5 次；读 alpha[0]，经 4 个 RGBA8 中间图，输出带 Mipmaps 标记的 6 个 R8 级别；第 140–229 行 | beta 内容是否 motion/feature/mask、具体数学不可知 | 可并入准备阶段 graph；不能因为 R8 就声明已量化为 W8A8 |
| `gamma0..4` | main-pass，7 组×5=35 次；使用对应 alpha、前一 gamma，gamma4 额外读 beta 的一个级别，输出 RGBA16F；第 235–337 行 | 输出是不是光流，所有内部算子、权重与尺度含义 | 候选主计算 graph；跨尺度依赖应留在同图内部，减少同步 |
| `delta0..4` | main-pass，只在 i=4..6，3×5=15 次；读 alpha、前一 delta，delta4 还读 beta；末端 RGBA16F；第 339–436 行 | 不可把 delta 名称解释为某确定数学残差或流方向 | 适合与 gamma/epsilon 联合迁移；必须保留真实依赖 |
| `epsilon0..4` | main-pass，3×5=15 次；epsilon0 读 alpha、gamma[i-1] 和前一 delta；epsilon4 还读前一 epsilon；末端 RGBA16F；第 438–535 行 | 同上；不能宣称某个置信度/遮挡头 | 与主计算 graph 一起评估，避免每个小 pass 独立跨引擎 |
| `generate` | main-pass，1 次；读原始两帧、gamma[6]、delta[2]、epsilon[2]，写最终同尺寸输出；第 541–563 行 | 精确 warping、混合、遮挡/UI/HDR 运算不可知；“最终合成”仅描述位置，不证明所有算法 | 可先保留 GPU，也可在具有精确模型表达时合入主图；输出语义必须一致 |
| 资源规划、descriptor、barrier、队列、pacing | C++ 负责资源寿命/复用、依赖分组、绑定、命令录制、同步与呈现 | 不属于待转换的神经网络算子 | 继续由 host/Vulkan 实现，重写与新引擎相关的资源/同步部分 |

源码的图像格式是 RGBA8_UNORM、R8_UNORM、RGBA16F：[signature/image.hpp 第 14–22 行](https://git.lsfg-vk.dev/lsfg-vk/tree/lsfg-vk-pipeline/src/signature/image.hpp?id=0e7a3898c1285b13df8596f2bd2cbb8f85b4383b#n14)。这些图像存储类型不等于 QNN 张量的量化方案，更不能从 layer count 直接推断神经网络 channel count；仍需理解 shader 的编码/打包方式。

## 最有价值的既有切分

[v3_1.cpp 第 231–233 行](https://git.lsfg-vk.dev/lsfg-vk/tree/lsfg-vk-pipeline/src/pipelines/v3_1.cpp?id=0e7a3898c1285b13df8596f2bd2cbb8f85b4383b#n231) 在 beta4 后执行 `s.split()`。

- 每对真实输入帧执行一次 pre-pass：`1 + 7×4 + 5 = 34` 次逻辑 dispatch。
- 每张生成帧执行一次 main-pass：`7×5 + 3×5 + 3×5 + 1 = 66` 次。
- 因此倍率 M 的此部分逻辑 dispatch 数是 `34 + 66×(M−1)`；2×/3×/4×分别为 100/166/232。此计数不包含 GPU copy、barrier，也不说明任何阶段耗时占比。

对应执行位置为 [Context::dispatch 第 171–225 行](https://git.lsfg-vk.dev/lsfg-vk/tree/lsfg-vk-pipeline/src/lsfgvk.cpp?id=0e7a3898c1285b13df8596f2bd2cbb8f85b4383b#n171) 和 [Context::acquire 第 228–314 行](https://git.lsfg-vk.dev/lsfg-vk/tree/lsfg-vk-pipeline/src/lsfgvk.cpp?id=0e7a3898c1285b13df8596f2bd2cbb8f85b4383b#n228)。

若能取得精确图，优先考虑如下接口形态：

```text
prepare(frame_previous, frame_current) → resident_features
synthesize(resident_features, frame_previous, frame_current, t) → generated_frame
```

这与既有 pre/main 执行节奏一致，可在多倍率下复用一次准备结果。它是建议的新图边界，不是声称已有 QNN 模型。若选用替代开源 VFI 模型，应按该模型的真实输入/输出设计，不能把它伪装成 alpha/beta 的逐 pass 等价替代。

另一可行切分为 GPU mipmap/预处理 → 一个融合 NPU 主图 → GPU 最终合成。选择切分需结合准确模型与 profiler，不能仅根据阶段名称。逐个把 100 个左右的小 shader dispatch 变成跨引擎 graph dispatch 可能制造大量同步和布局转换，通常不适合作为优化目标。

## `vkCmdDispatchDataGraphARM` 的具体接入方式

规范路径确实支持 Vulkan 调度 QNN graph：[VK_QCOM_data_graph_model](https://docs.vulkan.org/refpages/latest/refpages/source/VK_QCOM_data_graph_model.html)。但它不是当前 shader pipeline 的另一个执行按钮。

1. **新增 graph 资源类型与初始化。** 在 `lsfgvk::Instance` / `Context` 下提供独立 `DataGraphBackend`。加载由 QNN 工具链生成的 Vulkan pipeline cache/identifier/图元数据，创建 data graph pipeline、descriptor layout、tensor view 与 session。`ShaderLibrary` 中的 DLL shader 不能直接拿来当 graph cache。
2. **新增命令录制分支。** 当前 [pipeline.cpp 第 755–776 行](https://git.lsfg-vk.dev/lsfg-vk/tree/lsfg-vk-pipeline/src/pipeline.cpp?id=0e7a3898c1285b13df8596f2bd2cbb8f85b4383b#n755) 绑定 compute pipeline + image descriptors + push constants + dispatch。data graph 路径需要先绑定 `VK_PIPELINE_BIND_POINT_DATA_GRAPH_ARM`、兼容 tensor descriptor，再调用 `vkCmdDispatchDataGraphARM(commandBuffer, session, ...)`，而不是只改最后一行。session 还需完成其报告的内存绑定要求：[命令有效性条件](https://docs.vulkan.org/refpages/latest/refpages/source/vkCmdDispatchDataGraphARM.html)。
3. **队列与同步单独规划。** 必须选择支持 data graph 且实际暴露目标引擎/操作的 queue family；为其创建指定 processing engine 的 command pool。如果只支持 data graph，则不能复用当前录制普通 barrier/transfer/updateBuffer 的 command buffer。图间依赖需拆分 submit，并用受支持的外部 semaphore 协调；dispatch 没有隐式消除数据 hazard。现有 compute-only queue 不能默认当作 NPU queue：[QCOM 提案的 command buffer 与 synchronization 章节](https://docs.vulkan.org/features/latest/features/proposals/VK_QCOM_data_graph_model.html#_command_buffers)。
4. **以 tensor 元数据建立交换契约。** 现有 image allocation、OPAQUE_FD 不能自动作为 graph tensor。按报告的 foreign memory handle types、tensor strides/tiling、图输入类型及 image alias 条件分配。若可 alias 则复用内存；若不可，则用 GPU image/tensor conversion/copy。具体依据：[QCOM tensors 与 alias 示例](https://docs.vulkan.org/features/latest/features/proposals/VK_QCOM_data_graph_model.html#_tensors)。
5. **图形学细节保持显式。** iteration 双帧历史、timestamp、performance variant、flow_scale 改变分辨率、HDR/UI处理目前是不同层次的状态，不能假设新 QNN 图会自动继承。可以重建/选择 shape 专用 graph，将 t 作为模型输入；模型是否允许运行时 t 则须检查其实际定义。

建议实现中把“GPU prep → graph submit → GPU post”视为一个后端工作单元，让 `Context::dispatch/acquire` 管理它的依赖与完成通知；现有 API边界和呈现调度仍可保留。

## 多倍率帧时序可以保留，但有明确条件

当前每张生成帧的时间参数由 [pipeline.cpp 第 868–876 行](https://git.lsfg-vk.dev/lsfg-vk/tree/lsfg-vk-pipeline/src/pipeline.cpp?id=0e7a3898c1285b13df8596f2bd2cbb8f85b4383b#n868) 给出：`t = (index+1)/(total+1)`，其中 `total=M−1`。即 2× 为 1/2，3× 为 1/3、2/3，4× 为 1/4、1/2、3/4。

data graph 只改变生成计算的执行方式，不要求改变这些输出时刻。仍可按每张生成帧完成后再安排 present，并遵守原实现“输出消费前不能覆盖同一个 destination”的握手。

必须同时满足：

- 所选 VFI 模型支持这些 t，或者拥有经过质量/成本验证的实现方法。只有 midpoint 的 ANVIL 原型不能直接宣称保持任意 3×/4×行为；递归 midpoint 也无法自然得到 1/3、2/3。
- 每个结果只有在 graph 完成且必要 GPU post/copy 完成后才发出可呈现通知。先 signal 再完成转换会使调度层读到错误内容。
- 外部 semaphore 类型与 timeline 能力要实机查询；不匹配时增加兼容的同步桥接，保持既有逻辑顺序。
- graph tensor、resident_features、session scratch 在所有消费者完成前保持有效；异步重叠要有独立 slot/session 或正确串行化，不能沿用仅为单 GPU 临时图计算的复用寿命。
- 倍率/分辨率/performance/flow_scale 切换、原帧旁路后恢复、swapchain 销毁都要 drain 或 cancel 在途工作并重置历史，延续 ARM64 补丁的语义。

最终可以保持多倍率呈现契约；能否达到时限、获得 GPU 卸载收益，仍取决于所选图、真实驱动引擎能力与含转换/同步的实测。
