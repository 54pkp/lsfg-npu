# 来源与校验记录

核查日期：2026-09-15。此清单保存可重新获取的来源，不收录第三方二进制或完整源码副本。详细主张在各报告中就近引用对应文件与行号。

## 固定源码基线

| 项目 | 提交 / 版本 | 作用 |
|---|---|---|
| [mydanyi/decky-lsfg-vk-arm64](https://github.com/mydanyi/decky-lsfg-vk-arm64/tree/2da96d3365576fa23e6844c0f3161073dfceb9d8) | `2da96d3365576fa23e6844c0f3161073dfceb9d8` | 插件、原生补丁及上游基线 |
| [lsfg-vk](https://git.lsfg-vk.dev/lsfg-vk/tree/?id=0e7a3898c1285b13df8596f2bd2cbb8f85b4383b) | `0e7a3898c1285b13df8596f2bd2cbb8f85b4383b` | 原生 Vulkan 管线、资源和时序 |
| [drewano/hexscale](https://github.com/drewano/hexscale/tree/ce4aaacf60e6370a5a7086e19828b52f36a84bd6) | `ce4aaacf60e6370a5a7086e19828b52f36a84bd6` | 原型与占位实现核查 |
| [54pkp/hexscale](https://github.com/54pkp/hexscale/tree/8fed58387dcd5b417910b71f15b12d3aa1e509fa) | `8fed58387dcd5b417910b71f15b12d3aa1e509fa` | 真实 QNN 适配、模型和验证范围 |
| [armada-os/armada](https://github.com/armada-os/armada/tree/f7eef8886d69af69fa5e9af015d7919a188aaacc) | `f7eef8886d69af69fa5e9af015d7919a188aaacc` | 系统构建和板级固件目录 |
| [armada-os/armada-packages](https://github.com/armada-os/armada-packages/tree/24d88394fd6ae777e7616c3ec245a7a35b5d2f81) | `24d88394fd6ae777e7616c3ec245a7a35b5d2f81` | 内核配置、设备树、Mesa 配方与补丁 |
| [Qualcomm Vulkan 示例](https://github.com/SnapdragonGameStudios/adreno-gpu-vulkan-code-sample-framework/tree/8177ad3c0498cc92a2588ef6c170148cbcf7ae16) | `8177ad3c0498cc92a2588ef6c170148cbcf7ae16` | Graph pipeline、引擎选择及 Android 内存实现 |
| [Hexagon DSP 配套文件说明](https://github.com/linux-msm/hexagon-dsp-binaries/tree/bbb0e6b59143e052dbf26b371ee239ab4b0881b0) | `bbb0e6b59143e052dbf26b371ee239ab4b0881b0` | DSP shell / 固件版本配套要求 |

## 已下载并校验的官方包

| 文件 | 字节数 | SHA-256 |
|---|---:|---|
| `onnxruntime_qnn-2.6.0-cp311-cp311-manylinux_2_34_aarch64.whl` | 82232800 | `189544b5e51ba4d3b3354b4e938f396a83dd833aba33e59d9f9486171ef8105c` |
| `mesa-26.2.2.tar.xz` | 68533264 | `eeb29ca7e56cfaa8e8a79538dcf834e3b18e501c31bef5145e959ea437cc4216` |

下载来源：[官方 PyPI 文件列表](https://pypi.org/project/onnxruntime-qnn/2.6.0/#files)、[对应 wheel](https://files.pythonhosted.org/packages/fc/0f/e1c6fc9c27ab58045a802c0e244c0e0c19cd16247a2548bce7a8e322ecb8/onnxruntime_qnn-2.6.0-cp311-cp311-manylinux_2_34_aarch64.whl)、[Mesa 官方发布包](https://archive.mesa3d.org/mesa-26.2.2.tar.xz)。Mesa 的预期校验值来自上述固定版本 `armada-packages/mesa/BASE.env`。

QNN 包只做 ZIP 内容和 ELF 头检查，没有执行：`libQnnHtp.so` 与 `libQnnHtpV79Stub.so` 的 `e_machine=183`（AArch64），`libQnnHtpV79Skel.so` 的 `e_machine=164`（Hexagon）。文件存在不等于已在 SM8750/Armada 初始化成功。

Mesa 包检查范围包括 `src/freedreno/vulkan/`、`docs/features.txt` 和 Armada 自有三个补丁。扩展支持表位于 `src/freedreno/vulkan/tu_device.cc` 的 `get_device_extensions()`。未完整审计构建中引用的 Fedora SRPM 自带补丁，也未检查目标设备实际安装的 ICD。

下载完成后，可在 Linux 使用 `sha256sum 文件` 或在 PowerShell 使用 `Get-FileHash -Algorithm SHA256 文件` 比对。仅下载并读取包不构成模型或设备的性能测试。

## 官方开发与规范入口

- [QAIRT / QNN SDK](https://www.qualcomm.com/developer/software/qualcomm-ai-engine-direct-sdk)
- [QNN 设备与工具链矩阵](https://docs.qualcomm.com/doc/80-63442-10/topic/QNN_general_overview.html)
- [Hexagon NPU SDK](https://www.qualcomm.com/developer/software/hexagon-npu-sdk)
- [Qualcomm FastRPC](https://github.com/qualcomm/fastrpc)
- [ONNX Runtime QNN EP](https://github.com/onnxruntime/onnxruntime-qnn/blob/main/docs/execution_providers/QNN-ExecutionProvider.md)
- [VK_QCOM_data_graph_model](https://docs.vulkan.org/refpages/latest/refpages/source/VK_QCOM_data_graph_model.html)
- [VK_ARM_data_graph](https://docs.vulkan.org/refpages/latest/refpages/source/VK_ARM_data_graph.html)
- [VK_ARM_tensors](https://docs.vulkan.org/refpages/latest/refpages/source/VK_ARM_tensors.html)
- [VK_ARM_data_graph_instruction_set_tosa](https://docs.vulkan.org/refpages/latest/refpages/source/VK_ARM_data_graph_instruction_set_tosa.html)
- [VK_ARM_data_graph_optical_flow](https://docs.vulkan.org/refpages/latest/refpages/source/VK_ARM_data_graph_optical_flow.html)
- [SPV_ARM_graph](https://github.khronos.org/SPIRV-Registry/extensions/ARM/SPV_ARM_graph.html)

这些未锁定版本的文档可能更新；报告中的事实限定于核查日期。模型、论文及其测量条件另见 [模型笔记](model-notes.md)。
