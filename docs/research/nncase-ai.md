# nncase 与 AI 进阶（nncase-ai）

> 来源：`ref/k230-nncase-dev-guide.md`、`ref/k230-nncase-version-map.md`、`ref/k230-nncase-faq.md`、`ref/k230-ai-demo-intro.md`、`ref/lushan-ai-demo-profile.html`、`ref/canmv-k230-tutorial.md`、`ref/canmv-examples/20-YOLO-*.py`、`ref/k230-rvv-perf-notes.md`。

## 1. nncase 是什么

面向 AI 加速器（K210/K510/K230/CPU）的神经网络编译器，分两半：

- **Compiler**（PC 侧，pip 装 `nncase`）：导入 TFLite/ONNX → IR/优化/量化(PTQ)/tiling → 生成 **.kmodel**（平坦化、零拷贝加载、静态内存分配）。
- **Runtime**（板上）：`nncase_runtime` Python 模块 / C++ 库，加载 kmodel 推理；K230 上 KPU 部分 8/16bit 量化执行，CPU 兜底算子走 RVV 加速。

只支持两种输入格式：**TFLite 和 ONNX**——PyTorch/TensorFlow/PaddlePaddle 都要转成 ONNX/TFLite 再喂（datasheet 里"支持 TF/PyTorch"指的是经转换链）。

## 2. 模型转换路径（PyTorch/ONNX → kmodel）

```
PyTorch ──torch.onnx.export()──►  ONNX ─┐
TF/Keras ────────tf.lite─────────► TFLite ┘
                                        │
        pip install nncase  (PC, 版本要与固件 runtime 对应)
                                        ▼
  Compiler: CompileOptions(预处理/动态shape) → import_onnx/import_tflite
            → use_ptq(PTQTensorOptions + 校准集 set_tensor_data)
            → compile() → kmodel
                                        ▼
        板上: nncase_runtime.load_model(kmodel) + AI2D 前处理 + KPU 推理
```

要点（`ref/k230-nncase-dev-guide.md`）：
- **PTQ 训练后量化**：给几十~几百张校准图（float tensor），`PTQTensorOptions` 设 quantize_type（默认 int8），损失 <1%（datasheet）。
- **CompileOptions.preprocess**：可在转换时挂前处理（rgb swap / scale / mean / std），把归一化编进模型，省板端代码。
- **动态 shape**：CompileOptions 支持配 input shape range；但 K230 KPU 部署建议固定 shape。
- 版本对应（`ref/k230-nncase-version-map.md` + GitHub release 资产名）：K230 SDK 1.8.0→nncase 2.9.0、2.0.0→2.10.0；**CanMV 固件 v1.8 内置 nncase 2.11.0**。转换用 nncase 版本要与板上 runtime 兼容，官方 FAQ（`ref/k230-nncase-faq.md`）专门讲版本错配问题。
- 编译环境：PC x86_64 Linux/Windows，Python 3.x；完整脚本示例在 `ref/canmv-k230-tutorial.md`（"模型编译和模拟器推理"章节，含 nncase 模拟器跑法）。

## 3. 板上推理（CanMV/MicroPython）

两条层：

### 3.1 裸 nncase_runtime（自己写全流程）
```python
import nncase_runtime as nn
import ulab.numpy as np
interp = nn.interpreter()
interp.load_model(kmodel_bytes)          # 零拷贝读入
# 输入 tensor 用物理地址共享时, AI2D 输出直接作 KPU 输入（省一次拷贝）
```
- C++ 版教程同构：AI2D builder 配 crop/shift/pad/resize/affine → `ai2d_builder->build_schedule()`；KPU 侧 `host_runtime_tensor::create(..., hrt::pool_shared)`（`ref/canmv-k230-tutorial.md` nncase 推理节）。

### 3.2 AI Demo 框架（庐山派例程主流写法）
- 继承 `AIBase` 写任务类，四类任务：单模型/多模型/自定义预处理/无预处理（表格见 `ref/lushan-ai-demo-profile.html`）。
- `PipeLine` 封装：摄像头双通道——YUV420SP 直通 Display、RGBP888 喂 AI；`pl.get_frame()` 取帧、`pl.osd_img` 叠加结果、`pl.show_image()`。
- `Ai2d` 类：硬件前处理 crop/shift/pad/resize/affine，`set_ai2d_dtype(nn.ai2d_format.NCHW_FMT, ...)`。
- 模板代码骨架：`config_preprocess()` 里 `self.ai2d.resize(nn.interp_method.tf_bilinear, nn.interp_mode.half_pixel)` → `self.ai2d.build(in_shape, out_shape)`；用户只重写 `postprocess()`/`draw_result()`。
- AI Demo 目录（wiki ai-demo）：人脸（识别/分割/特征点/3D/朝向/注册/注视）、人体（检测/关键点/跌倒）、人手（检测/方向/关键点/分类/手势/石头剪刀布）；介绍文档 `ref/k230-ai-demo-intro.md`。

## 4. YOLO 在庐山派上的现状

- canmv_k230 固件自带整套 YOLO 封装与例程：`resources/examples/20-YOLO-Module-Examples/`，覆盖 **YOLOv5 / YOLOv8 / YOLO11 / YOLO26** × {cls, det, obb, pose, seg} × {image, video}（GitHub 仓库可见，存档例程 `ref/canmv-examples/yolo11_det_video.py`、`yolo26_det_video.py`）。
- 模型路径习惯：`/sdcard/examples/kmodel/*.kmodel`（如 `fruit_det_yolo11n_320.kmodel`）。
- 例程代码全貌（yolo11 检测，视频模式）：
```python
from libs.PipeLine import PipeLine, ScopedTiming
from libs.YOLO import YOLO11
pl = PipeLine(rgb888p_size=[640,360], display_mode="auto", display_size=None)
pl.create()                               # 可传 sensor_id=2 指定摄像头
yolo = YOLO11(task_type="detect", mode="video", kmodel_path=...,
              labels=[...], rgb888p_size=[640,360], model_input_size=[320,320],
              display_size=pl.get_display_size(), conf_thresh=0.5, nms_thresh=0.45)
yolo.config_preprocess()
while True:
    img = pl.get_frame()
    res = yolo.run(img)
    yolo.draw_result(res, pl.osd_img)
    pl.show_image()
    gc.collect()
```
- 自己训的 YOLO：ONNX 导出后按上面第 2 节转 kmodel（社区有现成转换脚本/notebook），后处理参数（anchors/conf/nms）在 `libs/YOLO*.py` 里改。
- 性能锚点：datasheet YoloV5S ≥38fps @INT8（KPU）；YOLO11n-320 这类小模型在 CanMV 例程定位为实时。

## 5. 相关工具链与文档

| 工具/文档 | 位置 | 用途 |
|---|---|---|
| nncase 仓库 | github.com/kendryte/nncase | compiler/runtime 源码与 docs/ |
| nncase Runtime API（CanMV 版） | `ref/lushan-api-nncase-runtime.html` | interpreter/tensor/ai2d Python API |
| K230 nncase 开发指南 | `ref/k230-nncase-dev-guide.md` | 全量 CompileOptions/PTQ/API 细节 |
| HHB 部署工具 | `ref/k230-hhb-deploy.md` | 平头哥系工具（另一条部署路线，了解即可） |
| AI Demo 源码 | canmv_k230 仓库 `port/ai_demo/`（kws/tts_zh 等）+ TF 卡 /sdcard/examples/05-AI-Demo | 直接抄现成任务类 |
| nncase FAQ | `ref/k230-nncase-faq.md` | 版本错配/精度排查 |

## 6. 给本项目的落地建议

1. 拍相机先落地（camera-api.md），AI 阶段直接用 PipeLine/AIBase 框架，别裸写 nncase。
2. 模型尽量固定输入 shape、INT8 PTQ、后处理放 Python 侧（ulab/numpy），与官方 demo 结构一致最好抄。
3. 转换 nncase 版本优先对齐固件内置 2.11.x（以 release notes/官方 FAQ 为准）。
4. 显示分辨率与 AI 分辨率解耦：`rgb888p_size`（喂 AI，宽 16 对齐）与 `display_size` 分开传。

## 未找到 / 存疑
- nncase 2.11 的 release notes 细节（相对 2.9/2.10 的算子/量化变化）未抓取（GitHub nncase releases 可后补）。
- KPU 对 INT16 的完整算子覆盖表（datasheet 只说支持 INT8/INT16 精度）。
- 庐山派实测 YOLO fps 数字：官方未给，需实测。
