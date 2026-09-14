# Qualcomm Hexagon / Armada Linux 平台核查

查询日期：2026-09-15。以下是公开资料及源码核查，不含目标设备实测。

## 结论

Linux ARM64 并非完全没有 Hexagon 支持；但“Hexagon NPU”不是使所有芯片、固件、系统 ABI、模型二进制都兼容的单一部署标准。Snapdragon 8 Elite / SM8750 / Hexagon v79 的 Armada 原型值得做，已经有内核配置、CDSP 设备树和板级固件基础，尚需证明 Linux QNN runtime + FastRPC 用户态 + DSP 动态库在具体掌机上能完成真实 HTP graphExecute。

## 官方 SDK 与目标矩阵

- [QAIRT/QNN 官方 Overview / Supported Snapdragon devices](https://docs.qualcomm.com/doc/80-63442-10/topic/QNN_general_overview.html)：SM8750 是 SoC model 69、Hexagon v79，支持工具链列仅 aarch64-android；SM8650 是 57/v75、SM8550 是 43/v73，亦列 Android。Dragonwing Q-8750 同为 v79、SoC 105，却明确列 aarch64-oe-linux-gcc11.2 和 Android；QCS/QCM8550、QCS9100 也列 Linux。这说明 Linux runtime 存在，但不能直接把移动芯片 Android 支持理解成 Armada 官方验证。X Elite/Plus 的 Windows 支持同样不能代替 Linux 支持。
- [Qualcomm AI Engine Direct SDK](https://www.qualcomm.com/developer/software/qualcomm-ai-engine-direct-sdk)：QNN 提供模型图 API、HTP 后端，也能由 ONNX Runtime / LiteRT 使用；适合神经网络推理。
- [Hexagon NPU SDK](https://www.qualcomm.com/developer/software/hexagon-npu-sdk)：原生 DSP / HVX 自定义代码及多媒体计算开发。页面“Windows & Linux”有开发主机语境，需另看每个目标包、系统与固件；不能据此承诺所有 Linux 掌机可运行。
- [Qualcomm FastRPC 源码](https://github.com/qualcomm/fastrpc)：官方现提供原生 Linux 与 Android 的不同编译流程。链路包含 CPU 用户库、内核驱动、rpmsg、DSP driver、DSP 用户 PD 与 skel。README 支持 Linux 原生编译；要求 FastRPC 与 DMA heap 权限。它本身不是 QNN、也不是自动将 Vulkan shader 编译到 NPU 的层。

## Armada 的已存在基础（固定 commit）

源码 HEAD：armada `f7eef8886d69af69fa5e9af015d7919a188aaacc`（2026-09-14）；armada-packages `24d88394fd6ae777e7616c3ec245a7a35b5d2f81`（2026-09-13）。源码状态不等于每位用户已安装的镜像状态。

1. [内核 config，第 57 行](https://github.com/armada-os/armada-packages/blob/24d88394fd6ae777e7616c3ec245a7a35b5d2f81/kernel/config/armada-kernel.config.overrides#L57)：`CONFIG_QCOM_FASTRPC=y`；同文件第 298 行 `CONFIG_QCOM_Q6V5_PAS=m`。
2. [Odin 3 设备树，第 242 行](https://github.com/armada-os/armada-packages/blob/24d88394fd6ae777e7616c3ec245a7a35b5d2f81/kernel/dts/cq8725s-ayn-odin3.dts#L242)：`remoteproc_cdsp` 为 `status = "okay"`，指向 `qcom/sm8750/ayn/odin3/cdsp.mbn`、`cdsp_dtb.mbn`；该设备 compatible 包括 cq8725s、sm8750，因此应运行时探测实际 SoC，不能只用商品名称。
3. [Odin 3 固件目录](https://github.com/armada-os/armada/tree/f7eef8886d69af69fa5e9af015d7919a188aaacc/system_files/usr/lib/firmware/qcom/sm8750/ayn/odin3)：包含上述 CDSP 固件及 cdspr.jsn。
4. [安装内核脚本](https://github.com/armada-os/armada/blob/f7eef8886d69af69fa5e9af015d7919a188aaacc/build_files/20-install-kernel.sh)将仓库固件复制到 `/usr/lib/firmware`。内核基于 kernel.org 加补丁，当前包 BASE.env 是 7.2.3。
5. 本次搜索完整 armada 文件树及 build_files 脚本，未找到显式 FastRPC 用户库、QNN/QAIRT 或 DSP shell 的安装项。基础镜像或传递依赖可能含有部分组件，因此只能说“未在显式配置中确认”，不能断言设备没有这些组件。

## 固件版本匹配是跨设备通用化的硬门槛

[linux-msm/hexagon-dsp-binaries README](https://github.com/linux-msm/hexagon-dsp-binaries/blob/bbb0e6b59143e052dbf26b371ee239ab4b0881b0/README.md)明确：linux-firmware 的 DSP 固件之外，FastRPC 还需要 DSP 侧 shell / 动态库；这些二进制绑定具体 SoC 和固件 revision，不能用一套文件覆盖所有目标。

[同仓 config 第 65 行](https://github.com/linux-msm/hexagon-dsp-binaries/blob/bbb0e6b59143e052dbf26b371ee239ab4b0881b0/config.txt#L65)已收录 SM8750-MTP 的 CDSP.HT.3.1-00889-PAKALA-1 配套文件，但这不证明与 Odin 3 自带 CDSP 固件匹配。Hamoa IoT EVK 也有对应条目，同样不能外推所有 X Elite 笔记本。

建议探测顺序：实际 SoC/Hexagon arch、系统 ABI → CDSP remoteproc running → FastRPC 权限与用户库 → 对应 shell/skel 和固件 revision → QNN HTP provider/backend/device → 加载本 SoC context → graphExecute 与黄金输出比较 → 稳定长跑/挂起恢复 → 整帧性能与 GPU/NPU 并行争用。

## 用户补充的 hexscale：不能当作已经完成设备验证的证据

### drewano/hexscale

HEAD `ce4aaacf60e6370a5a7086e19828b52f36a84bd6`。

[QNN backend 源码](https://github.com/drewano/hexscale/blob/ce4aaacf60e6370a5a7086e19828b52f36a84bd6/daemon/src/qnn_backend.cpp)说明目前是骨架：initialize 只 dlopen libQnnHtp.so，缺库仍返回 ready；load_context_binary 只把文件读入临时 vector，未创建 QNN context；execute_dmabuf 只计空函数耗时并返回 success；execute_host_memory 是 CPU 双线性循环。没有 QNN provider、contextCreateFromBinary、graphExecute，故 README 的零拷贝、720p→1080p <1ms/<1W 不构成实测证据。

[FastRPC session](https://github.com/drewano/hexscale/blob/ce4aaacf60e6370a5a7086e19828b52f36a84bd6/daemon/src/fastrpc_session.cpp)做设备 open/ioctl / mmap 等尝试，但这不会自动实现 QNN 执行；仅检测到 `/dev/accel/accel0` 就称为 QDA 也不构成完整驱动/ABI 验证。

### 54pkp/hexscale

HEAD `8fed58387dcd5b417910b71f15b12d3aa1e509fa`。

[README-QNN](https://github.com/54pkp/hexscale/blob/8fed58387dcd5b417910b71f15b12d3aa1e509fa/README-QNN.md)明确声明：目标 Odin3/Armada 没有真机测试。开发机为 Windows，已进行 QNN CPU 执行验证及 ARM64 ELF/依赖检查；这些不能算 NPU 测试。实际 SR 是同步 GPU→主机→daemon/QNN→主机→GPU，未实现 dma-buf 零拷贝，也不自动降低游戏渲染分辨率。

这份实现仍有可复用价值：真实 QNN runtime 调用、v79/SoC69 XLSR context、Linux 部署 wrapper、失败报错与 smoke probe。README 记录所取 SDK 为 QAIRT 2.45.0.260326，含 aarch64-oe-linux-gcc11.2 与 hexagon-v79/unsigned；包不分发 Qualcomm runtime/driver。

[qnn-wrapper](https://github.com/54pkp/hexscale/blob/8fed58387dcd5b417910b71f15b12d3aa1e509fa/packaging/qnn/qnn-wrapper)要求外部 Linux libQnnHtp.so / libQnnSystem.so，设置 LD_LIBRARY_PATH 与 ADSP_LIBRARY_PATH；[qnn_probe.cpp](https://github.com/54pkp/hexscale/blob/8fed58387dcd5b417910b71f15b12d3aa1e509fa/tests/qnn_probe.cpp)会 open/load/execute 并检查输出，可作为真机第一步，但仍需补充 profiling 证据确认 HTP 执行及实际质量/性能。Android .so 不能直接替代 glibc Linux .so；项目本身 glibc 2.28 基线不保证 SDK 的 ABI 兼容性。

准确结论：这些仓库降低了从零实现的成本；没有推翻“8 Elite + Armada 的整帧实时收益仍需真机验证”。上游 hexscale 不应被用来支撑 sub-ms 或功耗宣传，fork 更适合作为功能正确性与平台 smoke 的起点。

## 追加：VK_QCOM_data_graph_model 与 Armada 实际驱动核查

### 已读取官方样例

[Qualcomm graph_pipelines 样例](https://github.com/SnapdragonGameStudios/adreno-gpu-vulkan-code-sample-framework/tree/8177ad3c0498cc92a2588ef6c170148cbcf7ae16/samples/graph_pipelines)实际创建 tensor、data-graph pipeline/session 并提交到图队列。它为 Vulkan→模型图执行提供真实 API 示例，不能解读成自动接管任意现有 Vulkan shader。

- [application.cpp 第 89 行](https://github.com/SnapdragonGameStudios/adreno-gpu-vulkan-code-sample-framework/blob/8177ad3c0498cc92a2588ef6c170148cbcf7ae16/samples/graph_pipelines/code/main/application.cpp#L89)请求 VK_ARM_tensors、VK_ARM_data_graph、VK_QCOM_data_graph_model；还检查是否有图队列。默认图像尺寸 960×540→1920×1080，读取预编译 PipelineCache.bin。
- [QcomDataGraphModel.cpp 第 165 行](https://github.com/SnapdragonGameStudios/adreno-gpu-vulkan-code-sample-framework/blob/8177ad3c0498cc92a2588ef6c170148cbcf7ae16/samples/graph_pipelines/code/main/ml/QcomDataGraphModel.cpp#L165)优先选择 NEURAL_QCOM，但随后允许 COMPUTE_QCOM fallback；所以样例成功不能独自证明 NPU 卸载。要打印并核验 engine / operation、结合 profiling。
- [TensorResources.cpp 第 57 行](https://github.com/SnapdragonGameStudios/adreno-gpu-vulkan-code-sample-framework/blob/8177ad3c0498cc92a2588ef6c170148cbcf7ae16/samples/graph_pipelines/code/main/ml/TensorResources.cpp#L57)及第 111 行硬编码 ANDROID_HARDWARE_BUFFER 外部内存。Armada 的 glibc Linux 需改用被驱动验证支持的 DMA-BUF / FD 路径，并检查 tensor/buffer memory type、格式、布局、同步兼容性。
- 框架 README 主要构建目标是 Android / Windows；提及 Linux 为 WSL Ubuntu 构建环境，不是 SM8750 + Armada NPU 认证表。样例未提供可据以承诺支持的 SM8750/Linux 最低驱动版本。

### Mesa 发布源码提供了明确的当前版本结论

[Armada Mesa BASE.env（固定 commit）](https://github.com/armada-os/armada-packages/blob/24d88394fd6ae777e7616c3ec245a7a35b5d2f81/mesa/BASE.env)锁定 Mesa 26.2.2。已实际下载其指定的[官方发布源码包](https://archive.mesa3d.org/mesa-26.2.2.tar.xz)，实算 SHA-256 为 `eeb29ca7e56cfaa8e8a79538dcf834e3b18e501c31bef5145e959ea437cc4216`，与 BASE.env 完全一致。

已提取并核查整个 `src/freedreno/vulkan/` 及 `docs/features.txt`。`tu_device.cc` 第 191 行 get_device_extensions、第 204 行开始的结构体完整初始化只声明列出的扩展；三个 data graph / tensor 扩展均未启用。第 1831 行将该表安装到 physical device。整个 Turnip 源目录也没有相应 data graph / tensor 调度实现。已有 KHR_external_memory_fd（第 228 行）与 EXT_external_memory_dma_buf（第 329 行）不能补足缺失的 graph / NPU 扩展。

Armada 此 commit 的三个 Mesa 补丁已逐个读过：`0001-disable-turnip-sparse-sync.patch`、`0002-add-a830-chip-id.patch`、`0003-ir3-disable-bindless-ubo-const-lowering.patch`，仅改同步、A830 chip ID、UBO lowering；没有增添上述扩展。[补丁目录](https://github.com/armada-os/armada-packages/tree/24d88394fd6ae777e7616c3ec245a7a35b5d2f81/mesa/patches)。因此可以确认所核查的上游 Mesa 26.2.2 Turnip 不提供此路径，且 Armada 自有补丁未添加它；实际安装镜像、其他下游补丁及用户自行替换驱动仍应运行时核验。build.sh 还使用 Fedora `mesa-26.2.0-1.fc45` SRPM 提供打包规范；本次未解开该 SRPM 核查其自身附加补丁，以上确认范围是已下载的上游源码与 Armada 自有三补丁。

本地源码证据：`research/platform-sources/mesa-26.2.2/src/freedreno/vulkan/tu_device.cc`。这个结论来自实际源码的支持表及实现核查，不是搜索引擎没搜到结果。

### 运行时 gate

1. 从游戏实际使用的 ICD / physical device 枚举 VK_ARM_tensors、VK_ARM_data_graph、VK_QCOM_data_graph_model；记录驱动名、版本、设备 ID。
2. 查询 tensor、dataGraph、dataGraphModel 等 feature bits，不能只有扩展字符串。
3. 枚举支持 data graph 的队列；用 vkGetPhysicalDeviceQueueFamilyDataGraphPropertiesARM 查询 engine / operation。卸载 NPU 目标应选并记录 NEURAL_QCOM + 模型操作，不能悄悄用 COMPUTE_QCOM 代替。
4. Linux 外部资源检查：通过 vkGetPhysicalDeviceExternalTensorPropertiesARM 等查询 tensor 的 DMA_BUF/OPAQUE_FD import/export、格式/维度/usage、内存类型和 buffer/image 兼容性；如跨队列/处理器，同步与所有权转换也须明确。
5. 用可兼容的 QNN→QCOM pipeline-cache 模型实际 dispatch 并校验输出、profiling，再测并发游戏帧时长。普通 QNN context 不能未经转换假定就是这个 cache。

相关官方规范：[扩展](https://docs.vulkan.org/refpages/latest/refpages/source/VK_QCOM_data_graph_model.html)、[engine 类型](https://docs.vulkan.org/refpages/latest/refpages/source/VkPhysicalDeviceDataGraphProcessingEngineTypeARM.html)、[external tensor 能力参数](https://docs.vulkan.org/refpages/latest/refpages/source/VkPhysicalDeviceExternalTensorInfoARM.html)。
