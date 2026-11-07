# K230 CPU / 体系结构注意点（cpu-notes）

> 原始资料见 `ref/`。行末括号标注来源。

## 1. 双核异构分工

| 核 | 频率 | 指令集 | Cache | 跑什么 |
|---|---|---|---|---|
| CPU1 = 大核 | 最高 1.6GHz | RV64 + RVV 1.0（128bit 向量）+ FPU | L1 32K+32K，**L2 256KB** | RT-Smart + CanMV/MicroPython + MPP 多媒体 + AI |
| CPU0 = 小核 | 最高 800MHz | RV64GCB + FPU | L1 32K+32K，**L2 128KB** | Linux(buildroot)（仅双系统 SDK 镜像）；CanMV 镜像里无 Linux 负载 |

- 两核都有 MMU、PLIC（208 外部中断源）、JTAG。
- 启动优化文档确认频率分工："k230_sdk 默认大核 1.6GHz，小核 800MHz"（`ref/k230-boot-opt.md`）。
- datasheet 中 CPU0=800MHz / CPU1=1.6GHz RVV（`ref/k230-datasheet.md` "CPU subsystem"）——即 **RVV 在 CPU1（大核）**，CanMV 的 Python 层跑在大核上。
- 中断/时钟：CMU 管理时钟，外部 24MHz OSC + 4 个 PLL；**CPU1 与 KPU 支持 DVFS**，二者可用同一时钟源同步变频（`ref/k230-datasheet.md` system component）。

## 2. RVV 1.0 向量扩展：什么时候有用

- 官方定位：nncase 无法被 KPU 加速的算子落到 CPU 执行，RVV 大幅提升这类算子（如 Transformer 的 LayerNorm/Softmax 类）；KPU 之外的"CPU 算子兜底"是主要受益方（`ref/k230-rvv-perf-notes.md`）。
- CanMV changelog 显示 RVV 已用在：图像处理（memcpy/内存操作类 HAL）、PUF 固件、SPI 驱动加速等（`ref/canmv-k230-changelog.md`）；v1.6 release note 明确"系统级 RVV 性能优化"。
- 对我们：纯 Python/ulab 的循环（色彩变换、NMS、前处理）会比无 RVV 平台快，但**别指望用 Python 手写算子打平 KPU**；重活应交给 KPU/AI2D/2D 引擎。
- RVV 实战文档（`ref/k230-rvv-practice.md`）给出了 C 层 intrinsics 优化案例，MicroPython 用户层不可直接调用，仅固件移植时相关。

## 3. 内存布局

### 3.1 硬件
- DDR：LPDDR4，16bit × 2 通道，最高 3200Mbps；最高容量 2GB。LPDDR3 32bit 2133Mbps 也支持（`ref/k230-datasheet.md`）。
- 片内 SRAM：**2MB 共享 SRAM**（CPU/KPU/解压模块共享，两组 128bit AXI slave，KPU 端口优先）+ **2MB 默认给 KPU**；解压模块独占其中 768KB 缓冲（`ref/k230-datasheet.md`）。
- 庐山派 K230-CanMV 板：1GB LPDDR4（板载）；同家族 Lite-K230D 为 K230D SiP 内置 128MB（`ref/lushan-quick-start.html`）。

### 3.2 软件划分
- 双系统 SDK：见 os-firmware.md 第 5 节分区表（LINUX/RTT/MMZ/AI_MODEL 各占一块，menuconfig 可调，基址需 2MB 对齐）。
- CanMV（RT-Smart 单系统）：无 Linux 分区；多媒体帧 buffer 全部走 **MMZ/VB 物理连续内存池**，由 `MediaManager` 统一申请释放；内存占用分析方法见 `ref/k230-memory-analysis.md`（大核堆/页内存 + 多媒体专用内存三块）。
- 实践注意：
  - Python 侧 GC 堆与 MMZ 是两套内存；大帧（如 1080p RGB565 ≈ 4MB）必须由媒体池分配，不能 `malloc` 在 GC 堆里。
  - `MediaManager` buffer 不足会直接报错（常见于 osd_num/to_ide 开太多），显示参数见 camera-api.md。
  - 内存优化指南（`ref/k230-memory-opt.md`）讲的是 Linux 侧裁剪；CanMV 侧等价手段是降低分辨率/减少通道/减小 OSD 层数。

## 4. TF 卡启动流程

1. 上电 → **BootROM**：读 usip 参数分区，尝试从启动介质引导（SD 卡：分区 0 镜像）。
2. 失败 → 自动进 **USB 烧录模式**（"K230 USB Boot Device"），等待 BurnTool/IDE 烧写（`ref/k230-burntool-guide.md`）。
3. 成功 → u-boot（快起模式不进命令行）→ opensbi → 大核 RT-Smart/CanMV。
4. CanMV 首次启动：把 TF 卡除固件分区外的剩余空间格式化成 FAT 挂到 `/data`，并自动重启一次（`ref/lushan-quick-start.html`）。
5. MicroPython 起来后依次找 `/sdcard/boot.py` → `/sdcard/main.py` 自动执行（`ref/lushan-offline-run.html`）。

优化参考：`ref/k230-boot-opt.md`（bootrom 后先加载大小核镜像，裁剪镜像+减少驱动是提速关键；CanMV v1.7 也优化了 uboot/RT-Smart 启动时间）。

## 5. Cache / IO / 内存对齐注意

- L1 32K+32K、大核 L2 256KB：手动优化 C 层时按 32B/64B 对齐即可；官方建议在 **AI2D/2D 引擎的 DDR 地址按 16B 对齐**（datasheet D Engine NOTE2）。
- 共享 SRAM（2MB）可作为低延迟缓冲：AI2D 加速器可直接访问系统 SRAM；KPU port 优先级最高（datasheet SRAM 节）。
- CanMV 层面没有公开的用户态 cache 管理文档（clean/invalidate 由 MPP 驱动内部处理）——**未找到** MicroPython 层的 cache API，默认无需关心。
- 物理连续内存（MMZ）是 VENC/VO/VICAP DMA 的前提；Python 里体现为 MediaManager.Buffer（`ref/lushan-api-media.html`）。

## 6. KPU 约束与 nncase 转换路径

### 6.1 硬件能力
- 精度：**INT8 / INT16**，支持权重稀疏压缩；量化精度损失 <1%（datasheet）。
- 参考性能 @INT8：ResNet50 ≥85fps，MobileNetV2 ≥670fps，**YoloV5S ≥38fps**（datasheet；社区常引 ~6 TOPS 算力档）。
- 支持框架：TensorFlow/PyTorch/TFLite/PaddlePaddle/ONNX——但都要经 nncase 编译成 `.kmodel` 才能在板上跑。

### 6.2 算子覆盖
- datasheet 给出 ONNX 算子集（Conv/ConvTranspose/MatMul/LSTM/Resize/Where 等约 130 个，见 `ref/k230-datasheet.md` KPU 节）。
- nncase 侧算子清单：TFLite/ONNX 两张表（`ref/k230-nncase-dev-guide.md`）。**未被 KPU 支持的算子自动落 CPU（RVV 加速）**，混合执行由 nncase 的 Partition 完成。
- 常见坑：动态 shape、自定义 op、部分 NMS/decode 后处理会走 CPU，建议模型导出前把后处理留在模型外，或用 AI Demo 现成的 postprocess。

### 6.3 转换路径（简版，详见 nncase-ai.md）
```
PyTorch ──torch.onnx.export──► ONNX ─┐
TF/Keras ────────────────────► TFLite ┤
                                      ▼
              nncase compiler (PC, pip 装 nncase)
              PTQ 量化(校准集) + 前处理配置(CompileOptions)
                                      ▼
                                   .kmodel ──► 板上 nncase_runtime + KPU
```

## 7. 其他加速单元（相机链路相关）
- **AI2D**（独立硬件）：crop/shift/pad/resize/affine + CSC，NCHW/uint8，专门做模型前处理；MicroPython 里是 `Ai2d` 类。
- **2D GDMA/DEngine**：mirror/rotate(90/180/270)/CSC/OSD/draw border，最大 64K×64K。
- **DPU**：3D 结构光深度引擎（庐山派用不上）。
- **GPU**：2.5D（DPU 配套），非通用 3D。
- VPU 编解码：H.264/H.265/JPEG/MJPEG，最大 8MP@20fps 编码；JPEG 编码仅 baseline YUV420/YUV422（`ref/k230-datasheet.md` video codec）。

## 未找到 / 存疑
- KPU 的确切 TOPS 数值与频率文档未给出（datasheet 只给 fps；"~6 TOPS INT8"为社区/宣传口径）。
- CanMV 固件下大核 DVFS 是否默认开启：未找到明确说明。
- MicroPython 层 cache 控制接口：未找到（可能不存在）。
