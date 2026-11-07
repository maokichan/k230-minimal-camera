# K230 极简相机 · 官方资料调研索引（INDEX）

> 目标板：立创·庐山派 K230-CanMV（1GB），固件路线 CanMV（MicroPython）。
> 原始文件全部在 `ref/`；本目录各主题总结见下方"关键结论"（时间线由 git 承载，不在正文记日期）。
> 嘉楠官网（kendryte.com / developer.canaan-creative.com）在调研环境 DNS 不可达，改用 GitHub kendryte 组织 + 立创 wiki，两者即官方一手来源。

## 1. 主题总结（本目录）

| 文件 | 内容 |
|---|---|
| `os-firmware.md` | CanMV / Linux+RT-Smart SDK / RTT-only 三条路线、启动链、能力边界、固件版本与下载、内存划分 |
| `cpu-notes.md` | 双核分工、RVV 1.0、DDR/SRAM 布局、TF 启动流程、cache/对齐、KPU 约束与转换路径 |
| `camera-api.md` | Sensor/Image/MediaManager/Display/VENC API、最小拍照代码、JPEG 两条路、TF 卡路径 |
| `board-lushan.md` | 烧录流程（BurningTool/Web IDE）、IDE/串口、离线运行、板载外设（GC2093/WiFi/屏）、庐山派固件差异 |
| `nncase-ai.md` | onnx/tflite→kmodel 转换、AI Demo 框架、YOLO 例程现状、版本对应 |

## 2. 原始资料清单（ref/）

### 2.1 嘉楠 k230_docs（github.com/kendryte/k230_docs, raw 存档）

| 文件 | 来源 URL（github raw, main 分支 zh/ 下同名文件） | 一句话内容 |
|---|---|---|
| `k230-datasheet.md` (81K) | zh/00_hardware/K230_datasheet.md | K230 产品 datasheet：CPU/KPU/DPU/内存/ISP/VENC/外设规格 |
| `k230-hardware-design-guide.md` (39K) | zh/00_hardware/K230_硬件设计指南.md | 硬件设计指南（电源树/接口设计约束） |
| `k230-demo-board-resource-guide.md` (15K) | zh/00_hardware/K230_DEMO_BOARD资源使用指南.md | CanMV-K230 DEMO 板资源使用 |
| `canmv-k230-v1.0-schematic.pdf` (3.0M) | zh/00_hardware/CanMV_K230/CanMV-K230-V1.0_2023-09-15.pdf | 嘉楠 CanMV-K230 V1.0 板原理图 PDF |
| `k230-sdk-user-manual.md` (17K) | zh/01_software/board/K230_SDK_使用说明.md | K230 SDK 架构/编译/烧录/内存 menuconfig |
| `k230-sdk-version-notes.md` (34K) | zh/03_other/K230_SDK_版本说明.md | K230 SDK 各版本改动记录 |
| `k230-canmv-board-demo-guide.md` (52K) | zh/01_software/board/examples/K230_SDK_CanMV_Board_Demo使用指南.md | CanMV 板 C/MPP 例程指南 |
| `k230-vicap-api.md` (73K) | zh/01_software/board/mpp/K230_VICAP_API参考.md | VICAP 摄像头输入 C API 参考 |
| `k230-venc-api.md` (122K) | zh/01_software/board/mpp/K230_视频编解码_API参考.md | VENC/VDEC C API 参考 |
| `k230-sysctrl-api.md` (85K) | zh/01_software/board/mpp/K230_系统控制_API参考.md | vb/mmz/系统控制 C API |
| `k230-camera-sensor-porting.md` (23K) | zh/01_software/board/mpp/K230_Camera_Sensor适配指南.md | Sensor 驱动适配（换摄像头时用） |
| `k230-boot-opt.md` (9K) | zh/02_applications/tutorials/K230_启动优化指南.md | 启动顺序图、启动时间测量/裁剪 |
| `k230-memory-analysis.md` (24K) | zh/02_applications/tutorials/K230_内存占用分析指南.md | 大/小核内存占用分析方法 |
| `k230-memory-opt.md` (10K) | zh/02_applications/tutorials/K230_内存优化指南.md | 内存优化（Linux 侧） |
| `k230-rvv-perf-notes.md` (10K) | zh/02_applications/tutorials/K230_RVV优化性能说明.md | RVV 对模型推理的作用与实测 |
| `k230-rvv-practice.md` (11K) | zh/02_applications/tutorials/K230_RVV实战.md | RVV intrinsics 优化实战 |
| `k230-sharefs.md` (5K) | zh/02_applications/tutorials/K230_大小核通讯_sharefs用法介绍.md | 大小核文件共享机制 |
| `k230-debian-ubuntu.md` (11K) | zh/03_other/K230_debian_ubuntu说明.md | Debian/Ubuntu 镜像说明 |
| `k230-burntool-guide.md` (6K) | zh/01_software/pc/burntool/K230_SDK_Burntool使用指南.md | BootROM USB 烧录原理与工具 |
| `k230-nncase-dev-guide.md` (76K) | zh/01_software/board/ai/K230_nncase_开发指南.md | nncase 全量开发指南（转换+runtime+AI2D/KPU） |
| `k230-nncase-version-map.md` (2K) | zh/03_other/K230_SDK_nncase版本对应关系.md | SDK↔nncase 版本表 |
| `k230-nncase-faq.md` (5K) | zh/03_other/K230_nncase常见问题解答.md | nncase 常见问题 |
| `k230-ai-demo-intro.md` (13K) | zh/02_applications/ai_demos/K230_AI_Demo介绍.md | 官方 AI Demo 清单与运行方法 |
| `k230-hhb-deploy.md` (12K) | zh/02_applications/tutorials/K230_AI实战_HHB神经网络模型部署工具.md | HHB 部署工具教程 |
| `canmv-k230-tutorial.md` (91K) | zh/CanMV_K230_教程.md | CanMV-K230 官方教程（含 nncase 模型编译/推理全流程） |

### 2.2 canmv_k230 仓库（github.com/kendryte/canmv_k230, 分支 canmv_k230）

| 文件 | 来源 | 一句话内容 |
|---|---|---|
| `canmv-k230-readme.md` (2K) | README.md | CanMV 固件下载渠道与烧录入口 |
| `canmv-k230-build.md` (2K) | BUILD.md | 自编 CanMV 固件步骤 |
| `canmv-k230-changelog.md` (33K) | CHANGELOG.md | v1.0→v1.8 全量更新记录（RVV/板型/AI 改动） |
| `canmv-examples/`（11 个 .py/.h） | resources/examples/、port/boards/k230_canmv_lckfb/ | 官方例程：`camera_snapshot_and_save.py`（最小拍照）、`video_encoder.py`（VENC）、`mjpeg_web_server.py`、YOLO11/26 检测、手动曝光、庐山派板级配置 |

### 2.3 立创 wiki（wiki.lckfb.com/zh-hans/lushan-pi-k230/…，HTML 存档）

| 文件 | 对应页面 | 一句话内容 |
|---|---|---|
| `lushan-quick-start.html` | quick-start.html | 烧录/首次启动/串口/IDE 全流程（必看页） |
| `lushan-firmware.html` | open-source-hardware/canmv-k230-firmware.html | 固件页（stub，指向 canmv_k230 仓库） |
| `lushan-ide.html` | open-source-hardware/canmv-k230-ide.html | IDE 页（stub，指向 canmv_ide 仓库） |
| `lushan-board-profile.html` | open-source-hardware/profile.html | 板卡定位/供电设计 |
| `lushan-k230-vs-k230d.html` | k230vsk230d.html | 选型对比表（JS 渲染，正文未抓到） |
| `lushan-use-sensor.html` | image-recog/use-sensor.html | 摄像头教程 + 按键拍照存 TF 卡完整例程 |
| `lushan-display.html` | image-recog/display.html | 显示教程与 Display.init 参数 |
| `lushan-video.html` | media/video.html | MP4 录制/播放例程 |
| `lushan-wifi.html` | network/wifi.html | WiFi 连接例程（RTL8189FTV, 2.4G 限制） |
| `lushan-offline-run.html` | ide-usage/offline-run.html | boot.py/main.py 离线自启动 |
| `lushan-api-sensor.html` | api/mpp/k230_canmv_sensor_module_api.html | Sensor 模块 API 手册 |
| `lushan-api-venc.html` | api/mpp/k230_canmv_venc_module_api.html | VENC（H264/H265）API 手册 |
| `lushan-api-media.html` | api/mpp/k230_canmv_media_module_api.html | MediaManager API 手册 |
| `lushan-api-image.html` | api/openmv/image_module_api.html | Image 模块 API 手册（compress/save/draw） |
| `lushan-api-nncase-runtime.html` | api/nncase/k230_canmv_nncase_runtime_api.html | nncase_runtime API 手册 |
| `lushan-ai-demo-profile.html` | ai-demo/profile.html | AI Demo 开发框架（PipeLine/AIBase/Ai2d） |
| `lushan-faq.html` | faq.html | 常见问题（启动/TF 卡/串口/排线） |
| `lushan-micropython-beginner.html` | basic/micropython-beginner.html | MicroPython 基础 |

> 单文件 >20MB 的下载件无（最大 3MB 原理图 PDF）。固件 .img.gz（~292MB）按约定不落盘，仅记 URL：
> `https://github.com/kendryte/canmv_k230/releases/download/v1.8/CanMV_K230_LCKFB_micropython_v1.8-0-gc2d1f5c_nncase_v2.11.0.img.gz`

## 3. 关键结论速览（详见各主题文件）

### 主题 1 · OS/固件形态（os-firmware.md）
1. 官方三条路线：Linux+RT-Smart 双系统、RT-Smart 单系统、CanMV；CanMV = RT-Smart 上的 MicroPython（OpenMV 风格），跑在大核 1.6GHz，**无 Linux**。
2. CanMV 跑不了 Linux 程序、无官方裸机；高性能 C 需求要走 k230_sdk（另一套镜像/开发流）。
3. 启动链 BootROM→uboot→opensbi→RT-Smart/CanMV；仅 SD 卡启动，启动失败自动进 USB 烧录模式。
4. 固件用 GitHub release 的 `CanMV_K230_LCKFB_*.img.gz`（解压后 .img），最新 v1.8（2026-07，nncase 2.11.0）；micropython 镜像与 K230 SDK 镜像功能集不同，不可混用。
5. CanMV 内存无 Linux 分区，1GB 全归大核侧，帧 buffer 走 MMZ/VB 池（MediaManager 管理）。

### 主题 2 · CPU 注意点（cpu-notes.md）
1. CPU1（大核）1.6GHz 带 RVV 1.0（128bit）跑 CanMV；CPU0（小核）800MHz 跑 Linux（双系统 SDK 才有）；RVV 主要加速 KPU 兜底算子与图像 memcpy 类操作。
2. DDR 为 LPDDR4 双 16bit 通道（≤3200Mbps，≤2GB）；片内 SRAM 2MB 共享 + 2MB KPU；AI2D/2D 引擎地址需 16B 对齐。
3. TF 启动：BootROM 读 usip→uboot→大核；CanMV 首次启动把 TF 剩余空间格式化 FAT 挂 /data 并自动重启一次。
4. KPU：INT8/INT16 + 权重稀疏压缩；锚点性能 ResNet50 ≥85fps、YoloV5S ≥38fps；算子覆盖以 nncase 支持表为准，不支持的落 CPU(RVV)。
5. 转换链只有 ONNX/TFLite 两种入口，一切框架先转这两者再 nncase 编译成 kmodel。

### 主题 3 · 相机链路 API（camera-api.md）
1. 顺序铁律：`sensor.reset()` → 配置 → `Display.init` → `MediaManager.init()` → `sensor.run()` → `snapshot()`；退出反向 `sensor.stop()→Display.deinit()→MediaManager.deinit()`。
2. 庐山派默认摄像头在 CSI2：`Sensor(id=2)` 自动识别 GC2093；每路 3 通道（chn0/1/2）可同出不同格式/尺寸，AI 用 RGBP888、显示用 YUV420SP/RGB565。
3. 拍 JPEG 最短路径：`img = sensor.snapshot(); img.save("/data/xxx.jpg", quality=90)` 或 `img.compress(quality=95)` 后自己写文件。
4. 硬件 VENC（Python API）只暴露 H264/H265（最多 4 路，第 4 路给 IDE）；JPEG 静图走 image.save/compress，MP4 录制走 `Mp4Container`（H265+G711U）。
5. 保存路径：`/data`（FAT 大分区）与 `/sdcard`（U 盘可见，放 main.py/例程）；wiki 例程用 `/sdcard/images` + 编号防覆盖。

### 主题 4 · 庐山派专属（board-lushan.md）
1. 烧录：K230BurningTool + BOOT/RESET 进 BootROM（Zadig 装 WinUSB）；或 CanMV IDE Web；固件必须 .img（.gz 先解压）且带 LCKFB（K230 板）；K230D Lite 固件不可混烧。
2. IDE：VS Code CanMV 扩展（首推）/CanMV IDE Web；串口 USB CDC VID/PID 1209:abd1；同一时刻只能一个工具占串口。
3. 离线运行：`/sdcard/boot.py`（勿死循环）→ `/sdcard/main.py` 自动执行；IDE 菜单一键保存为 main.py。
4. 外设：GC2093 1080p60（三种镜头，CSI2 默认）、WiFi RTL8189FTV 仅 2.4G、ST7701 800x480 屏 / LT9611 HDMI、按键 GPIO53、RGB LED 62/20/63。
5. 庐山派固件 = canmv_k230 仓库 k230_canmv_lckfb 板级配置产物，与其他板同源同版本，仅板级差异；例程/库预装在 /sdcard。

### 主题 5 · nncase 与 AI（nncase-ai.md）
1. 转换路径：PyTorch/TF/Paddle → ONNX/TFLite → nncase compiler（PTQ+校准集）→ kmodel → 板上 nncase_runtime；转换版本要配固件（CanMV v1.8 = nncase 2.11.0）。
2. 板上推荐用官方 AI Demo 框架（PipeLine + AIBase + Ai2d），摄像头自动双通道：YUV420SP 直通显示、RGBP888 喂 AI；Ai2d 硬件前处理零拷贝接 KPU。
3. YOLO 现成例程覆盖 YOLOv5/v8/11/26 × cls/det/obb/pose/seg，模型放 `/sdcard/examples/kmodel/`，30 行代码可跑。
4. 自训模型：固定输入 shape、INT8 PTQ、后处理留在 Python 侧（ulab），结构对齐官方 demo 最省事。
5. KPU 不覆盖的算子自动落 CPU（RVV 加速），Anchor/decode/NMS 类后处理是主要落点。

## 4. 未找到的信息（明确缺口）

1. **K230 完整 TRM（寄存器手册）PDF**：公开渠道未见；datasheet 只有 markdown 摘要版。
2. **嘉楠官网在线文档站**（developer.canaan-creative.com、kendryte.com/k230_canmv 文档站）：调研环境 DNS 不可达，未能抓取；内容以 GitHub 仓库文档为准。
3. **庐山派 1GB CanMV 固件的精确内存分区/VB 池数值**：官方文档未给，需上板看启动日志。
4. **k230vsk230d 选型表正文**：wiki 页 JS 渲染未抓到（差异要点已从 datasheet/quick-start 补齐）。
5. **KPU TOPS 官方数字与 INT16 完整算子表**：datasheet 只给 fps 与 INT8/INT16 能力，无 TOPS 标称。
6. **img.compress 底层实现（软编 or 硬 JPEG）与单帧拍照耗时**：官方未写明，需实测。
