# 标题：修复 ESP32-S3 因 CH343 串口芯片配置不匹配导致的上传失败
# 涉及层次：环境、配置

## 1. 背景

- 目标：把麦克风测试代码烧录到 ESP32-S3 开发板，验证 INMP441 能正常采集音频。
- 遇到的具体问题：
  1. 上传时报 `Could not open COM3, the port is busy or doesn't exist`，串口被占用或拒绝访问。
  2. 之后演变为 `A serial exception error occurred: Write timeout`，esptool 写入数据时超时。
  3. 串口监视器输出不稳定，有时两行有时三行，按 RESET 键无反应。
  4. 手动按住 BOOT 进下载模式，依然无法上传。
- 为什么这个问题值得记录：这次排查横跨了“环境”和“配置”两个层次，最终定位到根因是**板子用的是 CH343 外部串口芯片，而不是 ESP32-S3 原生 USB**。这个信息在板子外观上看不出来，必须通过 VID/PID 才能识别。记录下这条经验，以后遇到同类问题可以直接从“查串口芯片型号”入手。

## 2. 排查过程

- 现象：
  - `PermissionError(13, '拒绝访问')` 和 `Serial port busy`，COM3 被占用。
  - `Write timeout`，esptool 无法与芯片握手。
  - 电源灯一直亮，说明供电稳定，不是断电问题。
  - 串口输出不稳定，按 RESET 无反应。
- 尝试过的方法（包括失败的）：
  1. 关闭串口监视器、重启 Arduino IDE，无效。
  2. 用 PowerShell 强制结束所有 Arduino 和 Python 进程，无效。
  3. 换 USB 线、换 USB 口，无效。
  4. 手动按住 BOOT + RESET 进下载模式，仍然报 `Write timeout`。
  5. 检查开发板管理器，发现装的是 `Arduino ESP32 Boards 2.0.18`（Arduino 官方精简版），不含通用 ESP32-S3 板定义。
  6. 确认已安装 `esp32 by Espressif Systems 3.3.10-cn`，选对了 `ESP32S3 Dev Module`。
- 关键线索：
  1. 开发板信息显示 **VID: 0x1A86，PID: 0x55D3**。`0x1A86` 是 WCH（沁恒）的厂商 ID，`0x55D3` 对应 **CH343** 串口芯片。
  2. 这说明板子**不是**用 ESP32-S3 原生的 USB-Serial/JTAG，而是外挂了一颗 CH343 芯片做 USB 转串口。
  3. 之前一直把 `USB CDC On Boot` 设成 `Enabled`，那是针对原生 USB 的，和 CH343 不匹配。
- 根本原因：
  1. 板子用的是 CH343 外部串口芯片，但 `USB CDC On Boot` 被设成 `Enabled`，导致 esptool 走错了通信通道，握手失败。
  2. CH343 驱动可能没装好或状态异常，导致串口通信不稳定、`Write timeout`。

## 3. 解决方案

- 具体步骤：
  1. 下载并安装 **CH343 驱动**（WCH 官网：`https://www.wch.cn/downloads/CH343SER_EXE.html`）。
  2. 安装后**重启电脑**，让驱动生效。
  3. 打开 Arduino IDE，**工具 → 开发板**，选 **ESP32S3 Dev Module**。
  4. 在工具菜单里把 **USB CDC On Boot** 改成 **Disabled**（因为用的是 CH343 外部串口，不是原生 USB）。
  5. **Flash Size** 设为 **16MB**，**PSRAM** 设为 **OPI PSRAM**。
  6. **Upload Speed** 先设为 **115200**，稳定后再调高。
  7. 点上传。
- 关键命令/代码片段：

查看开发板 USB 信息（Arduino IDE 里点“获得开发板信息”）：

```
BN: 未知开发板
VID: 0x1A86
PID: 0x55D3
SN: 5C93084535
```

VID/PID 对照：
- `0x1A86` = WCH（沁恒）
- `0x55D3` = CH343

- 需要注意的坑：
  1. **`USB CDC On Boot` 的设置取决于板子用的是原生 USB 还是外部串口芯片**。原生 USB 设 `Enabled`，CH343 这类外部芯片设 `Disabled`。
  2. **开发板管理器里可能同时装了多个 ESP32 包**。`Arduino ESP32 Boards`（Arduino 官方版）只含少数板子，`esp32 by Espressif Systems` 才是完整包，含 `ESP32S3 Dev Module`。要用后者。
  3. **CH343 驱动必须装**，否则串口通信不稳定，表现为 `Write timeout` 或输出断断续续。
  4. **不要只看板子外观判断串口芯片**，必须通过 VID/PID 确认。很多第三方 ESP32-S3 板子都用 CH343，而不是原生 USB。
- **转折：从“端口占用”到“芯片不匹配”
  一开始所有报错都指向“COM3 被占用”，我花了很多时间在关进程、换线、换口上。直到点开“获得开发板信息”，看到 VID `0x1A86`，才意识到问题不是端口被谁占了，而是**板子的串口芯片类型和 IDE 配置不匹配**。这个转折点把排查方向从“软件占用”彻底转向了“硬件识别”。

## 4. 验证

- 验证方式：
  1. 安装 CH343 驱动、改 `USB CDC On Boot` 为 `Disabled` 后，重新上传。
  2. 观察 Arduino IDE 是否显示“上传成功”。
  3. 上传成功后，打开串口监视器，波特率 115200，看是否稳定输出麦克风峰值。
- 结果：
  - 上传成功，Arduino IDE 显示正常写入。
  - 串口监视器稳定输出，峰值随声音变化，不再出现“输出两行就断”的现象。

## 5. 总结与反思

- 这次学到的：软硬接驳项目中问题的原因是很复杂的，比如这次看起来是软件上的问题实际上是硬件识别不通过
- 下次遇到类似问题，先检查什么：开发板信息

## 6. 核心概念

### 下载模式

ESP32-S3 芯片上电时，会读取几个**Strapping 引脚**的电平，决定进入哪种启动模式。其中最关键的是 **GPIO0**：

- **GPIO0 为高电平**（默认）→ 芯片从 Flash 启动，运行你烧录的程序。
- **GPIO0 为低电平** → 芯片进入**下载模式**，等待 esptool 通过串口发送固件。

开发板上的 **BOOT 键**就是用来把 GPIO0 拉低的。正常上传时，esptool 会通过串口的 DTR/RTS 信号自动控制 EN 和 GPIO0，让芯片自动进入下载模式，不需要手动按键。

但当**自动复位电路失效**（比如 CH343 驱动不匹配、USB 线质量差、供电不稳）时，自动进入下载模式会失败。这时就需要**手动进下载模式**：

1. 按住 BOOT 键（GPIO0 拉低）
2. 按一下 RESET 键（EN 拉低，芯片复位）
3. 松开 RESET
4. 松开 BOOT

芯片就会停在下载模式，等待 esptool 写入。上传完成后，按一下 RESET 或断电重启，芯片会重新从 Flash 启动，运行新程序。

**注意**：如果用的是 ESP32-S3 原生 USB-Serial/JTAG（不是 CH343），手动进下载模式后，esptool 无法用默认复位方式退出下载模式，必须按 EN 键或断电重启，让芯片重新采样启动引脚。