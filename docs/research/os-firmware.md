# K230 操作系统 / 固件形态调研（os-firmware）

> 调研日期：2026-11。原始资料见 `ref/` 目录，行末括号标注来源文件或 URL。

## 1. K230 官方 SDK / 固件全景

K230 官方共有三条软件路线，均由嘉楠（Canaan/kendryte）维护：

| 路线 | 仓库 / 产物 | 跑在哪个核 | 关系 |
|---|---|---|---|
| Linux + RT-Smart 双系统 SDK | `k230_sdk`（github.com/kendryte/k230_sdk），产物 `sysimage-sdcard.img` | 大核（CPU1, 1.6GHz）跑 RT-Smart + MPP 多媒体应用；小核（CPU0, 800MHz）跑 Linux(buildroot) + uboot | 两系统并存，靠核间共享内存 IPCM + IPCMSG/sharefs 协作 |
| RT-Smart 单系统（rtt-only） | `k230_sdk`，`k230_canmv_only_rtt_defconfig` | 仅大核 RT-Smart | 无 Linux，小核不跑系统；多媒体中间件有 RTT-only 专用 API |
| CanMV 固件 | `canmv_k230`（github.com/kendryte/canmv_k230），MicroPython 镜像 | 大核跑 RT-Smart + MicroPython VM + MPP Python binding | 本质是 RT-Smart 单系统 + OpenMV 风格 MicroPython；仓库描述即 "micropython openmv rtsmart mpp opensbi uboot" |

（来源：`ref/k230-sdk-user-manual.md`、`ref/canmv-k230-readme.md`、GitHub 仓库元数据）

补充变体：

- K230 SDK 也提供 debian/ubuntu 大核镜像的说明（`ref/k230-debian-ubuntu.md`）——属于 Linux 侧玩法，与 CanMV 无关。
- K230D 是 K230 的 SiP 封装版（片内合封 LPDDR4），庐山派家族的 Lite-K230D 板用它，内存只有 128MB，固件与 K230 板**不可混用**（`ref/lushan-quick-start.html`）。

## 2. 启动链路（各固件共用）

```
BootROM(ROM) ──► usip 参数分区 ──► u-boot ──► opensbi ──► 大核 RT-Smart(+CanMV/MicroPython)
                                    └──────► 小核 Linux（仅双系统镜像；CanMV 镜像无此步负载）
```

- 启动介质支持 SPI NOR/NAND、SD/eMMC；**CanMV-K230 类主板仅支持 SD 卡启动**（`ref/k230-sdk-user-manual.md`）。
- BootROM 启动失败（如无 TF 卡/镜像损坏）会跳入 USB 烧录模式，PC 上表现为 "K230 USB Boot Device"，这正是烧录工具的入口（`ref/k230-burntool-guide.md`）。
- SDK 默认编译"快起镜像"（uboot 不进命令行直接引导系统），可 menuconfig 关 `CONFIG_QUICK_BOOT`（`ref/k230-sdk-user-manual.md`）。

## 3. 我们用 CanMV 的能力边界

### 3.1 能做什么
- MicroPython（OpenMV 风格 API：sensor/image/display）+ 全套 MPP 多媒体 binding（VENC/VDEC/Audio/MP4/RTSP/Display）。
- AI：nncase runtime + KPU + AI2D 硬件前处理；官方 AI Demo 框架（PipeLine/AIBase/Ai2d/YOLO 封装）。
- 通用 IO：FPIOA/GPIO/PWM/UART/I2C/SPI/ADC/RTC/WDT/Timer、network(WLAN/socket)、lvgl、ulab.numpy。
- 固件里内置大量例程与模型，位于 TF 卡 `/sdcard/examples`、`/sdcard/libs`（`ref/lushan-quick-start.html`）。

### 3.2 不能做什么 / 限制
- **跑不了 Linux 程序**：CanMV 镜像里小核没有 Linux 用户空间，无 shell/apt/gcc/pip 官方支持；要跑 C 程序得换 K230 SDK 镜像（Linux+RTT 或 RTT-only），或在 RT-Smart 侧交叉编译 .elf 并经 sharefs 加载（那是 k230_sdk 的开发流，`ref/k230-sdk-user-manual.md`）。
- **裸机（bare-metal）没有官方路线**：CanMV 仍含 opensbi/uboot 引导层，MicroPython 跑在 RT-Smart 内核之上。Wiki 用户贡献区有人做了"基于 Rust 的 K230 裸机嵌入式编程"，属社区实验性质。
- 高性能自定义算法只能走两条路：固件内 C 模块重编固件（改 canmv_k230 源码），或转成 kmodel 让 KPU/NPU 做。
- ISP 目前无 MicroPython 级曝光/白平衡手动 API（AWB/AE 自动）；wiki 明确说"关 AE 成像会变差，暂无 API"（`ref/lushan-use-sensor.html`，官方 17-Sensor 例程里有手动曝光例程 `camera_manual_exposure_lcd.py`，能力以例程为准）。

### 3.3 与"极简相机"的匹配结论
- 取图→JPEG→存卡这条链全部在 CanMV 能力内（见 camera-api.md），**不需要** Linux/双系统 SDK。
- 将来加屏幕预览（ST7701 LCD / LT9611 HDMI / IDE 虚拟屏）和 AI 检测同样是 CanMV 原生支持。

## 4. 固件版本与下载（截至 2026-11）

### 4.1 CanMV 固件（我们用的）
- GitHub Releases：github.com/kendryte/canmv_k230/releases
  - 最新稳定 **v1.8**（2026-07-03），内置 nncase v2.11.0；庐山派 K230 板对应资产
    `CanMV_K230_LCKFB_micropython_v1.8-0-gc2d1f5c_nncase_v2.11.0.img.gz`（约 292MB，.gz 需解压成 .img 再烧）。
  - 同一 release 里还有 K230D Lushanpi_Lite（Lite-K230D 板用）、01Studio、DongshanPI 等其他板资产，别下错。
  - `PreRelease` tag：开发分支每日构建，只留最新。
- 官方社区镜像站：`https://kendryte-download.canaan-creative.com/k230/release/sdk_images/`（`ref/canmv-k230-tutorial.md`）；README 指路嘉楠开发者社区 kendryte.com/resource 的 K230/Images 分类，文件名含 `micropython` 才是 CanMV 镜像。
- v1.7 起新增 `k230d_canmv_lushanpi_lite` 板型；v1.6 开始大量用 RVV 加速（图像处理/HAL）、MPP 私有池重构（`ref/canmv-k230-changelog.md`）。

### 4.2 K230 Linux/RTT SDK 镜像
- 由 k230_sdk 自行编译（docker/Ubuntu 20.04），产物 `sysimage-sdcard.img(.gz)`，可发安全镜像（aes/sm）；版本号 1.0.1→2.0.0，与 nncase 版本一一对应（`ref/k230-nncase-version-map.md`）。
- **micropython 镜像与 K230 SDK 镜像功能集不同，不能拿 SDK 镜像跑 MicroPython**（`ref/canmv-k230-readme.md` 原话强调）。

### 4.3 版本对应关系（速查）
| K230 SDK（Linux 侧） | nncase |
|---|---|
| 1.8.0 | 2.9.0 |
| 2.0.0 | 2.10.0 |
| CanMV 固件 v1.8 | 2.11.0 |

完整表见 `ref/k230-nncase-version-map.md`。

## 5. 内存划分（固件形态决定）

双系统 SDK 默认 menuconfig 布局（总 512MB 档，`ref/k230-sdk-user-manual.md`）：

| 分区 | 基址 | 大小 | 用途 |
|---|---|---|---|
| PARAM | 0x00000000 | 1MB | 启动参数 |
| IPCM | 0x00100000 | 1MB | 核间通讯共享内存 |
| RTT_SYS | 0x00200000 | ~126MB | 大核 RT-Smart |
| LINUX_SYS | 0x08000000 | 128MB | 小核 Linux |
| AI_MODEL | 0x1FC00000 | 4MB | AI 模型加载区 |
| MMZ | 0x10000000 | ~252MB | 多媒体物理连续内存池 |

- CanMV 镜像（RT-Smart 单系统形态）没有 Linux 分区，1GB DDR 全部归大核侧；多媒体 buffer 走 MMZ/VB 池，由 `MediaManager` 统一管理（见 camera-api.md）。庐山派 1GB 版的精确分区数值未在已下载文档中给出（见"未找到"）。
- RTT-only 配置没有 LINUX 内存项；AI_MODEL 基址/大小可 menuconfig 改。

## 6. 对本项目的操作建议

1. 固件直接用 GitHub Release 的 `CanMV_K230_LCKFB_*.img.gz`（解压后 .img），版本 ≥ v1.8。
2. 开发期用 IDE 在线跑；部署期把脚本存成 `/sdcard/main.py`（见 board-lushan.md）。
3. 若后期确需 Linux（如移植现成 Linux 视觉库），再评估 k230_sdk 双系统镜像——相机链路 API 与 CanMV 完全不同（C/MPP），工程量另算。

## 未找到 / 存疑
- 庐山派 1GB CanMV 固件的精确内存分区表（VB 池大小等）：官方 wiki/仓库文档未直接给出，需运行时看日志或读固件 defconfig。
- K230 TRM（Technical Reference Manual）PDF：公开渠道未见完整版，datasheet 以 markdown 版为主（`ref/k230-datasheet.md`）。
- 嘉楠官网 developer.canaan-creative.com / kendryte.com 在本次调研环境不可达（DNS 失败），在线版文档站未能直接抓取，用 GitHub 仓库文档替代。
