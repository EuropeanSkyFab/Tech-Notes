# 标题：STM32 第一次烧录后 ST-Link 无法连接（芯片被锁）

# 涉及层次：环境、配置、代码


## 1. 背景

- 目标：用 STM32F103C8T6 做状态机作业，通过 ST-Link 烧录程序。
- 遇到的具体问题：第一次烧录成功，之后每次烧录都报 `Error in initializing ST-LINK device. Reason: No device found on target.`
- 为什么这个问题值得记录：换板子、换 ST-Link、换杜邦线、重装驱动、重启电脑都无效，容易误判为硬件损坏，实际是软件配置问题。


## 2. 排查过程

- 现象：
  - 新板子第一次烧录成功，之后所有烧录都报 `No device found on target`
  - 设备管理器能识别 ST-Link
  - CubeProgrammer 能识别 ST-Link 序列号、电压（3.27V）
  - 但始终无法获取芯片 ID（`Error: Unable to get core ID`）

- 尝试过的方法（包括失败的）：
  - 更换杜邦线：无效
  - 更换 STM32 板子：新板子第一次能烧，之后仍然被锁
  - 更换 ST-Link：无效
  - 重装 ST-Link 驱动：无效
  - 重启电脑：无效
  - CubeProgrammer 的 `Under reset` / `Hardware reset` / `Hot plug` 模式：均报 `DEV_TARGET_HELD_UNDER_RESET` 或 `No STM32 target found`
  - 按住 RESET 键连接：无效

- 关键线索：
  - 板载 PC13 在第一次烧录前是闪烁的（出厂程序），烧录后变成常亮
  - 说明用户程序在运行，但 SWD 引脚被禁用了

- 根本原因：
  - CubeMX 中 `System Core → SYS → Debug` 配置为 `No Debug`
  - 生成的代码不初始化 SWD 调试接口（PA13/PA14）
  - 芯片烧录后运行用户程序，SWD 引脚不再作为调试口，ST-Link 无法连接
  - 新板子出厂时 SWD 默认为开启状态，所以第一次能烧录；烧录后用户程序未配置 SWD，芯片即被"锁"


## 3. 解决方案
- 具体步骤：

  **第一步：救回被锁的芯片（BOOT0 大法）**
  1. 断开 ST-Link 与板子的 4 根杜邦线
  2. 将板子上的 BOOT0 跳线帽从 `0` 拔到 `1`（系统存储器启动模式）
  3. 按一下 RESET 键
  4. 重新连接 ST-Link，打开 STM32CubeProgrammer
  5. `模式` 选 `Normal`，`复位模式` 选 `Hardware reset`，点 `连接`
  6. 连接成功后，点左侧 `擦除` 图标 → `全片擦除`
  7. 擦除完成后，将 BOOT0 跳线帽拔回 `0`
  8. 按一下 RESET 键

  **第二步：修复工程配置（根本解决）**
  1. CubeMX 中打开 `.ioc` 文件
  2. `System Core → SYS → Debug` 改为 `Serial Wire`
  3. `GENERATE CODE` 重新生成代码
  4. VS Code 中重新编译、烧录
  5. 之后多次烧录均正常

- 关键配置：
  - `SYS → Debug = Serial Wire`（不是 `No Debug`）
  - BOOT0 = 1 时芯片从系统存储器启动，不运行用户程序，SWD 保持可用

- 需要注意的坑：
  - `No Debug` 是 CubeMX 的默认选项之一，新建工程时容易忽略
  - 一旦烧录了禁用 SWD 的程序，后续只能通过 BOOT0 跳线或 Connect under Reset 救回
  - 换硬件无法解决此问题，因为是软件配置导致
  - 芯片被锁后，CubeProgrammer 的 `Under reset` 模式可能无效，必须用 BOOT0 方式


## 4. 验证

- 验证方式：
  - 修改 `SYS → Debug = Serial Wire` 并重新烧录后，连续多次烧录
  - 每次烧录均正常，不再报 `No device found on target`

- 结果：
  - 多次烧录成功，ST-Link 能稳定连接
  - 板子按预期运行状态机程序


## 5. 总结与反思

- 这次学到的：更换杜邦线后问题仍存在不代表板子有问题，可能是配置问题
- 下次遇到类似问题，先检查什么：芯片是否无意间被锁了


## 6.核心概念

### 6.1 SWD

- **全称**：Serial Wire Debug，串行线调试。
- **是什么**：ARM Cortex-M 芯片的**两线调试接口**，只用两根线就能实现烧录和调试。
- **两个引脚**：
  - `SWDIO`：数据线（STM32F103 上是 PA13）
  - `SWCLK`：时钟线（STM32F103 上是 PA14）
- **和 JTAG 的区别**：JTAG 需要 4~5 根线，SWD 只要 2 根，引脚更省，更适合小封装芯片。
- **为什么会被禁用**：
  - STM32 的 PA13/PA14 是**复用引脚**，默认功能是 SWD，但也可以配成普通 GPIO。
  - 如果代码里把 PA13/PA14 配成 GPIO，或 CubeMX 的 `SYS → Debug` 选了 `No Debug`，芯片跑起来后 SWD 就被禁用，ST-Link 再也连不上，俗称"锁芯片"。
- **在 CubeMX 里的对应配置**：`System Core → SYS → Debug → Serial Wire`。

### 6.2 系统存储器启动模式

- **是什么**：STM32 的一种启动方式，芯片上电后**不从用户 Flash 启动，而是从 ST 出厂预烧的 bootloader 启动**。
- **怎么进入**：
  - 把 `BOOT0` 引脚拉高（跳到 1），`BOOT1`（PB2）拉低（跳到 0）。
  - 按一下 RESET，芯片进入系统存储器启动模式。
- **bootloader 是什么**：ST 出厂时烧在芯片内部一段只读区域的程序，用于通过串口、USB、CAN 等接口烧录 Flash。**它不是用户程序，不占用用户 Flash 空间。**
- **为什么能救砖**：
  - 用户程序运行时会禁用 SWD。
  - 但 bootloader **不碰 PA13/PA14**，SWD 保持可用。
  - 所以 BOOT0 = 1 时，ST-Link 能正常连接芯片，可以擦除或重新烧录。
- **三种启动模式对比**：

| BOOT0 | BOOT1 | 启动位置 | 用途 |
|-------|-------|---------|------|
| 0 | x | 主 Flash | 正常运行用户程序 |
| 1 | 0 | 系统存储器 | 运行 bootloader（救砖、串口下载） |
| 1 | 1 | 内置 SRAM | 调试用，几乎不用 |

- **注意事项**：
  - 救砖完成后，**必须把 BOOT0 拨回 0**，否则芯片会一直停在 bootloader，不运行用户程序。
  - 系统存储器启动模式下，用户程序不会被执行，所以现象是"板子什么都不做"。
