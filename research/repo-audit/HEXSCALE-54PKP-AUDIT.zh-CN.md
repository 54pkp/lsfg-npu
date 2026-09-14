# 54pkp/hexscale 对 lsfg NPU 后端的参考价值

审计日期：2026-09-15。固定提交：`8fed58387dcd5b417910b71f15b12d3aa1e509fa`，开发版 `0.2.0-dev1`。已下载并阅读源码；下述历史测试来自作者的记录，本次没有重跑其 Windows/ARM64/NPU 测试。

## 核心结论

此分支已经提供可复用的真实 QNN 调用代码、超分帧处理与离线插帧原型，比仅有 QNN 占位接口的设计更进一步。它仍没有证明 Snapdragon 8 Elite + Armada 的实时 NPU 加速，也没有自动游戏插帧功能。不能把它称为可直接接管 lsfg GPU 工作的成熟后端。

该结论同时由代码和作者明确记录支持：[README](https://github.com/54pkp/hexscale/blob/8fed58387dcd5b417910b71f15b12d3aa1e509fa/README.md)、[验证 JSON](https://github.com/54pkp/hexscale/blob/8fed58387dcd5b417910b71f15b12d3aa1e509fa/docs/validation/0.2.0-dev1.json)。后者标注 `arm64_runtime_tested=false`、`anvil_w8a8_calibrated=false`、`automatic_game_frame_generation=false`。

## 真实实现与尚未完成的部分

| 项目 | 源码验证 | 对接含义 |
|---|---|---|
| QNN 图执行 | 动态加载 provider，支持 DLC/context，最终调用 `graphExecute` | 可提取为 QNN 后端基础库 |
| I/O | 仅静态 FLOAT32、每张量 UINT8/INT8；`QNN_TENSORMEMTYPE_RAW` + 主机 storage | 不是共享 DMA-BUF/零拷贝后端 |
| SR | 读取 XLSR 真实形状，分块推理，重采样到请求尺寸 | 可作为正确性基线，需要重新优化整帧时延 |
| 游戏帧路径 | Vulkan present 中同步 readback → socket daemon → upload | 可演示图像通路；会阻塞呈现，不代表卸载后更快 |
| 游戏分辨率 | 请求输出与输入同宽高，swapchain 尺寸保持 | 尚不能自动用低渲染分辨率换取 GPU 帧率收益 |
| 插帧 | 命令行读入前后 RGBA 帧和外供 FLOW.f32，输出一个 midpoint | 离线原型，缺运动源和多帧呈现调度 |
| 运动 | Vulkan 平滑已有向量；CPU 预对齐 | 平滑算法不是从两张游戏画面估计光流 |
| 目标机验证 | Windows QNN CPU/RTX 4060、交叉编译记录 | 不能据此证明 HTP/NPU/Armada 兼容或性能 |

### QNN 调用确实存在

`daemon/src/qnn_runtime.cpp` 检查 provider 接口、创建 backend/context、读取张量信息；[第 451–478 行](https://github.com/54pkp/hexscale/blob/8fed58387dcd5b417910b71f15b12d3aa1e509fa/daemon/src/qnn_runtime.cpp#L451) 最终调用真实 `api.graphExecute(...)`。这不是仅返回模拟结果的接口。

但 [第 70–121 行](https://github.com/54pkp/hexscale/blob/8fed58387dcd5b417910b71f15b12d3aa1e509fa/daemon/src/qnn_runtime.cpp#L70) 只支持静态形状及有限类型，使用 `QNN_TENSORMEMTYPE_RAW` 和 `clientBuf`。`QnnHtpBackend::execute_dmabuf` 明确返回未实现：[qnn_backend.cpp](https://github.com/54pkp/hexscale/blob/8fed58387dcd5b417910b71f15b12d3aa1e509fa/daemon/src/qnn_backend.cpp#L80)。

此外名称 `QnnHtpBackend` 不等于本次执行必然在 HTP：它可加载不同 QNN backend，且存在显式 `"cpu"` 双线性开发模式。作者记录的真实模型执行验证使用 Qualcomm QNN CPU provider；没有用 CPU 回退冒充 NPU，但也不能把记录当 NPU 结果：[README-QNN](https://github.com/54pkp/hexscale/blob/8fed58387dcd5b417910b71f15b12d3aa1e509fa/README-QNN.md)。

### Vulkan 图像交换是同步复制

[layer_entry.cpp 第 115–178 行](https://github.com/54pkp/hexscale/blob/8fed58387dcd5b417910b71f15b12d3aa1e509fa/layer/src/layer_entry.cpp#L115) 在 GPU 图像与 host-visible/coherent buffer 之间调用 `vkCmdCopyImageToBuffer` / `vkCmdCopyBufferToImage`，并用 `vkWaitForFences(..., UINT64_MAX)` 同步等待。

[第 350–407 行](https://github.com/54pkp/hexscale/blob/8fed58387dcd5b417910b71f15b12d3aa1e509fa/layer/src/layer_entry.cpp#L350) 每帧创建临时 `Copy` 资源，把 mapped 内容拷贝到 `std::vector`，调用 `frames::request`，拷贝回 GPU buffer，再上传后呈现。此处理期间还持有 device mutex。它是验证优先的完整 CPU 往返路径。

[第 386 行](https://github.com/54pkp/hexscale/blob/8fed58387dcd5b417910b71f15b12d3aa1e509fa/layer/src/layer_entry.cpp#L386) 明确请求 `input.width, input.height`；[第 294–325 行](https://github.com/54pkp/hexscale/blob/8fed58387dcd5b417910b71f15b12d3aa1e509fa/layer/src/layer_entry.cpp#L294) 创建 swapchain 时只添加 transfer usage，不改变尺寸。因此它尚未实现渲染/显示分辨率分离。

### SR 分块次数不能忽视

随附 XLSR 为 128×128 → 512×512、W8A8，目标 context 为 SoC 69、Hexagon v79，使用 QAIRT 2.45.0.260326：[MODEL-MANIFEST.json](https://github.com/54pkp/hexscale/blob/8fed58387dcd5b417910b71f15b12d3aa1e509fa/models/MODEL-MANIFEST.json)。

[frame_processor.cpp 第 56–73 行](https://github.com/54pkp/hexscale/blob/8fed58387dcd5b417910b71f15b12d3aa1e509fa/daemon/src/frame_processor.cpp#L56) 使用 8 像素 halo，实际步长是 112×112，并逐块同步调用模型。按该代码计算，1280×720 输入需要 `ceil(1280/112) × ceil(720/112) = 84` 次图执行，1920×1080 输入需要 `18 × 10 = 180` 次；这只是调用次数推导，不是时延测量。模型 4×输出再被 CPU 重采样到目标尺寸，所以不能引用单个 128×128 patch 的推理时延宣称整帧低于 1 ms。

### ANVIL 还缺游戏运动源

[cli/src/interpolate.cpp 第 129 行起](https://github.com/54pkp/hexscale/blob/8fed58387dcd5b417910b71f15b12d3aa1e509fa/cli/src/interpolate.cpp#L129) 明确读取 `PREVIOUS.rgba CURRENT.rgba FLOW.f32` 三个文件。既有 Vulkan 代码只对 FLOW 中的外供粗向量执行中值/高斯平滑；不是从图像估计光流。

[frame_processor.cpp 第 97–164 行](https://github.com/54pkp/hexscale/blob/8fed58387dcd5b417910b71f15b12d3aa1e509fa/daemon/src/frame_processor.cpp#L97) 在 CPU 上用前向 flow 的 ±1/2 做预对齐，拼接 6 通道张量，执行 QNN 残差网络，再把残差加到对齐帧均值。模型要求输入输出精确匹配帧宽高；随附模型为 1080p FP32，不是已校准的 INT8 游戏插帧网络。

lsfg 的 present 层只有最终图像，没有通用可用的引擎 motion vectors 或视频解码器运动向量。因此选 ANVIL 路线还要实现图像光流估计或按引擎接入运动矢量，并把该成本计入 NPU 收益。它不会因替换 LSFG 最后一个 shader 就得到完整插帧。

## 建议复用方式

1. 优先独立使用 `daemon/include/qnn_runtime.hpp` / `daemon/src/qnn_runtime.cpp`、模型 manifest/probe 与转换工具，在真实 Armada 上证明所选 QNN HTP runtime 可以加载模型并得到正确结果。
2. 把 `frame_processor.cpp` 的 SR 分块/布局转换与参考结果测试作为正确性基线；随后移除逐帧临时分配、复用 buffers、减少 host 格式转换与图调用次数。
3. 在 lsfg 的 `ContextWrapper ↔ Context` 边界引入新的后端接口。可保留原有呈现/倍帧时序与失败回退，而不是同时叠加两个修改 `vkQueuePresentKHR` 的 layer，造成重复 readback、同步和顺序复杂性。
4. 若首先做 SR，需在引擎或 Gamescope/合成器侧实现低分辨率输入与高分辨率输出之间的分离。若首先做 VFI，先解决运动源并验证 2× midpoint，再扩展任意倍率/节奏恢复。
5. NPU 版与 GPU 版对比必须测游戏真实输入帧率、输出 frame pacing、端到端延迟、GPU busy、CPU开销与 SoC 总功耗；只比 graphExecute 时间不足以证明加速。

仓库代码标为 MIT；随包 XLSR 与 ANVIL 模型各有许可文件。具体重新分发模型与 Qualcomm runtime 仍应按对应来源处理，不应把仓库 MIT 标头扩展到 SDK/模型所有内容。
