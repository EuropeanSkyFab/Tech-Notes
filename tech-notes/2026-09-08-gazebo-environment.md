# 标题：在 RTX5060 和 i5-14400 的WSL环境中打通Gazebo与QGC的MAVLink通信

## 1. 背景
- 目标：在wsl中配置Ubuntu-22.04，安装PX4飞控与gazebo物理模拟软件，并令其连接QGC地面站
- 遇到的具体问题：1.在 WSL 中配置 PX4 + Gazebo 环境，编译成功后，Gazebo 窗口无法显示
                2.PX4 仿真正常启动后，Gazebo 窗口显示，但 QGroundControl（QGC）无法连接到 PX4。
                3.连接成功后，无人机仍无法解锁/起飞，提示 Arming denied: Resolve system health failures first
                4.尝试在 PX4 控制台修改参数 COM_RC_IN_MODE 0，但 param 命令不可用，无法修改参数。
                5.因WSL网络问题，转向Docker，但拉取镜像时遇到网络问题（EOF、no such host、402 Payment Required），国内镜像加速器也纷纷失败（白名单限制、域名无法解析）。
                6.决定安装 Ubuntu 双系统以绕过 WSL 和 Docker 的种种限制，但安装完成后，系统无法进入桌面，卡在 BusyBox（initramfs），找不到根分区。
                7.转向SwarmBox以配置PX4+Gazebo，但遇到网络波动，下载进程受阻
                8.回到原方案尝试通过数据流打通gazebo与QGC间的通信，但仍然受阻
                (PS:1、2、3、4、6、7为环境配置问题，5、8为网络问题)
- 为什么这个问题值得记录：在AI指导下尝试多条路径耗费十多个小时仍无法解决环境配置问题，说明原因可能不正常

## 2. 排查过程
- 现象：问题1中调出xeyes，发现xeyes正常显示，说明WSL图形显示正常
        问题5中在docker拉取镜像报错信息为“连接超时”或“地址不存在”
        问题6中Ubuntu在执行Linux命令时出现“找不到根分区”的报错
        问题8中解析gazebo与QGC的数据流，发现其为非MAVlink数据，与传统MAVlink协议不匹配
        且问题1在设计gazebo启动操作中反复出现
- 尝试过的方法：1.将Ubuntu-22.04更新至Ubuntu-26.04
                2.安装配置X server
                3.尝试修复找不到根分区问题（chroot、更新内核、重新生成 initramfs/GRUB）
                4.转向Swarmbox尝试替代WSL以避免兼容问题
- 关键线索：1.意识到可能与本地的GeForce RTX5060显卡与inter i5-14400CPU硬件有关
            2.多次排查gazebo与QGC之间的通信问题，发现地址、端口、数据流等正常
- 根本原因：1.WSL的默认 NAT网络模式会将PX4发出的MAVLink数据包隔离在虚拟网络内，导致Windows端的QGC无法接收到数据。
            2.排查发现gazebo与QGC之间数据流为非MAVlink协议信息，导致QGC无法解析数据流进而导致连接失败
- **转折：意识到问题出现在网络连接层而非配置层

## 3. 解决方案
- 具体步骤：开启WSL的“镜像网络模式”（Mirrored Networking Mode）：进入C:\Users\EuropeanSkyFab\目录，创建并手动配置“.wslconfig”的.txt配置文件，随后执行wsl --shutdown以
            重启WSL，最后重新打开QGC以验证链接
- 关键命令/代码片段：
        [wsl2]
        networkingMode=mirrored
- 需要注意的坑：一般来说安装有wsl后用户目录会自动创建WSL配置文件了，但某些情况下（比如本次）会出现无WSL配置文件的情况，此时就需要手动创建
                （顺带一提，开启镜像模式还是在本人在半崩溃状态下逼问AI“凭什么别人没问题就我有问题”时AI给的“常用方法”，可见提示词工程有多必要）

## 4. 验证
- 验证方式：QGC地面站显示连接成功，并且QGC发送“起飞”指令时，gazebo显示无人机正常起飞
- 结果：成功打通gazebo与QGC的信号链路，所需软件环境配置完毕

## 5. 总结与反思
- 这次学到的：Linux基础命令、MAVlink协议、双系统概念、软硬件兼容问题
- 下次遇到类似问题，先检查什么：1. 硬件与系统版本是否匹配（RTX 5060 需要较新内核）
                            2. WSL 网络模式是否为 mirrored
                            3. MAVLink 数据流是否被正确转发
                            4. QGC 端口和 IP 配置是否正确

## 6. 核心概念

1. **WSL（Windows Subsystem for Linux）**

   WSL 是 Windows 上运行 Linux 程序的兼容层，让你不用装双系统或虚拟机就能在 Windows 里使用 Linux 工具。它有两个版本：WSL 1 是系统调用翻译层，WSL 2 则是轻量级虚拟机，内置完整的 Linux 内核。WSL 2 默认使用 NAT 网络模式，会把 Linux 内部的网络流量隔离在虚拟网络中，这正是本次 QGC 无法收到 MAVLink 数据的根本原因。

2. **地面站（Ground Control Station, GCS）**

   地面站是无人机的地面控制中心，通常是一个软件界面，用来实时监控无人机状态、发送飞行指令、规划航线、修改飞控参数。QGroundControl（QGC）就是 PX4 生态中最常用的地面站。它通过 MAVLink 协议与飞控通信，把姿态、位置、电量等数据可视化，并允许你点击“起飞”“降落”等按钮控制无人机。

3. **数据流（Data Stream）**

   数据流指的是在设备之间持续传输的数据序列。在无人机系统中，飞控会不断向地面站发送姿态、位置、传感器读数等数据流。本次问题中，PX4 确实在发送数据流，但由于 WSL 的 NAT 网络隔离，数据流没有到达 Windows 端的 QGC，导致 QGC 无法解析并建立连接。

4. **MAVLink 协议**

   MAVLink 是无人机与地面站之间通信的标准协议，全称 Micro Air Vehicle Link。它定义了消息的格式和规则，包括心跳包、姿态数据、位置信息、任务指令等。PX4 和 QGC 都依赖 MAVLink 进行双向通信。本次排查中，你发现 QGC 收到的数据流首字节不是 MAVLink 的标准起始标志（`0xFE`），说明数据流被网络层干扰，不是 QGC 能识别的有效 MAVLink 消息。

5. **NAT 网络模式**

   NAT（Network Address Translation，网络地址转换）是 WSL 2 默认的网络模式。在这种模式下，WSL 内部有一个独立的虚拟网卡和 IP 地址，它与 Windows 主机不在同一个网络里。WSL 访问外部网络时，会通过 Windows 主机做一层地址转换，就像家里多台设备共用同一个公网 IP 一样。

   问题在于，**外部设备无法主动访问 WSL 内部的服务**，WSL 内部的广播包也不会自动转发到 Windows 主机。PX4 在 WSL 里向 `14550` 端口发送 MAVLink 数据时，数据被困在 WSL 的虚拟网络内，Windows 端的 QGC 根本收不到。只有开启镜像网络模式（Mirrored Networking Mode）后，WSL 与 Windows 共享同一个网络栈，数据才能被 QGC 接收。