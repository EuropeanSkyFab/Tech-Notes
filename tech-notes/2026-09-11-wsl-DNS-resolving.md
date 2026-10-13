# 标题：解决 WSL 中 DNS 解析失败导致 pip 无法安装 MAVSDK 的问题

## 1. 背景

- 目标：在 WSL 中安装 MAVSDK-Python，用于编写无人机控制脚本。

- 遇到的具体问题：执行 `pip3 install mavsdk` 时，pip 反复报 `Temporary failure in name resolution`，无法解析 `pypi.org`，安装失败。

- 为什么这个问题值得记录：DNS 解析失败是 WSL 网络配置中的常见问题，尤其在开启镜像网络模式后，WSL 会自动生成一个虚拟 DNS 网关，但这个网关有时无法解析外网域名。这个问题不解决，所有需要联网的操作（pip、apt、git clone）都会失败。

## 2. 排查过程

- 现象：
  - `pip3 install mavsdk` 报 `NewConnectionError` 和 `Temporary failure in name resolution`。
  - `ping -c 3 pypi.org` 同样报 `Temporary failure in name resolution`。
  - 查看 `/etc/resolv.conf`，发现 `nameserver` 指向 `10.255.255.254`（WSL 虚拟 DNS 网关）。

- 尝试过的方法（包括失败的）：
  1. 直接重试 `pip3 install mavsdk`，仍然失败。
  2. 检查 `/etc/resolv.conf`，确认 DNS 配置指向的是 WSL 内部网关，而非公共 DNS。

- 关键线索：
  - `/etc/resolv.conf` 中的 `nameserver 10.255.255.254` 是 WSL 在镜像网络模式下自动生成的虚拟 DNS 网关。
  - 这个网关本身无法解析外网域名，需要手动替换为公共 DNS。

- 根本原因：
  - WSL 自动生成的 DNS 配置（`10.255.255.254`）在当前网络环境下无法完成外网域名解析。
  - 需要手动将 `/etc/resolv.conf` 中的 `nameserver` 替换为公共 DNS（如 `8.8.8.8`），并在必要时禁用 WSL 的自动生成，防止重启后被覆盖。

## 3. 解决方案

- 具体步骤：
  1. 编辑 `/etc/resolv.conf`，替换 `nameserver` 为公共 DNS。
  2. 测试网络连通性。
  3. 安装 MAVSDK。
  4. 为防止重启后 DNS 被覆盖，编辑 `/etc/wsl.conf` 禁用自动生成。

- 关键命令/代码片段：
  ```bash
  # 1. 替换 DNS
  sudo nano /etc/resolv.conf
  # 写入以下内容：
  # nameserver 8.8.8.8
  # nameserver 114.114.114.114

  # 2. 测试网络
  ping -c 3 pypi.org

  # 3. 安装 MAVSDK（使用清华镜像加速）
  pip3 install mavsdk -i https://pypi.tuna.tsinghua.edu.cn/simple

  # 4. 禁止 WSL 自动生成 resolv.conf
  sudo nano /etc/wsl.conf
  # 写入：
  # [network]
  # generateResolvConf = false
  # 保存后，在 Windows PowerShell 执行 wsl --shutdown 重启 WSL
  ```

- 需要注意的坑：
  - WSL 每次重启都会自动生成 `/etc/resolv.conf`，如果不设置 `generateResolvConf = false`，手动修改的 DNS 会在重启后丢失。
  - 禁用自动生成后，需要手动确保 `/etc/resolv.conf` 存在且内容正确，否则 WSL 可能完全无法解析 DNS。
  - 使用国内镜像源（清华 PyPI 镜像）可以加速 pip 安装，避免因网络波动导致安装中断。

- **转折：意识到问题出在 WSL 自动生成的 DNS 网关无法解析外网域名，而非 pip 或网络本身故障。**

## 4. 验证

- 验证方式：
  1. `ping -c 3 pypi.org` 返回正常的 ICMP 响应，说明 DNS 解析已恢复。
  2. `pip3 install mavsdk` 成功完成安装，无报错。
  3. 运行 `python3 -c "import mavsdk; print(mavsdk.__version__)"`，能打印出版本号。

- 结果：MAVSDK-Python 成功安装，WSL 网络恢复正常，可以继续编写无人机控制脚本。

## 5. 总结与反思

- 这次学到的：网络下载进程中DNS解析路径的重要性
- 下次遇到类似问题，先检查什么：检查虚拟DNS是否可以正常解析

## 6. 核心概念

1. **虚拟 DNS 网关**

   在 WSL 2 的镜像网络模式下，WSL 会自动生成一个虚拟 DNS 网关（如 `10.255.255.254`），用于转发 WSL 内部的 DNS 请求。它的作用是让 WSL 里的程序能通过 Windows 主机的网络栈解析域名。但在某些网络环境下，这个虚拟网关本身无法完成解析，导致 WSL 内的所有域名请求失败。此时需要手动将 DNS 替换为公共 DNS。

2. **公共 DNS**

   公共 DNS 是由第三方机构提供的、面向所有人开放的域名解析服务。常见的公共 DNS 包括 Google 的 `8.8.8.8`、国内电信的 `114.114.114.114`、阿里云的 `223.5.5.5` 等。它们的优点是稳定、解析速度快，不依赖本地网络环境。当 WSL 的虚拟 DNS 网关失效时，手动指定公共 DNS 可以绕过这个问题，恢复正常的域名解析。