# Hexagon 模型与端到端加速证据

调研日期：2026-09-15。仅桌面源码/公开资料验证，未在 Snapdragon 8 Elite + Armada OS 真机执行。

## Hexscale 源码与模型证据

`drewano/hexscale` README宣称W8A8 720p→1080p低于1ms/1W，但没有可复核设备和测试协议，其FP16一律退化scalar DSP的说法与Qualcomm官方v69+浮点支持冲突。[平台审计](platform-notes.md)还确认其执行路径未调用QNN graphExecute，因此这些性能句子不能作为真机证明。模型转换脚本亲核只调用onnx-converter，未执行校准/量化与context生成；SDK缺失时仅打印dry-run命令并return True。[原仓库](https://github.com/drewano/hexscale)、[转换脚本](https://github.com/drewano/hexscale/blob/ce4aaacf60e6370a5a7086e19828b52f36a84bd6/models/convert_qnn.py)。

`54pkp/hexscale` 固定commit `8fed58387dcd5b417910b71f15b12d3aa1e509fa`已提供更具体模型资产。XLSR W8A8来源Qualcomm v0.62.2，输入128×128、输出512×512（4×），context目标SM8750/v79/QAIRT2.45，但清单明确`hardware_tested:false`。ANVIL模型为FP32，INT8尚需校准，仅Windows QNN CPU推理经过验证。[模型清单](https://github.com/54pkp/hexscale/blob/8fed58387dcd5b417910b71f15b12d3aa1e509fa/models/MODEL-MANIFEST.json)。

该fork的`upscale_frame`采用8像素halo，因此128输入tile有效步长112；串行调用`execute_host_memory`，然后CPU双线性采样拼装目标输出。推断：720p输入对应84次tile推理。若目标1.5×，当前4×模型仍生成更大中间结果后缩小；单纯输出像素量比原生1.5×多16/2.25≈7.11倍，但不能把它说成整个网络FLOPs或时延也是7.11倍。重复halo、84次提交和CPU格式转换都应列入端到端成本。[实际处理代码](https://github.com/54pkp/hexscale/blob/8fed58387dcd5b417910b71f15b12d3aa1e509fa/daemon/src/frame_processor.cpp)。

现阶段优先改进模型形状与I/O，而非只切换后端名字：为720p→1080p建立原生1.5×或经过验证的较大tile版本，比较全帧/大tile的VTCM和内存成本。W8A8适合作为轻量SR的跨代候选；v79先用FP16作为质量校验，再比较W8A8。插帧流场/采样等敏感路径可试W8A16或保留FP16/GPU，但必须确认具体HTP算子组合能运行，不能预先保证混合精度既快又保真。v68与v69+应分别提供适用配置，跨代优先保持后端接口和验证流程一致。

该fork的发布验证文件明确`arm64_runtime_tested:false`、`anvil_w8a8_calibrated:false`、`automatic_game_frame_generation:false`。所以Hexscale提供可复用的工程起点，尚未提供8 Elite Armada OS游戏SR/FG性能的实测定论。[验证记录](https://github.com/54pkp/hexscale/blob/8fed58387dcd5b417910b71f15b12d3aa1e509fa/docs/validation/0.2.0-dev1.json)。

## 可用于最终结论的三个基准

| 证据 | 配置 | 数字 | 边界 |
| --- | --- | --- | --- |
| Qualcomm Research QuickSRNet 论文 | Snapdragon 8 Gen 1 Hexagon，2×超分，540p→1080p | 2.2 ms | 历史论文模型推理，不是8 Elite/Linux呈现链测量 |
| Qualcomm AI Hub Models v0.61.0 QuickSRNetSmall | Snapdragon 8 Elite Mobile，QNN_DLC，W8A8，QAIRT 2.45.0.260326154327，128×128输入，3×模型 | 0.193 ms | 极小输入，不能换算成720p/1080p游戏整帧承诺 |
| ANVIL v3 预印本及作者开源播放器 | Snapdragon 8 Gen 3、Android、CPU+GPU Vulkan+HTP INT8、1080p H.264 30→60 | 28.4 ms端到端中位 | 30分钟54,623对帧，94.9%≤33.3ms；依赖codec motion vectors；不是游戏/8 Elite验证 |

来源：[QuickSRNet论文](https://arxiv.org/html/2303.04336v1)、[固定版本QuickSRNetSmall提交](https://huggingface.co/qualcomm/QuickSRNetSmall/commit/c62c1316b6838b07e77fde0cdac2b92a2e0a5d49)、[ANVIL论文](https://arxiv.org/html/2603.26835v3)、[ANVIL作者代码](https://github.com/NihilDigit/anvil)。

QuickSRNet论文给出单帧、简单卷积/激活/DepthToSpace设计，并专门讨论游戏超分、1.5×模式与量化；精度不是DLSS式时域重建。该论文适合证明轻量SR与Hexagon的契合度。其部署步骤会调整DepthToSpace权重排列为DCR布局，提醒移植需处理具体张量布局。

小图数值必须固定版本：同一HF main在搜索缓存/实时页面中会出现0.191、0.193等差异，本笔记使用v0.61.0固定commit新增行的0.193 ms。该版本ONNX W8A8为0.235 ms，优先仅使用QNN_DLC数字。

ANVIL自身孤立NPU数字12.8 ms在论文方法中是BURST、10次热身后50–100次取minimum；HF模型卡写avg，因此最终建议只用可清晰解释的端到端28.4ms中位。ANVIL以软件H.264解码器导出的MV作为运动先验，GPU做预对齐/warp，NPU做卷积残差精修；游戏swapchain不存在该codec MV，不能直接接入替代LSFG。

## SDK算子与跨代可用性

当前[拆分版ONNX Runtime QNN EP文档](https://github.com/onnxruntime/onnxruntime-qnn/blob/main/docs/execution_providers/QNN-ExecutionProvider.md)的算子表包含Conv、ConvTranspose、DepthToSpace、Resize、GridSample。算子名受支持不等于其所有dtype、属性、shape组合都在HTP执行，需实际编译并检查delegation/trace。此文档仍明确QNN EP的输入shape需固定；因此建议固定常见渲染分辨率，按模型、精度、分辨率缓存已编译图。这里的限制应写“QNN EP”，不能外推整个QNN没有任何动态能力。

同一新文档说明：QAIRT≥2.35浮点图在支持的SoC上用FP16 math，enable_htp_fp16_precision开关已不改变HTP浮点执行精度；不要照抄旧站点“HTP只能量化”段落。FCB从QAIRT≥2.48开始可把多个SoC context装进同一DLC/EPContext，当前需要x86_64离线准备；这不是任意单个context自动兼容所有硬件。

Qualcomm官方[AI Hub Apps README](https://github.com/qualcomm/ai-hub-apps)明确FP16适用于Hexagon v69及更新，INT8/INT16适用于其支持的Snapdragon。仍须核查具体模型的算子精度支持与BSP/固件，而非按TOPS划线。

| SoC示例 | HTP架构 | 项目建议 |
| --- | --- | --- |
| SM8350 / Snapdragon 888 | v68 | 独立W8A8兼容档，不能默认FP16 |
| SM8450 / SM8475 | v69 | FP16与量化按实际性能选择 |
| SM8550 | v73 | 单独编译/验证配置 |
| SM8650 | v75 | 单独编译/验证配置 |
| SM8750 / Snapdragon 8 Elite | v79 | 首个目标；先建立本机可运行与延迟基线 |

映射证据：[PyTorch ExecuTorch官方SoC schema](https://github.com/pytorch/executorch/blob/main/backends/qualcomm/serialization/qc_schema.py)。

## 为什么插帧风险大于单帧超分

ANVIL研究的RIFE HTP V75 profile显示，特定360p FP16 RIFE执行中卷积仅占5.1%，Resize/GridSample/逐元素/布局相关操作占主要时间；这说明从GPU换到张量NPU未必更快。其W8A8实验也显示迭代flow的量化累积会损害质量。该研究是单一作者预印本、指定SDK/模型/设备的实验，不能当作所有RIFE、LSFG或未来v79的绝对上界。可把它用作试验设计依据：真实图profile、warp属性测试、量化前后游戏序列评价、禁止隐性CPU fallback。

[RIFE原作者仓库](https://github.com/hzwer/ECCV2022-RIFE)的“720p 30+FPS”基准明确使用2080Ti，不能当作移动NPU证据。未检索到Qualcomm AI Hub官方发布的可直接部署RIFE游戏插帧模型。建议保留现有Vulkan呈现/warp路径，先测试适合HTP的重卷积子图，或为游戏数据重新设计/训练适合NPU的插帧网络。

Real-ESRGAN-x4plus虽有[Qualcomm官方可部署模型](https://huggingface.co/qualcomm/Real-ESRGAN-x4plus)，但模型复杂度远高于QuickSRNet，不能将“可在NPU运行”当作“适合每帧游戏低延迟”。最终报告不必堆其小图数值。

## 共享内存和同步

Qualcomm官方[QCNode QNN封装文档](https://github.com/qualcomm/QcNode/blob/main/docs/QNN.md)支持注册输入输出buffer、zero-copy buffer sharing、同步/异步执行及完成回调，并能区分accelerator与RPC时间。其示例建议先分配buffer池后重复执行。该文档也有动态batch示例，故不要把EP静态shape限制泛化成所有QNN API限制。

[旧ORT QNN EP文档](https://onnxruntime.ai/docs/execution-providers/QNN-ExecutionProvider.html)有enable_htp_shared_memory_allocator并要求libcdsprpc.so/dll；新拆分EP文档已有较大变更，不能无验证把旧option名称写进新EP2.x部署命令。

推断：application↔HTP零拷贝，不自动证明Vulkan swapchain image↔HTP零拷贝。还必须验证GPU image tiling/modifier、通道布局、stride、外部内存导出导入、cache维护和GPU/NPU fence同步。即使两端共享DDR，格式变换与同步也可能抵消推理收益。VK_QCOM_data_graph_model的新通道详见[Vulkan迁移专项](../VULKAN-NPU-MIGRATION.zh-CN.md)，不应声称Vulkan永远无法访问NPU。

## 原型验收建议（工程判断）

1. 在8 Elite Armada OS上跑固定shape QuickSRNet，确认实际HTP执行、精度正确、固件与runtime匹配。
2. 分离统计纯推理、GPU↔NPU交换、格式/布局转换、present等待，热身后测P50/P95/P99及30分钟持续负载。
3. 使用720p/1080p真实游戏帧、UI文字、透明粒子、场景切换测质量，FP16正确性基线之后再测W8A8/W8A16。
4. SR和FG分别A/B：同输出分辨率/目标帧率比较GPU时间、游戏基础帧率、功耗和温度。新增SR会增加处理成本；只有降低游戏原始渲染分辨率或替代已有昂贵GPU推理才有充分理由期待腾出GPU。
5. 框架保留Vulkan回退，按具体SoC/驱动/HTP能力选择NPU图与精度；首版不要承诺所有Hexagon硬件同画质同帧率。
