# 标题：验证 PX4 在 Offboard 模式下的 setpoint 超时阈值与 MAVSDK 保活机制

## 1. 背景

- 目标：在学习 MAVSDK Offboard 模式时，搞清 PX4 对 setpoint 流的超时机制，确定"停发 setpoint 后飞控多久退出 Offboard"，以便设计正确的发送频率。
- 遇到的具体问题：PX4 官方文档称 Offboard 超时默认为 0.5 秒，但实际测试中停发 setpoint 长达 10 秒，飞控仍停留在 Offboard 模式，与文档不符。
- 为什么这个问题值得记录：超时机制直接决定 Offboard 代码的写法。若误以为"发一次即可长期生效"，在真机上会因超时退出触发失控保护。搞清是 PX4 放宽了超时、还是 MAVSDK 在底层保活，是理解整个 Offboard 控制链的关键。

## 2. 排查过程

- 现象：
  - 编写脚本依次测试停发 setpoint 2、3、4、5、7、10 秒，每次停发后通过 telemetry.flight_mode() 读取模式，结果全部仍为 OFFBOARD。
  - 每次测试结束时调用 offboard.stop()，QGC 中模式短暂变为 HOLD，下一个测试开始后又变回 OFFBOARD。
  - 在发送循环中加入逐条打印后，停发期间终端不再输出发送记录，确认 Python 应用层确实停止了发送，但 QGC 中模式仍为 OFFBOARD。

- 尝试过的方法（包括失败的）：
  1. 第一版测试脚本用 mode 与字符串 OFFBOARD 做等值比较，结果实时输出显示 OFFBOARD、汇总表却全部显示"否"。原因是 flight_mode() 返回的是枚举对象而非字符串，改为用字符串包含判断后恢复正常。
  2. 将测试时长从 0.2 到 2 秒扩展到 2 到 10 秒，飞控始终未退出 Offboard。
  3. 在发送循环里加逐条打印，用于区分"应用层是否在发"和"飞控是否收到"。

- 关键线索：
  - 应用层已停止发送（终端无打印），但飞控模式仍为 OFFBOARD，说明有应用层之外的东西在持续发送 setpoint。
  - offboard.stop() 能立即让模式变为 HOLD，说明退出动作本身有效，超时未触发并非因为飞控"卡死"。

- 根本原因：MAVSDK 的 offboard 模块在内部维护了一个"看门狗"，会在应用层调用过 set_velocity_ned() 或 set_position_ned() 之后，于后台以固定频率自动重发最后一次 setpoint。因此应用层停发并不会导致飞控超时退出，真正的退出只发生在显式调用 offboard.stop() 时。

## 3. 解决方案

- 具体步骤：
  1. 用字符串包含判断替代等值比较，修正枚举与字符串比较的 bug。
  2. 在发送循环内加逐条打印，区分"应用层发送"与"底层保活"。
  3. 依次测试停发 2、5、10 秒，观察终端打印与 QGC 模式变化。
  4. 确认结论：应用层停发后飞控仍处于 Offboard，退出只由 offboard.stop() 触发。

- 关键命令/代码片段：

  mode = await get_flight_mode(drone)
  still_offboard = ("OFFBOARD" in str(mode))

  async def send_setpoint_loop(drone, north, east, down, yaw, duration, hz=10):
      interval = 1.0 / hz
      end_time = asyncio.get_event_loop().time() + duration
      count = 0
      while asyncio.get_event_loop().time() < end_time:
          await drone.offboard.set_velocity_ned(
              VelocityNedYaw(north, east, down, yaw)
          )
          count += 1
          print("发送第", count, "次:", north, east, down, yaw)
          await asyncio.sleep(interval)

- 需要注意的坑：
  - telemetry.flight_mode() 返回的是枚举对象，直接与字符串比较会得到 False，必须转成字符串再比较。
  - 不能假设"发一次 setpoint 就长期生效"。在 MAVSDK 下能跑通，是因为底层保活；换用 pymavlink 直接发 MAVLink 消息、或 MAVSDK 进程崩溃时，飞控会在约 0.5 秒内退出 Offboard。
  - 依赖底层保活是不安全的，关键逻辑（如切换目标点）应主动发送 setpoint，让控制意图明确。

- 转折：意识到"停发不退出"不是 PX4 放宽了超时，而是 MAVSDK 在底层替应用层保活。

## 4. 验证

- 验证方式：
  1. 在发送循环内逐条打印，确认停发期间应用层确实不再发送。
  2. 停发期间观察 QGC 模式，确认仍为 OFFBOARD。
  3. 调用 offboard.stop() 后观察 QGC 模式，确认立即变为 HOLD。

- 结果：确认 PX4 的超时机制被 MAVSDK 的底层保活所覆盖，应用层停发不会导致退出，显式 offboard.stop() 才会退出。理解了 Offboard 的真实控制链：应用层 setpoint 到 MAVSDK 看门狗重发再到飞控执行，以及真机上必须自行保证发送可靠性的结论。

## 5. 总结与反思

- 这次学到的：通过观察QGC中Hold模式的短暂切换从而更快发现了MAVSDK保活机制的作用，节省了排查时间
- 下次遇到类似问题，先检查什么：在运行检查程序时应注意多个窗口中的变化，有时可起到加快问题排查的效果

## 6.核心概念

1. QGC 的 Hold 模式

   Hold（位置保持）是 PX4 的一种飞行模式，飞控会利用 GPS 或光流等位置信息，把无人机稳定在当前经纬度和高度上悬停。它和 Offboard 的关键区别是：Offboard 下飞控执行外部程序发来的 setpoint，Hold 下飞控自己维持位置。当 Offboard 因超时或主动 stop 退出时，PX4 的默认 fallback 就是切到 Hold，避免无人机继续盲飞。本次测试中，offboard.stop() 后 QGC 里短暂出现的 Hold，正是这个 fallback 在起作用。

2. 失控保护

   失控保护（Failsafe）是飞控在检测到控制链路异常时自动采取的安全措施。触发条件包括：遥控信号丢失、Offboard setpoint 超时、GPS 丢失、数据链中断等。触发后飞控会按预设策略动作，常见的有悬停、返航、降落。Offboard 模式下的 setpoint 超时（PX4 参数 COM_OF_LOSS_T，默认约 0.5 秒）就是一种失控保护：外部程序一旦停止发送，飞控就认为"控制端已失效"，自动退出 Offboard 并切到安全模式。这也是为什么依赖 MAVSDK 底层保活存在风险——真机上保活一旦中断，失控保护会立即接管。

3. setpoint 流

   setpoint 是外部程序发给飞控的"目标设定值"，可以是目标位置（set_position_ned）、目标速度（set_velocity_ned）或目标姿态（set_attitude）。Offboard 模式的本质是"setpoint 流"：外部程序以固定频率持续发送 setpoint，飞控持续执行。它不是"发一次就永久生效"的配置，而是一条条实时消息。飞控用超时机制判断这条流是否还活着——流断了，就退出 Offboard。因此正确的 Offboard 代码必须保证 setpoint 持续、稳定地发送，MAVSDK 的底层保活只是对这一要求的封装，不能替代对机制本身的理解。