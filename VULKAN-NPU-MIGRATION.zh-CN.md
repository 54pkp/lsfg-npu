Vulkan Data Graph 扩展与 LSFG-VK 负载迁移：专项核查

日期：2026-09-15。下面是规范、源码证据及工程设计，未在 Armada 真机执行。

**当前驱动结论：已核查的 Mesa 26.2.2 Turnip 不提供所需的三项 QCOM/ARM graph/tensor 扩展，Armada 自有补丁没有添加它们。** 已下载 [Armada 配方锁定](https://github.com/armada-os/armada-packages/blob/24d88394fd6ae777e7616c3ec245a7a35b5d2f81/mesa/BASE.env)的[官方源码包](https://archive.mesa3d.org/mesa-26.2.2.tar.xz)，SHA-256 为 `eeb29ca7e56cfaa8e8a79538dcf834e3b18e501c31bef5145e959ea437cc4216`，与配方匹配。`tu_device.cc` 的完整扩展表与整个 Turnip 目录未实现这些能力；三项[Armada 补丁](https://github.com/armada-os/armada-packages/tree/24d88394fd6ae777e7616c3ec245a7a35b5d2f81/mesa/patches)分别涉及同步、A830 ID 和 UBO lowering。未完整审计构建脚本引用的 Fedora SRPM 打包层全部补丁；用户实装镜像、自定义 ICD 或未来版本，应按下文运行时 gate 确认。

**可以把 Vulkan 作为 GPU/NPU 的统一调度接口；前提是驱动暴露真实的 Hexagon graph engine，并且待迁移算法已经具备可执行的图表达。** 扩展解决调度和资源互通接口，不能从现有 opaque shader 自动恢复模型语义。

| 扩展 | 实际能力 | 对本项目的意义 |
|---|---|---|
| `VK_QCOM_data_graph_model` | 引入 Qualcomm neural model / built-in model 操作和 Hexagon engine | 可导入准备好的 QNN 模型，在 Vulkan 管线中调度 NPU |
| `VK_ARM_data_graph` | 数据图 pipeline、session、queue 和 dispatch | 建立新的后端执行路径；不是现有 compute pipeline 的一个开关 |
| `VK_ARM_tensors` | tensor 资源、内存绑定、复制、图像 aliasing | 连接 GPU 图像预处理和 NPU 模型 I/O，减少不必要复制 |
| `VK_ARM_data_graph_instruction_set_tosa` | 标准 TOSA 图算子，并查询支持的 profile/level/实现质量 | 重建为标准张量图的备选；不能假定 Hexagon 驱动提供对应执行引擎 |
| `VK_ARM_data_graph_optical_flow` | 估计两幅图像的二维像素位移 | 可作为替代算法的运动估计候选；不保证 Hexagon 执行，也不保证输出等价于 LSFG 内部中间量 |

对应一手规范：[QCOM model](https://docs.vulkan.org/refpages/latest/refpages/source/VK_QCOM_data_graph_model.html)、[ARM data graph](https://docs.vulkan.org/refpages/latest/refpages/source/VK_ARM_data_graph.html)、[ARM tensors](https://docs.vulkan.org/refpages/latest/refpages/source/VK_ARM_tensors.html)、[TOSA](https://docs.vulkan.org/refpages/latest/refpages/source/VK_ARM_data_graph_instruction_set_tosa.html)、[optical flow](https://docs.vulkan.org/refpages/latest/refpages/source/VK_ARM_data_graph_optical_flow.html)。

**不能把两种 SPIR-V 使用方式混淆。** `SPV_ARM_graph` 用 `OpGraphARM`、图输入/输出及张量算子表达整资源的数据流。现有计算着色器则是逐线程的 GPU 程序，带采样、存储、局部工作组及同步语义。二者都可能用 SPIR-V 容器，但不是同一个执行模型；给现有 LSFG shader 换一个 pipeline 创建函数不会完成转换。[SPV_ARM_graph 规范](https://github.khronos.org/SPIRV-Registry/extensions/ARM/SPV_ARM_graph.html)

Qualcomm 的 foreign model 路径更具体：应用在 Vulkan 外准备模型二进制，添加 QCOM pipeline cache header，再创建 data graph pipeline。也就是说，可以维持 Vulkan 前端，但仍有模型转换/编译这个步骤。`Generic QNN` 是规范示例的 operation 名称，实际需查询实现暴露的名称和版本，不能硬编码后就假定成功。[QCOM 工作流程](https://docs.vulkan.org/features/latest/features/proposals/VK_QCOM_data_graph_model.html)

**建议设计两条 NPU 执行路径。**

```text
共同部分：LSFG/应用帧调度 → 图像/张量适配 → 后端 → GPU合成/呈现

路径 A（驱动支持时）
Vulkan graph pipeline → QNN 模型 → Hexagon

路径 B（没有该 Vulkan 扩展、但系统支持 QNN 时）
原生 QAIRT/QNN API → FastRPC → Hexagon

两条都不可用或错过帧期限：现有 Vulkan 后端 / 原帧呈现
```

这是统一接口内的能力选择，而不是要求叠加多个抢占 `vkQueuePresentKHR` 的 layer。路径 A 更有利于沿用 Vulkan 的资源和同步表达；路径 B 不要求等待 Turnip 实现 vendor graph 扩展。两者都需要处理模型 I/O、设备兼容和错误恢复。

**在目标设备上必须验证的门槛如下。** 它们是实际运行时检查，查看 Vulkan 版本或装上新头文件不能代替。

1. 对使用的物理设备和驱动记录 vendor/device ID、driver ID/版本，枚举扩展；确认 QCOM model、ARM graph/tensors 及其依赖，并查询相应 feature 位。需区分 ICD 原生扩展和额外 layer 暴露的扩展。
2. 枚举带 `VK_QUEUE_DATA_GRAPH_BIT_ARM` 的 queue family；调用 `vkGetPhysicalDeviceQueueFamilyDataGraphPropertiesARM`，查到 `VK_PHYSICAL_DEVICE_DATA_GRAPH_PROCESSING_ENGINE_TYPE_NEURAL_QCOM` 和所需模型 operation。只有 graph queue、只有 TOSA 或只有 GPU compute，均不足以证明 Hexagon 可用。
3. 查询该 foreign engine 支持的外部内存和信号量 handle 类型，验证模型 I/O 的格式、维度、stride 和图像 aliasing。不能预设已有的 `OPAQUE_FD` 就是所需 DMA-BUF，也不能保证 image/tensor 格式无转换成本。
4. 用一个已知输出的 V79 模型创建 pipeline 和持久 session，绑定实际 I/O，提交 `vkCmdDispatchDataGraphARM`；确认正确输出、NPU 执行证据及完整 GPU→NPU→GPU 同步。
5. 再运行游戏图像序列，统计整个帧路径与持续负载。若模型/图像布局不兼容，允许复制，但必须把复制计入性能。

查询和执行接口见：[graph engine 查询](https://docs.vulkan.org/refpages/latest/refpages/source/VK_ARM_data_graph.html)、[QCOM foreign engine 和同步要求](https://docs.vulkan.org/features/latest/features/proposals/VK_QCOM_data_graph_model.html)。初始化过程应分配可重复使用的图、session 和缓冲；避免每帧创建/销毁资源，这是工程建议。

特别注意：Arm 的开源 ML Emulation Layer 能在 Vulkan Compute 设备上模拟 graph/tensor/光流接口，适合开发和正确性测试。它的工作原理是把相关运算放到 Vulkan Compute，并不能凭空获得 Hexagon NPU。看到这些 ARM 扩展或运行了官方 graph 示例，不应直接报告 NPU 加速。[Arm 原作者实现说明](https://github.com/arm/ai-ml-emulation-layer-for-vulkan)

Qualcomm 自己的 `graph_pipelines` 示例也有两项移植注意点：代码优先 NEURAL_QCOM，但允许 COMPUTE_QCOM 回退；内存创建使用 ANDROID_HARDWARE_BUFFER。因此示例运行成功还需确认实际引擎，Armada 原生 Linux 需要重新协商受支持的外部内存句柄。[引擎选择源码](https://github.com/SnapdragonGameStudios/adreno-gpu-vulkan-code-sample-framework/blob/8177ad3c0498cc92a2588ef6c170148cbcf7ae16/samples/graph_pipelines/code/main/ml/QcomDataGraphModel.cpp#L165)、[Android 内存源码](https://github.com/SnapdragonGameStudios/adreno-gpu-vulkan-code-sample-framework/blob/8177ad3c0498cc92a2588ef6c170148cbcf7ae16/samples/graph_pipelines/code/main/ml/TensorResources.cpp#L57)

**对 LSFG 的迁移，应从可验证的子图边界入手。** 公开 v3.1 管线存在 mipmap、alpha/beta/gamma/delta/epsilon 和 generate 阶段；这些名称和资源连接不足以证明每个 shader 的数学定义。没有 shader 内部语义/模型参数，就不能精确将 alpha 等阶段替换成某个公开光流网络，也不能保证替换运动估计后现有 generate 能正确消费输出。[管线声明](https://git.lsfg-vk.dev/lsfg-vk/tree/lsfg-vk-pipeline/src/pipelines/v3_1.cpp?id=0e7a3898c1285b13df8596f2bd2cbb8f85b4383b#n23)

优先验证“完整、可定义 I/O 的重计算子图”或独立公开 SR/VFI 模型。前者需拿到确切算子和参数，后者会改变算法。不要把几十个 GPU dispatch 逐个拆成几十次 GPU/NPU 往返；应尽可能把连续可支持的阶段融合到图内。轻量复制、必要格式转换、warp 和最后合成先保留 GPU，再以 profiling 决定进一步迁移。

现有源码还提供了更明确的拆分：`beta4` 之后调用 `s.split()`，形成每对真实帧执行一次的 pre-pass，以及每个生成帧执行一次的 main-pass。下面数量来自管线声明计数，是逻辑 dispatch 数，不是测得的耗时或占比。

| 原有阶段 | 执行频率 | 逻辑 dispatch 数 | 适配建议 |
|---|---|---:|---|
| mipmaps + alpha + beta | 每对真实帧一次 | 1 + 28 + 5 = 34 | 对齐 `prepare(A,B)`，缓存可供后续生成使用的中间数据 |
| gamma + delta + epsilon + generate | 每个中间帧一次 | 35 + 15 + 15 + 1 = 66 | 对齐 `synthesize(A,B,prepared,t)`，依据模型逐步融合或保留 GPU 部分 |
| 截帧、资源交接、present、节奏控制 | 每个基础帧/输出帧 | 不作为图算子计数 | 保留宿主/Vulkan 调度，处理 NPU 完成事件和失败回退 |

来源：[split 边界](https://git.lsfg-vk.dev/lsfg-vk/tree/lsfg-vk-pipeline/src/pipelines/v3_1.cpp?id=0e7a3898c1285b13df8596f2bd2cbb8f85b4383b#n233)、[逐帧参数和执行](https://git.lsfg-vk.dev/lsfg-vk/tree/lsfg-vk-pipeline/src/pipeline.cpp?id=0e7a3898c1285b13df8596f2bd2cbb8f85b4383b#n869)。这只是可复用的执行结构，不证明 34/66 个 shader 均能在 NPU 高效执行。

原有多倍率通过不同 `t` 生成中间帧：M 倍需要 `t=k/M`。替代模型必须支持这些时间点；一个仅支持 midpoint 的 ANVIL 模型不能直接替代 3×所需的 1/3、2/3。递归 midpoint 也会增加调用和误差，并非免费扩展。[ANVIL midpoint 工具](https://github.com/54pkp/hexscale/blob/8fed58387dcd5b417910b71f15b12d3aa1e509fa/cli/src/interpolate.cpp)

同步同样需要重构：data-graph-only 队列不能照搬普通 compute command buffer 的 barrier/transfer；有依赖的 foreign graph 执行需要按规范安排 submit 和外部信号量。不能机械地把 `vkCmdDispatch` 改名为 `vkCmdDispatchDataGraphARM` 后沿用所有命令。[QCOM 同步规则](https://docs.vulkan.org/features/latest/features/proposals/VK_QCOM_data_graph_model.html#_synchronization)

如只替换部分 LSFG 阶段，应逐项固定中间张量的通道意义、坐标单位、采样规则、边界处理、精度和跨帧历史。只有输出格式尺寸相同并不能证明语义兼容。跨设备兼容同样应由模型和 engine 的能力检查保证，扩展名称本身不构成“所有 Hexagon 通用”的承诺。
