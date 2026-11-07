# 庐山派专属事项（board-lushan）

> 来源：`ref/lushan-*.html`（立创 wiki 原页存档）、`ref/canmv-examples/`、GitHub release 元数据。板卡硬件全开源：https://oshwhub.com/li-chuang-kai-fa-ban/li-chuang-lu-shan-pai-k230-canmv-kai-fa-ban

## 1. 板子概况

- 型号：立创·庐山派 K230-CanMV 开发板（K230 芯片，1GB 内存版）；姊妹板 **Lite-K230D** 用 K230D（SiP 内置 128MB LPDDR4），两者固件**不可混烧**，烧错可能无法启动甚至硬件异常（`ref/lushan-quick-start.html` DANGER 框）。
- 设计定位：电赛/极简视觉方案，"如无必要勿增实体"；标准版只留 CanMV 必需硬件（HDMI 经扩展板、WiFi、网口可选）。
- 供电三路（`ref/lushan-board-profile.html`）：USB Type-C 5V/2A（带过流/反接/防倒灌）；排针或 GH1.25-4P 双向 5V；GH1.25-2P 8-24V（DCDC 降压，可接 3S 锂电）。
- 正常上电状态：PWR 红灯常亮；用户 RGB 灯亮一下 2 秒内熄灭；否则按 FAQ 排查（灯微亮/不灭 = 没插卡进 bootloader）。
- 选型对比页 k230vsk230d（JS 渲染）未能抓到正文，K230/K230D 差异以 datasheet（SiP DDR）和 quick-start 为准。

## 2. 固件烧录 TF 卡（两种官方方式）

### 2.1 K230BurningTool（Windows，首推）
1. 下载固件：立创文档指向嘉楠开发者社区 Daily Build/稳定版，文件名须带 **LCKFB** 且为 **.img**（.gz 是压缩包要先解压，直接烧 .gz 会变砖）；GitHub release 资产 `CanMV_K230_LCKFB_micropython_v1.8-0-gc2d1f5c_nncase_v2.11.0.img.gz` 同样可用。
2. 进 BOOT 模式：按住 BOOT 键 → 短按 RESET → 设备管理器出现 "K230 USB Boot Device" 再松开。
3. 驱动：带感叹号时用 **Zadig** 装 WinUSB（Options → List All Devices → K230 USB Boot Device → WinUSB → Install）。
4. `K230BurningTool.exe`：选 .img，目标介质选 **SD Card**，开始，等 100%。
（另：嘉楠官方文档也支持 Linux `dd` 与 Rufus 烧 .img，`ref/k230-sdk-user-manual.md`；BurnTool 原理见 `ref/k230-burntool-guide.md`——USB 烧录实现在 BootROM 里）

### 2.2 CanMV IDE Web 网页烧录
- Chrome/Edge（Web Serial API），HTTPS 访问 wiki.lckfb.com 的 CanMV Web IDE。
- 板型选 `CanMV_K230_LCKFB_*.img`（Lite-K230D 选 `CanMV_K230D_Lushanpi_Lite_*.img`）。
- 两阶段烧录中途板子会重启断开，需要重新授权 USB 设备；完成后自动重启。

### 2.3 首次启动
- 自动把 TF 卡剩余空间格式化 FAT 挂到 `/data`，期间自动重启一次，完成后仍需等 10~30 秒别断电。
- 成功标志：指示灯点亮 + 设备管理器出现 "USB 串行设备（COMx）" + 一个 CanMV U 盘设备。
- FAQ 排查要点（`ref/lushan-faq.html`）：TF 卡要正规渠道（不兼容/速度不足是头号启动失败原因）；端口号冲突时到端口属性-高级改号；用户灯异常对应固件/卡问题。

## 3. IDE 与串口

| 工具 | 特点 |
|---|---|
| **VS Code + CanMV 扩展**（长期开发首推） | 扩展市场搜 "CanMV"；VS Code ≥1.90；命令面板 "CanMV: 连接开发板"；自动检测失败时设置 `canmv.serialPath` |
| CanMV IDE Web | 浏览器 Web Serial 直连，F5 运行；适合快速体验 |
| 传统串口终端 | USB CDC 虚拟串口，Win10/11 免驱；USB VID/PID **1209:abd1**；115200 调试口（排障时接串口0） |

- 同一时间只能一个程序占用串口；切换工具前先断开。
- 运行 .py = 解释执行（不是烧录）；停止用 Ctrl-C/停止按钮。
- IDE 版本口径：FAQ 提到 IDE 4.0.x（CanMV IDE K230，github.com/kendryte/canmv_ide/tree/k230，QtCreator 系 OpenMV IDE 衍生）。

## 4. 离线运行（脱离电脑自启动）

- 上电后 MicroPython 依次自动执行 `/sdcard/boot.py`（配置类，**不能有死循环**，否则阻塞 main.py）→ `/sdcard/main.py`（主逻辑）。
- IDE 保存法：菜单 工具 → "save open script to CanMV board (as main.py)"，会提示去注释/空格转 tab。
- 手动法：板子当 U 盘，把文件改名 main.py 复制到 sdcard 根目录。
- 极简相机的部署形态即：`main.py` 里跑拍照主循环。

## 5. 板载外设清单

| 外设 | 细节 | 对项目的影响 |
|---|---|---|
| **摄像头接口** | 3 路 22pin-0.5mm FPC：CSI2（正面立式，默认座）、CSI0/CSI1（背面卧式翻盖） | 默认 `Sensor(id=2)`；支持三摄同开 |
| **GC2093 摄像头**（标配） | 1920x1080@60fps；三个版本：定焦 70cm 小镜头（4.3mm F2.0）、M12 大镜头可旋转调焦（3.0mm F2.5 广角）、自动对焦版（音圈马达，`sensor.auto_focus(1)`） | 兼容树莓派 Zero/5 摄像头但需软件适配，官方推荐定制 GC2093 |
| **WiFi** | 板载模块内部芯片 **RTL8189FTV**，**仅 2.4GHz**（不支持 5G/双频合一路由器），板载陶瓷天线，可改 0Ω 电阻切 IPEX 外置天线 | `network.WLAN(network.STA_IF)`，STA/AP 两模式；上传照片可用 |
| **显示** | 1 路 MIPI DSI：立创 3.1 寸屏扩展板（ST7701, 800x480）、MIPI 转 HDMI 板（LT9611, 1080p）；另有 HDMI 扩展板 | 预览/OSD 见 camera-api.md |
| **按键/LED** | 用户按键 GPIO53（下拉输入），RGB LED 62/20/63（共阳） | 快门/指示灯直接可用 |
| **麦克风/耳机** | 板载贴片麦克风 + 3.5mm 耳机口（录 MP4 自带音轨） | 后期加录音 |
| **串口** | Type-C USB CDC（虚拟串口） | 调试日志 |

## 6. 庐山派固件 vs 嘉楠官方 canmv_k230 的关系

- 庐山派固件 = canmv_k230 仓库里 `port/boards/k230_canmv_lckfb/` 板级配置（`mpconfigboard.h` + `manifest.py`，见 `ref/canmv-examples/`）编译出的产物，release 资产名 `CanMV_K230_LCKFB_*`。
- 与 01Studio/DongshanPI/YAHBOOM 等资产同源同版本，只差板级外设配置（屏幕驱动、摄像头 ID、引脚复用等）。API/例程完全一致。
- 嘉楠官方 canmv_k230 的发行渠道（GitHub Release + kendryte 社区镜像站）现在就直接覆盖庐山派；wiki 固件页只是指路（`ref/lushan-firmware.html` 为 stub，指向 canmv_k230 仓库）。
- 例程与库随固件预装在 TF 卡：`/sdcard/examples`（01-Micropython-Basics…05-AI-Demo…17-Sensor…20-YOLO-Module-Examples 等）、`/sdcard/libs`（PipeLine/AIBase/AI2D/YOLO/Utils）。

## 7. 常见坑速查（FAQ + 各例程注释提炼）

1. 启动失败先怀疑 TF 卡（兼容性/假货/速度），再怀疑固件版本（要用最新 .img/LCKFB）。
2. "K230 USB Boot Device" 出现在设备管理器 = 系统没起来进了 bootloader，不是新串口。
3. IDE 连不上但 U 盘正常 = 串口号冲突，改端口号。
4. 默认摄像头排线装好后是歪的（CSI2 座子位置所限），塞进座子与亚克力面板之间即可。
5. 示例默认 HDMI 显示时接了 3.1 寸屏会 `RuntimeError: init panel failed`，改 `DISPLAY_MODE`。
6. WiFi 连不上：路由器必须分离出 2.4G 信号，手机热点强制 2.4G。
7. MP4 播放当前比正常速度快（固件 bug，录制不受影响）。

## 未找到 / 存疑
- k230vsk230d 选型表正文（JS 渲染）未抓到。
- 庐山派原理图 PDF（oshwhub 链接）未下载——硬件细节如需精确引脚/电源树请看开源仓库；wiki 引脚查询工具为在线交互页。
- 庐山派出厂预装固件版本（可能与最新 GitHub release 不同）：wiki 未标注，建议直接烧 v1.8。
