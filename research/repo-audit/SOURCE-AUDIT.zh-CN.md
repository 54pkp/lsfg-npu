# decky-lsfg-vk-arm64 / lsfg-vk 源码审计

审计日期：2026-09-15。此文件只记录源码与一手文档证据；未运行 ARM64/NPU 实机测试。

## 审计基线

- 用户仓库固定提交：`2da96d3365576fa23e6844c0f3161073dfceb9d8`（2026-09-14）。通过 GitHub API 确认，并下载该提交的源码 ZIP。
- 插件指定的原生上游基线：`0e7a3898c1285b13df8596f2bd2cbb8f85b4383b`。下载了上游该提交完整 tar.xz，并检查插件附带的 `native/patches/0001-arm64-recovery.patch`。
- 基线依据：[插件 native/README.md 第 3 行](https://github.com/mydanyi/decky-lsfg-vk-arm64/blob/2da96d3365576fa23e6844c0f3161073dfceb9d8/native/README.md#L3)。上游已移到 `git.lsfg-vk.dev`，不能拿旧 GitHub v1 的实现代替本项目 v2。

## 已验证的事实

### 1. 当前项目是插帧，不含现成超分后端

README 的功能是 GPU 插帧，Decky 插件主要提供配置、安装、启动脚本及界面；发布包附带修改过的 ARM64 Vulkan layer。参见 [README 第 9–17、33–45 行](https://github.com/mydanyi/decky-lsfg-vk-arm64/blob/2da96d3365576fa23e6844c0f3161073dfceb9d8/README.md#L9)。

`flow_scale` 指运动估计分辨率比例，不是输出画面的超分倍率。官方配置文档明确区分 frame multiplier、flow scale 与 performance mode：[Configuration Options](https://lsfg-vk.dev/docs/configuration/options/)。源码 UI 也标注为降低内部运动估计分辨率：[UI.qml 第 207–208 行](https://git.lsfg-vk.dev/lsfg-vk/tree/lsfg-vk-ui/resources/UI.qml?id=0e7a3898c1285b13df8596f2bd2cbb8f85b4383b#n207)。

`ContextWrapper` 以相同 `width/height` 创建输入与输出图像；输入为两层 RGBA8 图像，输出为单层 RGBA8 图像：[wrapper.cpp 第 170–231 行](https://git.lsfg-vk.dev/lsfg-vk/tree/lsfg-vk-layer/src/wrapper.cpp?id=0e7a3898c1285b13df8596f2bd2cbb8f85b4383b#n170)。公开目录中未发现独立 SR 实现、ONNX/QNN 接口或相应模型资产。因此若新增 NPU 超分，它属于新增功能，不能描述为迁移项目里已存在的超分模块。

### 2. 实际计算由独立原生 Vulkan pipeline 完成

调用链可概括为：

`Decky UI / Python 配置 → Vulkan layer 的 present hook → ContextWrapper → lsfgvk::Context → Vulkan compute pipeline → 生成图像 → swapchain / present`。

- 插件生成自己的 Vulkan layer manifest、`liblsfg-vk-v2-arm64.so` 路径和环境变量：[runtime_v2.py 第 7–8、95–103 行](https://github.com/mydanyi/decky-lsfg-vk-arm64/blob/2da96d3365576fa23e6844c0f3161073dfceb9d8/py_modules/lsfg_vk/runtime_v2.py#L7)。
- layer 静态链接 `lsfg-vk-pipeline`：[layer/CMakeLists.txt 第 1–11 行](https://git.lsfg-vk.dev/lsfg-vk/tree/lsfg-vk-layer/CMakeLists.txt?id=0e7a3898c1285b13df8596f2bd2cbb8f85b4383b#n1)。pipeline 自身是静态库：[pipeline/CMakeLists.txt 第 1–12 行](https://git.lsfg-vk.dev/lsfg-vk/tree/lsfg-vk-pipeline/CMakeLists.txt?id=0e7a3898c1285b13df8596f2bd2cbb8f85b4383b#n1)。
- `Instance` 创建 Vulkan 1.2 实例，选择 compute queue，检查 FP16，加载 shader library：[lsfgvk.cpp 第 65–114 行](https://git.lsfg-vk.dev/lsfg-vk/tree/lsfg-vk-pipeline/src/lsfgvk.cpp?id=0e7a3898c1285b13df8596f2bd2cbb8f85b4383b#n65)。
- 创建 compute pipelines：[pipeline.cpp 第 524–560 行](https://git.lsfg-vk.dev/lsfg-vk/tree/lsfg-vk-pipeline/src/pipeline.cpp?id=0e7a3898c1285b13df8596f2bd2cbb8f85b4383b#n524)。
- 绑定 compute pipeline 并 `dispatch`：[pipeline.cpp 第 755–776 行](https://git.lsfg-vk.dev/lsfg-vk/tree/lsfg-vk-pipeline/src/pipeline.cpp?id=0e7a3898c1285b13df8596f2bd2cbb8f85b4383b#n755)。
- 插帧图的声明包括 mipmap、alpha/beta/gamma/delta/epsilon 多阶段，最后 `generate` 使用原始两帧与中间结果输出：[v3_1.cpp 第 23–49 行](https://git.lsfg-vk.dev/lsfg-vk/tree/lsfg-vk-pipeline/src/pipelines/v3_1.cpp?id=0e7a3898c1285b13df8596f2bd2cbb8f85b4383b#n23)、[第 541–563 行](https://git.lsfg-vk.dev/lsfg-vk/tree/lsfg-vk-pipeline/src/pipelines/v3_1.cpp?id=0e7a3898c1285b13df8596f2bd2cbb8f85b4383b#n541)。

### 3. 模型与权重并未作为可转换的公开训练图提供

当前 v2 从用户提供的商业 `lsfg-vk.dll` 读取 PE 的 RT_RCDATA 资源；其资源字节直接交给 Vulkan `createShaderModule`。同一 shader 名称有 quality/performance、FP32/FP16 不同资源 ID。

- 资源 ID 与 shader 名称、变体选择及加载：[library.cpp 第 20–53、69–112 行](https://git.lsfg-vk.dev/lsfg-vk/tree/lsfg-vk-pipeline/src/library.cpp?id=0e7a3898c1285b13df8596f2bd2cbb8f85b4383b#n20)。
- DLL 解析：[library/dll.cpp](https://git.lsfg-vk.dev/lsfg-vk/tree/lsfg-vk-pipeline/src/library/dll.cpp?id=0e7a3898c1285b13df8596f2bd2cbb8f85b4383b)。
- shader code 原样作为 `pCode` 传入：[vkhelper.cpp 第 142–151 行](https://git.lsfg-vk.dev/lsfg-vk/tree/lsfg-vk-helper/src/vkhelper.cpp?id=0e7a3898c1285b13df8596f2bd2cbb8f85b4383b#n142)。
- 插件明确不附带商业 DLL，用户须从 Steam 的 lsfg-vk 分支获取：[README 第 21–31 行](https://github.com/mydanyi/decky-lsfg-vk-arm64/blob/2da96d3365576fa23e6844c0f3161073dfceb9d8/README.md#L21)。

没有商业 DLL 本体，也没有官方的模型结构/权重导出，因此不能声称已核实 LSFG 网络的全部算子、张量尺寸或权重格式。公开源码只足以确认“GPU shader 实现与调度”，不构成一个可直接交给 QNN converter 的 ONNX 模型。精确迁移 LSFG 算法需要另获原始图/参数、可合法转换的表达，或重建后逐级验证。若使用开源 SR/VFI 模型则是替换算法，其画质和时延也必须重新验收。

注意：[2025 年移植博客](https://lsfg-vk.dev/blog/porting-lsfg-to-native-vulkan/) 描述过 DXBC → DXVK → Vulkan 的历史路径；当前 v2 源码已直接从专用 DLL 提取 shader code，不能照搬旧实现描述。

## 可实施的接入建议（工程判断）

### 最自然的后端边界

`lsfg-vk-layer/src/wrapper.cpp` 的 `InstanceWrapper/ContextWrapper` 与 `lsfgvk::Instance/Context` 之间已有清楚的组件边界，适合新增 `FrameBackend` 接口，保留原 Vulkan 实现并新增 QNN 实现。

当前 API 包括 `Context(width,height,flowScale,performanceMode)`、`exportFds()`、`dispatch(total)`、`acquire()`、`idle()`：[lsfgvk.hpp 第 75–164 行](https://git.lsfg-vk.dev/lsfg-vk/tree/lsfg-vk-pipeline/include/lsfg-vk/lsfgvk.hpp?id=0e7a3898c1285b13df8596f2bd2cbb8f85b4383b#n75)。这能复用输入/输出、时序概念，却不是现成的 provider/plugin backend：`Instance` 内部硬编码 Vulkan，pipeline 静态链接，不能只修改 Decky Python 配置就变为 NPU。

建议接口显式包含输入/输出尺寸、像素或张量格式、时间戳、缓冲所有权、同步对象、历史重置、取消与失败回退。插件补丁已经在临时原帧输出后重置两帧历史，NPU 实现必须保持同类语义：[0001-arm64-recovery.patch 第 1468 行起](https://github.com/mydanyi/decky-lsfg-vk-arm64/blob/2da96d3365576fa23e6844c0f3161073dfceb9d8/native/patches/0001-arm64-recovery.patch#L1468)。

### 内存互通是独立的关键工作

现有 `exportFds()` 给出 source/destination/sync FDs，其中输入是两层 RGBA8 optimal 图像，sync 是 timeline semaphore。helper 导出/导入的是 `VK_EXTERNAL_MEMORY_HANDLE_TYPE_OPAQUE_FD_BIT`：[vkhelper.cpp 第 280–305 行](https://git.lsfg-vk.dev/lsfg-vk/tree/lsfg-vk-helper/src/vkhelper.cpp?id=0e7a3898c1285b13df8596f2bd2cbb8f85b4383b#n280)、[第 580–615 行](https://git.lsfg-vk.dev/lsfg-vk/tree/lsfg-vk-helper/src/vkhelper.cpp?id=0e7a3898c1285b13df8596f2bd2cbb8f85b4383b#n580)。wrapper 导入同样指定 OPAQUE_FD：[wrapper.cpp 第 140–163 行](https://git.lsfg-vk.dev/lsfg-vk/tree/lsfg-vk-layer/src/wrapper.cpp?id=0e7a3898c1285b13df8596f2bd2cbb8f85b4383b#n140)。

这些 FD 不是已验证的 QNN 可用线性张量/共享缓冲，不能以“同一 SoC 共享 DRAM”推断零拷贝。需要分别验证图像线性化、颜色/量化布局、DMA-BUF/共享分配器、缓存一致性、GPU → NPU 和 NPU → GPU 同步。最初可用 staged copy 建立正确性，再优化共享内存；对比必须计入整个往返开销。

### 能转移的是图像增强工作，而非游戏渲染本身

layer 位于最终图像呈现附近，只获得成品图像；shader、栅格化、游戏几何、合成与显示依然有 GPU 工作。NPU 后端的目标应是卸载 SR/VFI 的神经网络部分；GPU copy、预处理/后处理、warping、合成等是否值得保留在 GPU 要按模型与实测决定。即使某 Vulkan 扩展能调用 QNN graph，也不会把当前 compute SPIR-V 自动变为一个 NPU 图。

## 发布边界

用户仓库插件为 BSD-3-Clause，但原生 v2 为 CC BY-NC-ND 4.0，商业 DLL 另有条款。插件 README 明确区分：[README 第 49–53 行](https://github.com/mydanyi/decky-lsfg-vk-arm64/blob/2da96d3365576fa23e6844c0f3161073dfceb9d8/README.md#L49)。上游明确要求衍生集成/独立分支联系作者：[Contributing / Derivatives](https://lsfg-vk.dev/docs/contributing/#derivatives)。

这里只作一手许可事实记录；不因公开可读就推断可随 Armada 发行修改的 v2 核心或重打包商业 shader。公开发布应单独确认适用许可/授权，或选用有合适许可的独立后端实现。

## 结论

源码层面存在可改造的接入位置，适合保留 Decky/呈现与调度框架、引入全新的 NPU 图像后端。但当前既无现成 NPU provider，也无可直接转换的公开 LSFG 模型；超分须另行新增。只有在 Snapdragon 8 Elite 的实际 Armada 内核/固件/用户态环境成功执行 NPU 图，并完成含图像往返与同步的游戏负载对比后，才能验证是否实现实际 GPU 卸载与端到端加速。

## 补充：data graph 的两图切分与时序

`v3_1.cpp` 在 beta4 后用 `s.split()` 分开 pre-pass / main-pass：[第 231–233 行](https://git.lsfg-vk.dev/lsfg-vk/tree/lsfg-vk-pipeline/src/pipelines/v3_1.cpp?id=0e7a3898c1285b13df8596f2bd2cbb8f85b4383b#n231)。按 `appendPass` 及循环推导，pre-pass 包括 mipmaps 1 次、alpha 7×4 次、beta 5 次，共 34 次逻辑 shader dispatch；main-pass 包括 gamma 7×5 次、delta 3×5 次、epsilon 3×5 次、generate 1 次，共 66 次。这里的次数不是 profiler 耗时，也不能用名称断言它们分别是卷积、光流或遮挡网络。

`Context::dispatch(total)` 每对输入帧提交一次 pre-pass；`Context::acquire()` 每张生成帧提交一次 main-pass。因此倍率 M 对应 `34 + 66×(M−1)` 次此类 dispatch；2×/3×/4×分别为 100/166/232。相关 API：[lsfgvk.cpp 第 171–225 行](https://git.lsfg-vk.dev/lsfg-vk/tree/lsfg-vk-pipeline/src/lsfgvk.cpp?id=0e7a3898c1285b13df8596f2bd2cbb8f85b4383b#n171)、[第 228–314 行](https://git.lsfg-vk.dev/lsfg-vk/tree/lsfg-vk-pipeline/src/lsfgvk.cpp?id=0e7a3898c1285b13df8596f2bd2cbb8f85b4383b#n228)。

若获得可正确表达这些计算的模型图，适合将新后端设计为 `prepare(previous,current) → resident_features` 和 `synthesize(features,previous,current,t) → generated_frame` 两个 graph。准备结果在多倍率输出期间驻留并复用；这是新图的建议边界，不是已有可转换模型。替代模型则必须按其实际结构设计，不能假装与现有 shader pass 等价。

时间参数现为 `t=(index+1)/(total+1)=k/M`：[pipeline.cpp 第 868–876 行](https://git.lsfg-vk.dev/lsfg-vk/tree/lsfg-vk-pipeline/src/pipeline.cpp?id=0e7a3898c1285b13df8596f2bd2cbb8f85b4383b#n868)。数据图后端可保持这些时刻及既有呈现契约，但模型要能生成对应 t；仅 midpoint 的 ANVIL 不能直接保证 3×所需的 1/3、2/3。结果必须在 graph 和 GPU 后处理均完成后再 signal，并在消费者读取前禁止覆盖输出。

`VK_QCOM_data_graph_model` 确实可通过 Vulkan 执行 QNN 模型；适配需要新的 graph pipeline、tensor descriptor、session、支持目标引擎的 queue/pool，以及兼容的外部内存和 semaphore。对 data-graph-only queue，普通 compute barrier/transfer 不能原样录入；存在依赖的 graph dispatch 需按规范分 submit 和 semaphore 同步。不能机械替换 `pipeline.cpp:776` 的 `vkCmdDispatch`。详细阶段表、可知/未知边界与规范链接见同目录 [LSFG-DATA-GRAPH-ADAPTER.zh-CN.md](LSFG-DATA-GRAPH-ADAPTER.zh-CN.md)。
