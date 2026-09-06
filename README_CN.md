# e87_badge 中文文档

开源 Python 客户端 + Home Assistant 集成，用于控制 **E-Badge E87 / L8** 圆屏蓝牙电子徽章（通常与 Zrun 应用配对）。

## 功能特性

- 🖼️ **静态图片**：支持 JPEG/PNG 图片上传
- 📝 **文字渲染**：屏幕上实时渲染文字
- 🎞️ **多图轮播**：支持 MJPG AVI 格式的多张图片轮播
- 🖼️ **动画 GIF**：支持 GIF 动画播放
- 🧧 **弹幕滚字**：自定义颜色的滚动文本效果

## 技术基础

- 🔐 逆向工程破解了 **JieLi RCSP** 通讯帧格式和相互认证密码
- 📡 基于上游项目 [hybridherbst/web-bluetooth-e87](https://github.com/hybridherbst/web-bluetooth-e87) (MIT)
- 🏠 通过 Home Assistant 的 `habluetooth` 和 ESPHome `bluetooth_proxy` 实现整屋通信

详细的协议文档请参考 [`docs/protocol.md`](docs/protocol.md)。

---

## 安装 - Python 库 + CLI 工具

### 安装

```bash
pip install git+https://github.com/jumpingmushroom/e87_badge@v0.1.14
```

### 命令行使用

```bash
e87 discover                                 # 扫描附近的徽章
e87 image my-photo.png                       # 上传静态图片
e87 text "Hello" --size 96 --colour white    # 渲染文字
e87 slideshow a.png b.png c.png --ms 600     # 多图轮播（600ms 间隔）
e87 gif pulse.gif                            # 播放 GIF 动画
e87 danmaku "Welcome!" --fg red --bg yellow  # 弹幕滚字
```

**指定特定徽章：** 在命令后加 `--address AA:BB:CC:DD:EE:FF`（不指定时自动选择发现的第一个）

### Python 库 API

```python
import asyncio
from e87_badge import E87Client

async def main():
    async with E87Client("46:8D:00:01:2C:25") as badge:
        await badge.send_image("welcome.png")
        await badge.send_text("Hi")
        await badge.send_slideshow(["a.png", "b.png", "c.png"], frame_ms=500)
        await badge.send_gif("party.gif")
        await badge.send_danmaku("breaking news!", fg="red", bg="black")

asyncio.run(main())
```

`E87Client` 接受 MAC 地址字符串或 `bleak.BLEDevice` 对象。后者是 Home Assistant 通过最近蓝牙代理路由通信的方式。

---

## 安装 - Home Assistant 集成

自定义组件位于 `custom_components/e87_badge/`，通过 `manifest.json` 自动安装依赖库。

### HACS 方式（推荐）

1. HACS → 集成 → ⋮ → 自定义代码库
2. 添加 `https://github.com/jumpingmushroom/e87_badge` 为**集成**
3. 安装 "E87 Smart Digital Badge"，重启 Home Assistant

### 手动安装

```bash
cd /config
git clone --branch v0.1.14 https://github.com/jumpingmushroom/e87_badge
ln -s e87_badge/custom_components/e87_badge custom_components/e87_badge
```

重启 HA。当徽章广播时，会在 **设置 → 设备与服务 → 已发现** 中显示 "E87 Smart Digital Badge"。添加后获得：

- **1 个传感器实体**：显示连接状态
  - 属性：`last_sent_at`、`last_sent_type`、`rssi`、`proxy_source`
  
- **5 个服务**：

| 服务 | 字段说明 |
|---|---|
| `e87_badge.send_image` | `image`：本地路径、URL 或 base64 |
| `e87_badge.send_text` | `text`：文字内容；可选 `font`、`size`、`colour`、`bg` |
| `e87_badge.send_slideshow` | `images`：图片列表；可选 `frame_ms` 帧间隔 |
| `e87_badge.send_gif` | `image`：GIF 路径/URL/base64；可选 `max_fps` |
| `e87_badge.send_danmaku` | `text`：滚字内容；可选 `fg` 前景色、`bg` 背景色、`font`、`font_size`、`speed`、`fps` |

### 自动化示例

```yaml
- alias: 回家时显示欢迎图片
  trigger:
    - platform: state
      entity_id: person.johnny
      to: "home"
  action:
    - service: e87_badge.send_image
      target:
        entity_id: sensor.e87_badge_status
      data:
        image: /config/www/badges/welcome.png
```

### Home Assistant 集成注意事项

**⚠️ 增加最近的 ESPHome 蓝牙代理的连接槽**

徽章上传协议使用一个长连接 + 多个短命令往返，需要较多连接资源。

**推荐配置 - Bluedroid（ESPHome 默认）：**

```yaml
bluetooth_proxy:
  active: true
  connection_slots: 4

esp32_ble_tracker:
  scan_parameters:
    active: true   # 识别 "E87" local_name 所需
```

- 默认 Bluedroid 构建下，4 个槽位可正常工作
- 尝试 `5+` 槽位会导致启动失败：`Failed due to no resources. Try to reduce number of BLE clients in config.`

**⚠️ 不要尝试切换到 NimBLE** — ESPHome 的 `esp32_ble_tracker` 和 `bluetooth_proxy` 组件依赖 Bluedroid 头文件，无法编译 NimBLE 版本。

**其他建议：**

- 发送失败时，**重启一次代理**以清除残留状态
- **分散部署多个代理**，让每个代理距离徽章 ~3 米以内
- **上传时间预期**：
  - 小静态图片：5-15 秒
  - 轮播/GIF/弹幕：30-60 秒
  - 服务调用不会超时，但自动化中要考虑这些时间
- v0.1.12+ 传输中断时会自动发送 `CMD_STOP` 清空状态，旧版本需等待几分钟超时

---

## 系统要求

- Linux 或 Home Assistant OS 主机（需蓝牙支持）或 ESPHome `bluetooth_proxy`
- Python 3.11+（HA 本身要求 3.12+）
- 依赖库（自动安装）：`bleak`、`bleak-retry-connector`、`pillow`

## 许可证

MIT License。详见 [`LICENSE`](LICENSE)。

JieLi 认证密码表和 AVI 构建器移植自 [web-bluetooth-e87](https://github.com/hybridherbst/web-bluetooth-e87)（© 2026 Felix Herbst, MIT）。

---

## 相关资源

- 📡 **原始项目**：https://github.com/jumpingmushroom/e87_badge
- 🔍 **Zrun 应用** - 官方控制应用
- 📚 **协议详解**：查看 `docs/protocol.md`
- 🎯 **逆向工程数据**：查看 `docs/captures/` 目录
