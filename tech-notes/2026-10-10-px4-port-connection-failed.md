# 标题：MAVSDK 脚本启动时报 Failed to bind server to port 50051，更换端口后解决
# 涉及层次：代码、环境


## 1. 背景
- 目标：在 WSL 2 + Ubuntu 22.04 下运行 MAVSDK-Python 的 Offboard 控制脚本 `offboard-velocityned.py`，让 PX4+Gazebo 仿真无人机按速度指令飞行。
- 遇到的具体问题：脚本启动时报错 `Failed to bind server to port 50051 (grpc_server.cpp:180)`，导致 MAVSDK 无法连接到无人机。
- 为什么这个问题值得记录：这个报错表面看是“端口被占用”，但实际情况并非如此。排查过程中容易误判方向，记录清楚可以避免下次浪费时间。


## 2. 排查过程
- 现象：运行 `python3 offboard-velocityned.py` 后，终端输出 `Connection using udp:// is deprecated, please use udpin:// or udpout://`，紧接着报 `Failed to bind server to port 50051`。
- 尝试过的方法（包括失败的）：
  1. 用 `ss -ltnp | grep 50051` 检查端口占用，无输出。
  2. 用 `ps aux | grep mavsdk` 发现一个僵尸进程 `[mavsdk_server] <defunct>`，其父进程是之前的脚本 `offboard-velocityned.py`。
  3. `kill -9` 父进程后，僵尸进程被回收，`ps` 确认干净。
  4. 重新运行脚本，仍然报同样的错。
  5. 用 `sudo ss -ltnp | grep 50051` 再次确认，依然无输出，说明没有任何进程占用 50051。
- 关键线索：端口没有被任何进程占用，但 MAVSDK 仍然报“无法绑定”。说明问题不在“占用”，而在“绑定”这个动作本身。
- 根本原因：MAVSDK 内部 gRPC 服务默认使用 50051 端口，在 WSL 环境下可能因为内核状态、网络命名空间残留或 gRPC 自身行为导致绑定失败。换一个端口即可绕开。


## 3. 解决方案
- 具体步骤：
  1. 在 Python 脚本中，将 `System()` 改为显式指定另一个端口，例如 `System(port=50552)`。
  2. 重新运行脚本，MAVSDK 不再报端口绑定错误，正常连接无人机。
- 关键命令/代码片段：
````python
from mavsdk import System

async def run():
    drone = System(port=50552)
    await drone.connect(system_address="udpin://0.0.0.0:14540")
````
  - 修改脚本：`drone = System(port=50552)`
  - 连接无人机：`await drone.connect(system_address="udpin://0.0.0.0:14540")`
  - 终端运行：`python3 offboard-velocityned.py`
- 需要注意的坑：
  - `System(port=...)` 指定的端口是 MAVSDK 内部 gRPC 服务端口，不是无人机通信端口（无人机通信端口是 `system_address` 里的 14540）。
  - 端口号不要重复使用同一个值，避免与残留状态冲突。可以每次换一个不常用的端口，比如 50552、50553。
  - 不要用 VSCode 的“调试运行”按钮跑 MAVSDK 脚本，容易留下僵尸进程。直接在 WSL 终端里运行。


## 4. 验证
- 验证方式：重新运行 `python3 offboard-velocityned.py`。
- 结果：终端不再输出 `Failed to bind server to port 50051`，脚本正常进入“等待无人机连接...”状态，后续 Offboard 控制流程正常执行。


## 5. 总结与反思
- 这次学到的：端口的问题除了占用问题，也可能是纯粹无法连接导致的
- 下次遇到类似问题，先检查什么：检查是否真的没有占用，如果真的没有可以选择换个端口


## 6.核心概念


1. 僵尸进程（Zombie Process）：
    一个已经终止运行、但尚未被父进程回收的进程。它不占用 CPU 和内存，只保留一个进程表项，状态显示为 `Z` 或 `<defunct>`。`kill` 命令对它无效，必须杀掉其父进程，或由 init（PID 1）接管后自动回收。

2. 端口占用（Port Occupied）：
    操作系统通过“端口号”区分不同的网络服务。当一个进程绑定了某个端口（如 50051）后，其他进程无法再绑定同一端口，直到该端口被释放。但“绑定失败”不等于“被占用”，也可能是内核状态、网络命名空间或程序自身行为导致，此时换端口是最快的排查方式。