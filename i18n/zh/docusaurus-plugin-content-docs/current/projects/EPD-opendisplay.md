---
sidebar_position: 10
---

# NM-EPD-420-4C 与 OpenDisplay（含 Home Assistant 集成）

> 在 NM-EPD-420-4C 四色墨水屏开发板上运行 OpenDisplay 固件，通过 BLE 配置、上传图像，并接入 Home Assistant。

## 概述

[OpenDisplay](https://opendisplay.org/) 是一个以 **BLE（低功耗蓝牙）优先** 的开源电子纸生态系统：手机、电脑或 Home Assistant 主机可以直接连接显示屏，无需专用的 802.15.4 接入点。固件、浏览器工具和 Home Assistant 集成全部开源。

RockBase IoT 已将 NM-EPD-420-4C（ESP32-S3 + 4.2 英寸四色黑/白/红/黄墨水屏，400×300）适配进 OpenDisplay 固件，适配代码位于 [RockBase-iot/OpenDisplay-Firmware](https://github.com/RockBase-iot/OpenDisplay-Firmware) 仓库的 `nm-epd-420` 分支，面板型号为 `EP42YR_400x300`（Good Display 4.2 英寸四色面板系列，GDEY0420F51 / GDEM042F52）。

本文说明如何在 NM-EPD-420-4C 上安装 OpenDisplay 固件、通过 BLE 配置和上传图像，以及如何通过 Home Assistant 集成进行控制。

![OpenDisplay Firmware](/img/projects/epd-420-4c-opendisplay.png)

:::note 当前支持状态
以下功能已在 NM-EPD-420-4C 实机上验证：**全屏刷新、物理按键、BOOT 键深睡眠唤醒、电池电量上报、AHT20温湿度数据上报**。

需要注意的限制：

- 四色面板**不支持局部刷新**，每次全刷约 25–30 秒
- 板载 AHT20 温湿度数据已在固件中采集并广播，但需等待主机端（ [Home Assistant 集成](https://github.com/RockBase-iot/Home_Assistant_Integration) ）增加 AHT20 类型（类型7）支持后，温湿度实体才会在 Home Assistant 中显示
- 红/黄颜色打包顺序、电池电压换算系数等仍在硬件验证中，如遇到红黄色互换或电量显示偏差，请先在仓库提交 issue 确认
:::

## 为什么选择 OpenDisplay

- **BLE 优先的工作流**：无需专用网关即可配置设备和上传图像
- **基于浏览器的工具**：在支持的浏览器（Chrome / Edge 等支持 Web Bluetooth 的浏览器）中即可完成安装、配置和图像上传
- **Home Assistant 官方集成**：设备可被发现并接入 Home Assistant，支持自动化定时刷新画面
- **低功耗设计**：深度睡眠 + 定时/按键唤醒，适合电池供电常亮展示场景

## 准备工作

- NM-EPD-420-4C 开发板（ESP32-S3，16 MB Flash + 8 MB PSRAM, [购买链接：rockbase.shop](https://rockbase.shop/products/nm-epd-420)）
- USB-C 数据线（用于烧录固件）
- 支持 Web Bluetooth 的浏览器（Chrome / Edge，用于 BLE 配置和上传图像）
- 可选：Home Assistant 2026.7.0 或更高版本 + 蓝牙适配器（或 ESPHome Bluetooth Proxy）

## 步骤 1：安装固件

### 方式一：rockbaseiot web flasher(推荐)

直接在[RockBase IoT Web Flasher](https://flash.rockbaseiot.com) 页面选择项目：OpenDisplay，设备选择：NM-EPD-420-4C， 点击链接并下载，即可实现一键烧录。

### 方式二：烧录合并镜像

仓库提供了包含 bootloader、分区表和出厂配置的合并镜像 `bin/NM-EPD-420-4C_factory_full.bin`，使用 esptool 从 `0x0` 地址一次性烧录：

```bash
esptool.py --chip esp32s3 -p COMx -b 460800 write_flash 0x0 NM-EPD-420-4C_factory_full.bin
```

镜像只会在设备**没有已存配置**时写入出厂配置。如果设备上已有旧版配置，需要通过 BLE 写入最新配置（见步骤 2 的命令行方式）。

### 方式三：PlatformIO 源码编译

NM-EPD-420-4C 复用现有的 `esp32-s3-N16R8` 构建环境（没有单独的板级 env，所有板级差异都在运行时配置包中）：

```bash
git clone https://github.com/RockBase-iot/OpenDisplay-Firmware.git
cd OpenDisplay-Firmware
git checkout nm-epd-420
pio run -e esp32-s3-N16R8 -t upload
```

默认构建启用 Wi-Fi（LAN 传输）。如果希望设备同时通过 BLE 和 Wi-Fi 投递图像，需将配置中的 `communication_modes` 设为 `0x5`（BLE | WiFi）。

:::caution
安装 OpenDisplay 固件会替换设备上正在运行的固件（如出厂测试固件）。如需恢复，请参见 [NM-EPD-420 快速上手](../products/nm-epd-420-quickstart.md) 重新烧录出厂固件。
:::

## 步骤 2：通过 BLE 配置设备

### 浏览器方式

1. 打开 OpenDisplay 工具页面（ *官方暂未支持 NM-EPD-420-4C 预设* ）：[opendisplay.org/firmware/config](https://opendisplay.org/firmware/config/)
2. 点击 **Configure over Bluetooth**，在 BLE 配对对话框中选择你的 NM-EPD-420-4C 设备
3. 等待工具将配置写入设备，确认屏幕刷新或显示 OpenDisplay 启动画面

:::tip
NM-EPD-420-4C 的 Toolbox 预设（`nm-epd-420-4c`）已提交合并至 opendisplay.org。如果浏览器工具中暂时找不到该预设，请使用下面的命令行方式写入配置。
:::

### 命令行方式（od-device-cli）

使用仓库自带的 `od-device-cli.py` 工具，通过 BLE 把板级配置写入运行中的设备：

```bash
uv run tools/od-device-cli.py write-config --addr AA:BB:CC:DD:EE:FF \
    --input devices/nm-epd-420-4c.yaml
```

其中 `AA:BB:CC:DD:EE:FF` 为设备的蓝牙 MAC 地址。配置文件 `devices/nm-epd-420-4c.yaml` 已包含完整的面板、引脚、电源和按键配置，一般无需修改。

## 步骤 3：上传图像

1. 打开 OpenDisplay 显示工具：[opendisplay.org/firmware/display](https://opendisplay.org/firmware/display/)
2. 点击 **Connect**，在 BLE 配对对话框中选择设备
3. 点击 **Select Image** 选择本地图像文件
4. 点击 **Upload Image**，传输完成后屏幕会刷新显示图像

:::tip
使用与面板分辨率匹配的 **400×300** 像素图像可以获得最清晰的效果。四色面板每次全刷约 25–30 秒，请耐心等待刷新完成。
:::

## Home Assistant 集成

OpenDisplay 提供官方的 Home Assistant 集成（[OpenDisplay/Home_Assistant_Integration](https://github.com/OpenDisplay/Home_Assistant_Integration)），通过 BLE 与设备通信；当设备启用了 Wi-Fi 且近期通过 mDNS 广播过自身时，会优先走局域网投递图像。

### 前置条件

- Home Assistant **2026.7.0** 或更高版本
- 可正常工作的蓝牙环境，且支持**主动 BLE 连接**：
  - 带蓝牙硬件的 Home Assistant 主机，或
  - 运行 ESPHome 固件的 **Bluetooth Proxy**（`bluetooth_proxy` 需开启 `active: true`）
- 已上电且在蓝牙覆盖范围内的 NM-EPD-420-4C

:::info
Shelly 蓝牙代理只适合被动 BLE 传感器。OpenDisplay 图像上传需要主动 BLE 连接，请使用 Home Assistant 蓝牙适配器或 ESPHome Bluetooth Proxy。
:::

### 安装集成

**方式一：HACS（推荐）**

[![通过 HACS 安装](https://my.home-assistant.io/badges/hacs_repository.svg)](https://my.home-assistant.io/redirect/hacs_repository/?owner=OpenDisplay&repository=Home_Assistant_Integration)

**方式二：手动安装**

1. 从 [最新 Release](https://github.com/OpenDisplay/Home_Assistant_Integration/releases/latest) 下载 `opendisplay` 文件夹；若需要AHT20温湿度传感器，需要使用RockBase IoT源码文件，或复制`custom_components/opendisplay/sensor.py`替换官方的sensor.py文件。
2. 复制到 Home Assistant 的 `custom_components` 目录
3. 重启 Home Assistant

### 添加设备

1. 在 Home Assistant 中进入 **设置 > 设备与服务**
2. 设备进入蓝牙覆盖范围后会被自动发现，点击 OpenDisplay 卡片上的 **添加**；未被自动发现时，点击 **添加集成** 并搜索 **OpenDisplay**
3. 如果设备启用了加密，按提示输入 32 字符密钥
4. 分配设备名称和区域

### 集成提供的能力

| 类型 | 内容 |
| --- | --- |
| 显示内容 | image 实体，显示最近一次发送的画面；对睡眠中的设备显示排队中的画面并标记为待投递 |
| 传感器 | 电池电量与电压、信号强度、最后在线时间；温湿度实体需等待 AHT20 主机端支持（见文首说明） |
| 按键 | USER / BOOT 物理按键的事件实体 |
| 固件 | update 实体，可通过蓝牙 OTA 升级固件 |
| 状态 | 是否有内容等待投递、是否正在使用 Wi-Fi 投递 |

![OpenDisplay HA 集成界面](/img/projects/ha-opendisplay.png)

**电池设备会被妥善处理**：深度睡眠中的设备大部分时间处于离线状态，发送给它的内容会进入队列，在下次唤醒时投递，而不是直接失败。NM-EPD-420-4C 默认 40 秒无活动后进入深睡眠，1800 秒定时唤醒兜底，也可按 BOOT 键立即唤醒。

### 从 Home Assistant 上传图像

#### 方法一：

在homeassistant页面，如`homeassistant:8123`，进入 **设置 > 开发者工具 > Actions > Upload image**，选择设备并上传图像。

![HA上传图像](/img/projects/ha-upload-image.png)

在上传图像动作中还可以根据个人需求对图像进行不同的处理：如:

- Rotation ( 0/90/180/270)
- Dither mode ( None / Burkes/ Floyd-Steinberg / Ordered / Atkinson / Stucki/ Sierra / Sierra Lite / Jarvis, Judice & Ninke / dizzy )
- Refresh mode ( Full / Fast / Partial )
- Fit mode ( Stretch / Contain / Cover / Crop )

#### 方法二：

把图像放入 Home Assistant 的本地媒体文件夹，然后调用 `opendisplay.upload_image` 动作：

```yaml
action: opendisplay.upload_image
data:
  device_id: "your_device_id"
  image:
    media_content_id: "media-source://media_source/local/photo.png"
    media_content_type: "image/png"
```

结合定时触发器即可实现每日定时更新画面：

```yaml
triggers:
  - trigger: time
    at: "08:00:00"
actions:
  - action: opendisplay.upload_image
    data:
      device_id: "your_device_id"
      image:
        media_content_id: "media-source://media_source/local/daily.png"
        media_content_type: "image/png"
```

### 自定义绘制（drawcustom）

`drawcustom` 动作可以直接用文本、图形、图标、二维码、图片、图表和进度条组合成一帧画面，且支持模板渲染实时状态：

```yaml
action: opendisplay.drawcustom
target:
  device_id: "your_device_id"
data:
  payload:
    - type: text
      value: "温度: {{ states('sensor.temperature') }}°C"
      x: 10
      y: 10
      size: 24
    - type: progress_bar
      x_start: 10
      y_start: 50
      x_end: 180
      y_end: 70
      progress: "{{ states('sensor.battery') | int }}"
      show_percentage: true
    - type: icon
      value: mdi:battery-70
      x: 190
      y: 60
      size: 24
```

`drawcustom` 支持 **target**，一次调用可以绘制到多台设备、整个区域或标签。全部元素类型与字段见集成仓库的 [drawcustom 文档](https://github.com/OpenDisplay/Home_Assistant_Integration/blob/main/docs/drawcustom/supported_types.md)。

## 设备配置参考

NM-EPD-420-4C 的 OpenDisplay 板级配置（`devices/nm-epd-420-4c.yaml`）要点如下，供进阶用户参考：

| 项目 | 配置 |
| --- | --- |
| 构建环境 | `esp32-s3-N16R8`（ESP32-S3，16 MB QIO Flash + 8 MB OPI PSRAM） |
| 面板 | `EP42YR_400x300`（`panel_ic_type: 58`），400×300，四色 B/W/R/Y（2 bpp），不支持局刷 |
| EPD 引脚 | SCK=IO2，MOSI=IO1，CS=IO46，DC=IO4，RST=IO5，BUSY=IO6 |
| 按键 | USER=IO45（低电平有效），BOOT=IO0（低电平有效，兼作深睡眠唤醒源） |
| 电池 | ADC=IO3（1:2 分压，使能脚 IO43），600 mAh，40 秒空闲后深睡眠，1800 秒定时唤醒 |
| 传感器 | AHT20 温湿度（I²C：SCL=IO38，SDA=IO39，地址 0x38），唤醒时采样一次后断电 |
| 通信 | 默认 BLE（`communication_modes: 0x1`），可改为 `0x5` 启用 BLE + Wi-Fi |

:::note
NM-EPD-420 系列三种面板变体（三色 / 黑白 / 四色）共用同一块 PCB 和引脚定义，OpenDisplay 配置中只有 `panel_ic_type`、`color_scheme` 和全刷参数不同。本页的适配目前**仅针对四色版 NM-EPD-420-4C**。
:::

## 故障排查

- **红/黄颜色显示互换**：四色面板的 2 bpp 颜色打包顺序仍在硬件验证中，请先到仓库确认最新状态
- **每次刷新都超时**：优先怀疑 BUSY 极性配置，请在仓库提交 issue 并附上串口日志
- **BOOT 键相关异常**：IO0 同时是 ESP32-S3 的 ROM 下载引导脚，复位或上电前请确保 BOOT 键处于松开状态（除非有意进入下载模式）
- **Home Assistant 中没有温湿度实体**：属已知限制，等待 HA 集成增加 AHT20（SensorType 7）支持，或使用RockBase-iot 源码文件替换官方的 `sensor.py`。

## 相关资源

- [RockBase-iot/OpenDisplay-Firmware](https://github.com/RockBase-iot/OpenDisplay-Firmware)（`nm-epd-420` 分支，含 NM-EPD-420-4C 适配）
- [OpenDisplay 官网与浏览器工具](https://opendisplay.org/)
- [OpenDisplay Home Assistant 集成](https://github.com/OpenDisplay/Home_Assistant_Integration)
- [OpenDisplay Discord 社区](https://discord.gg/wgQ8XsgMkv)
- [NM-EPD-420 产品页](../products/nm-epd-420.md) | [NM-EPD-420 移植生态](./nm-epd-420-ecosystem.md)
