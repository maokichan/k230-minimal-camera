# 相机链路 API（CanMV/MicroPython 层）（camera-api）

> 来源：`ref/lushan-api-sensor.html`、`ref/lushan-api-image.html`、`ref/lushan-api-venc.html`、`ref/lushan-api-media.html`、`ref/lushan-use-sensor.html`、`ref/canmv-examples/`（canmv_k230 官方例程）。API 以嘉楠官方公布的为准（V0.7 固件后大改，本文按 ≥v0.7 现行 API）。

## 1. 链路架构

```
GC2093(CSI2) ─► Sensor 模块(每路3个输出通道 chn0/1/2) ─► snapshot() 取 image 对象
                     │ bind_layer 自动 link            │
                     ▼                                 ▼
              Display(VO: LCD/HDMI/IDE虚拟屏)    VENC(image.compress 或 Encoder)
                                                      │
                                                      ▼
                                              open(path,'wb') 写 TF 卡
```

- 最多 3 路传感器（CSI0/1/2）同时接入；每路 sensor 最多 3 个输出通道并行出图（庐山派默认摄像头接 **CSI2**，即 `Sensor(id=2)`，id=2 也是构造默认值）。
- 所有媒体 buffer 由 **MediaManager** 统一管理，先各模块配置、再 `MediaManager.init()`、最后才能 `sensor.run()`。

## 2. 初始化/退出顺序（写错就报错或漏帧）

```
构造 Sensor → sensor.reset() → set_framesize/set_pixformat →
Display.init(...)（bind_layer 需在此之前）→ MediaManager.init() →
sensor.run() → 循环 snapshot →
sensor.stop() → Display.deinit() → MediaManager.deinit()
```

- `sensor.run()` 必须在 `MediaManager.init()` 之后；`sensor.stop()` 必须在 `MediaManager.deinit()` 之前。
- 多传感器场景：只需其中一个调用 `run()`；退出时每个 sensor 都要 `stop()`。
- 主循环里放 `os.exitpoint()`，让 Ctrl-C/断点能安全退出（官方例程写法）。

## 3. Sensor 模块 API 速查

| 方法 | 说明 |
|---|---|
| `Sensor(id=2, [w,h,fps])` | 构造；默认 1920x1080@30fps。无需指定型号，固件自动识别（日志如 `use sensor 23, output 640x480@90`） |
| `sensor.reset()` | 构造后必须先调用 |
| `sensor.set_framesize(width=, height=, chn=CAM_CHN_ID_0)` | 设通道输出尺寸；宽度自动 16 对齐；也接受 `framesize=Sensor.FHD` 等常量（约 30 档：QQVGA…VGA…HD…FHD…UXGA） |
| `sensor.set_pixformat(Sensor.YUV420SP / RGB888 / RGBP888 / RGB565 / GRAYSCALE, chn=)` | 设通道像素格式 |
| `sensor.set_hmirror(b)` / `sensor.set_vflip(b)` | 水平镜像/垂直翻转 |
| `sensor.run()` / `sensor.stop()` | 启停采集（顺序规则见上） |
| `sensor.snapshot(chn=CAM_CHN_ID_0)` | 取一帧，返回 image 对象 |
| `sensor.bind_info()` | 给 `Display.bind_layer(**bind_info)` 用，实现零拷贝直通显示 |
| `sensor.auto_focus(1)` / `sensor.focus_pos()` | 仅自动对焦版摄像头（内置音圈马达） |

格式取舍：显示用 RGB565（省内存）或 YUV420SP（可直通 VO）；给 AI 的通道用 RGBP888（planar，AI Demo 默认）；编码走 YUV420SP。K230 内置 ISP，AWB/AE 自动，MicroPython 层暂无完整曝光/白平衡调节 API（官方例程 `camera_manual_exposure_lcd.py` 提供手动曝光）。

## 4. 最小拍照流程（要点 + 代码）

### 4.1 骨架（按官方 `camera_snapshot_and_save.py` 改写成"拍一张存 JPEG"）

```python
import os, time, image
from media.sensor import *
from media.display import *
from media.media import *

sensor = Sensor(id=2)                      # 庐山派默认摄像头在 CSI2
sensor.reset()
sensor.set_framesize(width=1920, height=1080, chn=CAM_CHN_ID_0)
sensor.set_pixformat(Sensor.YUV420SP, chn=CAM_CHN_ID_0)   # 编码友好格式
# 纯存卡可完全不初始化 Display；调试时可加：
# Display.init(Display.VIRT, width=1920, height=1080, to_ide=True)

MediaManager.init()
sensor.run()
for _ in range(30):                        # 丢前 30 帧，等 AE/AWB 稳定
    sensor.snapshot()

img = sensor.snapshot(chn=CAM_CHN_ID_0)
img.save("/data/photo_001.jpg", quality=90)   # 一步 JPEG 落盘

sensor.stop()
# Display.deinit()
MediaManager.deinit()
```

### 4.2 存图的三种写法（Image 模块）
| 写法 | 说明 |
|---|---|
| `img.save(path, roi=None, quality=50)` | 直接存盘；按后缀选编码（jpg/jpeg；也支持 bmp/pgm/ppm） |
| `data = img.compress(quality=95)` | 返回 JPEG 压缩后的 image 对象；可再 `open(fn,'wb').write(data)` 或复用 |
| `img.compressed(quality=50)` | 同上但返回新图像对象（命名差异注意） |

- `image.Image(path)` 可从 TF 卡读入 BMP/PGM/PPM/JPG 生成图像对象。
- 常用读信息方法：`img.width()/img.height()/img.format()`；转换：`img.to_rgb565(copy=True)`、`img.to_grayscale()`。
- 绘制 OSD：`img.draw_string()/draw_line()/draw_rectangle()/draw_circle()` 等全套 OpenMV 风格 API（`ref/lushan-api-image.html`）。
- IDE 预览专用：`img.compress_for_ide()`（或 Display `to_ide=True` 自动传帧缓冲）。

### 4.3 wiki 按键拍照例程的工程要点（`lushan-use-sensor`）
- 路径管理：存 `/sdcard/images/`，目录不存在先 `os.mkdir`；扫描已有文件名最大编号 +1，避免覆盖。
- 按键：`Pin(53, Pin.IN, Pin.PULL_DOWN)`，200ms 消抖；RGB LED（引脚 62/20/63，共阳，高电平灭）做快门指示。
- JPEG 质量 quality=95；文件名带分辨率（如 `lckfb_00465_800x480.jpg`）。
- 完整代码在 wiki 页（`ref/lushan-use-sensor.html`）。

## 5. JPEG/视频编码：两条路怎么选

| 需求 | 用什么 | 说明 |
|---|---|---|
| 静态 JPEG 拍照 | `img.save()` / `img.compress(quality)` | 简单直接；底层 OpenMV imlib JPEG 编码器，够快 |
| 连续 JPEG/MJPEG 流 | 硬件 VENC MJPEG（官方例程 `video_encoder.py`/`mjpeg_web_server.py`） | h264/h265 之外视频场景 |
| MP4 录制（H265+G711U 音频） | `media.mp4format.Mp4Container`（wiki 录放视频页） | muxer 写 `/data/test.mp4`；播放当前有倍速 bug，录制正常 |
| H.264/H.265 裸流 | `media.encoder.Encoder` 类 | **Python API 只暴露 H264/H265 payload**；最多 4 路，第 4 路固定给 IDE 传输，建议只用 [0,2] |

VENC Encoder 用法（`ref/lushan-api-venc.html`）：
```python
from media.encoder import *
Encoder.SetOutBufs(chn, buf_num, w, h)      # 必须在 MediaManager.init() 前
chnAttr = ChnAttrStr(encoder.PAYLOAD_TYPE_H264, encoder.H264_PROFILE_MAIN, w, h, gopLen=30)
Encoder.Create(chn, chnAttr); Encoder.Start(chn)
Encoder.SendFrame(chn, frame)               # frame 来自 sensor/vdec
stream = StreamData(); Encoder.GetStream(chn, stream)
... 写文件/推流 ...
Encoder.Stop(chn); Encoder.Destroy(chn)
```
硬件能力上限（datasheet）：编码 8MP@20fps；JPEG 编码仅 baseline YUV420/YUV422；H.264 ≤4096×4096，无 B 帧（CAVLC 时）。

## 6. Display（预览用）

```python
Display.init(type, width, height, osd_num=1, to_ide=False, fps=None, quality=90)
Display.show_image(img, x=0, y=0, layer=None, alpha=255, flag=0)
Display.bind_layer(src=(mod,dev,chn), dstlayer, rect, pix_format, alpha, flag)  # 需在 init 前
Display.deinit()
```
| type | 默认分辨率 | 备注 |
|---|---|---|
| `Display.VIRT` | 640x480@90 | IDE 帧缓冲虚拟屏，可自定 (64x64)~(4096x4096)、1-200fps；纯调试用 |
| `Display.ST7701` | 800x480@30 | 立创 3.1 寸屏扩展板（可竖屏 480x800） |
| `Display.LT9611` | 1920x1080@30 | MIPI 转 HDMI 扩展板（也支持 720p/480p） |

- 图层：2 个 VIDEO 层（仅 bind_layer，零拷贝直通）+ 4 个 OSD 层（show_image/bind_layer 均可）；`osd_num` 越大越吃内存。
- `to_ide=True` 会把显示帧压缩传给 IDE（多耗内存，quality 10-100）。
- 常见坑：脚本默认 HDMI 但板子接的是 LCD → `RuntimeError: init panel failed`（wiki 快速入门提醒）。

## 7. 文件保存到 TF 卡

| 路径 | 是什么 | 用途 |
|---|---|---|
| `/sdcard` | TF 卡用户分区，PC 上"CanMV U 盘"可见 | `main.py`/`boot.py`、自建图片目录、例程、kmodel |
| `/data` | TF 卡除固件外剩余空间格式化成的 FAT 大分区 | 大文件（照片/视频）建议存这里 |
| `/udisk` | 外插 U 盘挂载点 | 可选扩容 |

- 用 `uos` 标准函数：`os.listdir(path)`、`os.mkdir(path)`、`os.remove()`；判断存在可 `try: os.listdir` 捕获异常。
- 官方例程两种习惯都存在：`camera_snapshot_and_save.py` 存 `/data/`，wiki 按键例程存 `/sdcard/images/`。README（`ref/canmv-examples/camera_snapshot_and_save.py`）原始版存的是 raw YUV/RGB（用 7yuv 预览），JPEG 需自行 `img.save(...jpg)`。

## 8. 最小骨架（带预览的完整顺序，供 src/ 参考）

```python
import os, time, image
from media.sensor import *
from media.display import *
from media.media import *

try:
    sensor = Sensor(id=2)
    sensor.reset()
    sensor.set_framesize(Sensor.FHD, chn=CAM_CHN_ID_0)          # 1920x1080
    sensor.set_pixformat(Sensor.YUV420SP, chn=CAM_CHN_ID_0)
    bind_info = sensor.bind_info()
    Display.bind_layer(**bind_info, layer=Display.LAYER_VIDEO1)  # 直通预览（可选）
    Display.init(Display.VIRT, width=1920, height=1080, to_ide=True)
    MediaManager.init()
    sensor.run()
    n = 0
    while True:
        os.exitpoint()
        img = sensor.snapshot(chn=CAM_CHN_ID_0)
        if n == 30:                                              # 等 AE 稳定后拍
            img.save("/data/shot.jpg", quality=90)
        n += 1
except KeyboardInterrupt:
    pass
finally:
    sensor.stop()
    Display.deinit()
    os.exitpoint(os.EXITPOINT_ENABLE_SLEEP)
    time.sleep_ms(100)
    MediaManager.deinit()
```

## 未找到 / 存疑
- `img.compress` 的底层实现（imlib 软编还是走硬件 JPEG）官方文档未写明；官方拍照例程只用 image.save/compress，VENC 只给 H264/H265。
- `camera_snapshot_and_save.py` 里 `image.YUV420` 常量在当前固件的模块名（脚本未显式 import image，最新固件可能已修复）——照抄旧例程时注意。
- 单张拍照的实测耗时（AE 稳定 + JPEG 编码）未查到官方数字，需实测。
