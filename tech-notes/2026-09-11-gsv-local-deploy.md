# 标题：在主机电脑的 WSL 中部署 GSV 智能体并接入云端

## 1. 背景

- 目标：将主机电脑（Windows + WSL）注册为 GSV 云端智能体的执行节点，使 Agent 能够通过 `shell.exec`、`fs.*`、`net.fetch` 等能力控制本机。

- 遇到的具体问题：
  1. `gsv-install.sh` 安装脚本在下载二进制文件时被中断（`Connection reset by peer` / `unexpected eof while reading`），无法自动完成安装。
  2. 手动安装后，`gsv daemon install` 启动服务失败，`gsv daemon status` 报 `Failed to read gsvd service status`。
  3. `gsvd` 守护进程启动时报 `Missing non-interactive device credential`，无法读取 Token。

- 为什么这个问题值得记录：GSV 的 CLI 工具与守护进程读取的配置来源不一致，导致“看起来配置成功了，但守护进程读不到”的问题。这类工具链内部配置路径不一致的坑，搜索不到现成答案，需要手动排查。

## 2. 排查过程

- 现象：
  - 安装脚本下载二进制文件时，curl 报 `Connection reset by peer` 或 `unexpected eof while reading`。
  - `gsv daemon status` 显示 `Active: activating (auto-restart) (Result: exit-code)`，进程反复崩溃。
  - `gsv daemon diagnostics` 报 `Io(Os { code: 2, kind: NotFound })`。
  - `gsv daemon logs` 报 `Log file not found`，说明 `gsvd` 在写日志前就已崩溃。
  - 手动前台运行 `~/.gsv/bin/gsvd --foreground` 时，报 `Missing non-interactive device credential`。

- 尝试过的方法（包括失败的）：
  1. 多次重试 `bash gsv-install.sh`，均在下载阶段失败。
  2. 从已成功安装的另一台电脑手动拷贝 `gsv-linux-x64`、`gsvd-linux-x64`、`gsv-desktop-linux-x64` 三个文件，手动安装到 `~/.gsv/bin/`。
  3. 手动创建 `~/.gsv/config.toml`，写入 `gateway.url` 和 `device.token`，但 `gsvd` 仍然崩溃。
  4. 检查发现 `config.toml` 中 `token` 值缺少双引号，修正后仍然崩溃。
  5. 使用 `gsv daemon doctor` 确认可执行文件和服务定义均正常。
  6. 使用 `gsv daemon diagnostics`，报 `NotFound`，确认是配置文件路径问题。

- 关键线索：
  1. 手动前台运行 `gsvd` 时，报错信息直接指向 `device.token` 缺失，说明它读的不是 `~/.gsv/config.toml`。
  2. 通过命令行参数 `--token` 直接传给 `gsvd` 后，服务成功启动并连接云端，说明 Token 本身有效，问题出在配置加载路径上。
  3. 将 Token 写入 `~/.bashrc` 的 `GSV_TOKEN` 环境变量后，`gsv daemon restart` 成功，说明 `gsvd` 优先读取的是环境变量，而不是 `config.toml`。

- 根本原因：
  - `gsv` CLI 与 `gsvd` 守护进程读取配置的来源不一致。`gsv config --local set` 写入的配置并未被 `gsvd` 读取，`gsvd` 期望的凭证来自环境变量 `GSV_TOKEN` 或它自己路径下的配置。
  - 因此，“配置看起来写成功了”，但 `gsvd` 启动时依然找不到 Token，导致启动即崩溃。

## 3. 解决方案

- 具体步骤：
  1. 从已成功安装的电脑手动拷贝三个二进制文件到目标主机的 `~/.gsv/bin/`。
  2. 赋予可执行权限，并确保 `~/.gsv/bin` 在 `PATH` 中。
  3. 创建 Token：
     ```bash
     gsv auth token create --kind device --device esf-computer
     ```
  4. 将 Token 写入环境变量：
     ```bash
     echo 'export GSV_TOKEN="你的Token"' >> ~/.bashrc
     source ~/.bashrc
     ```
  5. 重启服务：
     ```bash
     gsv daemon restart
     gsv daemon status
     ```

- 关键命令/代码片段：
  ```bash
  # 手动安装二进制文件
  chmod +x gsv-linux-x64 gsvd-linux-x64 gsv-desktop-linux-x64
  mkdir -p ~/.gsv/bin
  cp gsv-linux-x64 ~/.gsv/bin/gsv
  cp gsvd-linux-x64 ~/.gsv/bin/gsvd
  cp gsv-desktop-linux-x64 ~/.gsv/bin/gsv-desktop
  echo 'export PATH="$HOME/.gsv/bin:$PATH"' >> ~/.bashrc
  source ~/.bashrc

  # 写入 Token 环境变量
  echo 'export GSV_TOKEN="你的Token"' >> ~/.bashrc
  source ~/.bashrc

  # 重启服务
  gsv daemon restart
  gsv daemon status
  ```

- 需要注意的坑：
  - `~/.gsv/config.toml` 中的 `token` 值必须用双引号包裹（TOML 语法）。
  - `gsv config --local set device.token` 写入的配置，`gsvd` 不一定读取，优先使用 `GSV_TOKEN` 环境变量。
  - WSL 的 systemd 用户服务在部分环境下不稳定，若服务模式反复失败，可先用 `gsvd --foreground` 前台运行验证。

- **转折：意识到 CLI 和 daemon 读取配置的来源不一致，而不是 Token 本身无效。**

## 4. 验证

- 验证方式：
  1. `gsv daemon status` 显示 `Active: active (running)` 且 `phase: Connected`。
  2. 在 GSV 控制台刷新页面，点击 MACHINES 节点，能看到 `esf-computer` 出现在设备列表中。
  3. 对 Friday 发送指令，让它在本机上执行 `shell.exec` 或 `fs.*` 操作，能成功返回结果。

- 结果：主机电脑成功注册为 GSV 执行节点，具备 `fs.*`、`shell.exec`、`net.fetch` 三种能力，云端的 Friday 可以控制本机。

## 5. 总结与反思（下次怎么更快）

- 这次学到的：注意语法细节、
- 下次遇到类似问题，先检查什么：

## 6. 核心概念

1. **CLI（Command Line Interface，命令行界面）**

   CLI 是你在终端里敲命令时调用的工具。它的特点是“**用完即走**”：你敲一次命令，它执行一次，执行完就退出，不会在后台常驻。

   在这次部署中，`gsv` 就是一个 CLI 工具。当你输入 `gsv daemon install` 或 `gsv auth token create` 时，你调用的就是它。它负责写配置、发指令、查状态，但不会一直待在后台。

2. **守护进程（Daemon）**

   守护进程是一个在后台长期运行、不打扰你的程序。它的特点是“**一直在岗**”：开机自动启动，不需要你手动触发，默默等待任务。

   在这次部署中，`gsvd` 就是一个守护进程。它负责读取配置、连接云端、等待 Agent 发来的指令，并在本机执行。你看到的 `phase: Connected` 就是它在后台保持连接的状态。

   两者的关系可以概括为：**CLI 负责“配置和指挥”，守护进程负责“持续执行”。** 你这次踩坑的根本原因，就是 CLI（`gsv`）写入配置的位置，和守护进程（`gsvd`）读取配置的位置不一致。