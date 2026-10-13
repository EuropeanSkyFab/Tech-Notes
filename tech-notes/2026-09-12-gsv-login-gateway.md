# 标题：解决 GSV CLI 中用户登录成功但后续命令仍报 401 认证失败的问题

## 1. 背景

- 目标：在主机电脑的 WSL 中，通过 `gsv chat` 与 GSV 云端智能体对话，并让它在本机执行文件操作。

- 遇到的具体问题：`gsv auth login` 提示 `Authenticated as europeanskyfab` 登录成功，但紧接着执行 `gsv chat`、`gsv auth token list` 等任何需要连接 Gateway 的命令，都报 `GatewayRpcError { call: "sys.connect", code: 401, message: "Authentication failed" }`。

- 为什么这个问题值得记录：登录明明成功了，但后续命令仍然认证失败，说明“登录写入的凭证”和“命令读取的凭证”不是同一个来源。这类工具链内部凭证来源不一致的问题，是 GSV 这类多角色系统里的常见坑。

## 2. 排查过程

- 现象：
  - `gsv auth login` 输出 `Authenticated as europeanskyfab. Session cached until ...`，看起来登录成功。
  - 紧接着执行 `gsv chat`，报 `401 Authentication failed`。
  - 执行 `gsv auth token list`，同样报 `401`。

- 尝试过的方法（包括失败的）：
  1. 重新执行 `gsv auth login`，再次提示登录成功，但问题依旧。
  2. 检查 `gsv chat` 的连接地址，确认是 `wss://gsv.europeanskyfab.xyz/ws`，地址无误。
  3. 回顾之前部署 `gsvd` 时，曾把设备 Token（`gsv_machine_...`）写入 `~/.bashrc` 的 `GSV_TOKEN` 环境变量。

- 关键线索：
  - `gsv auth login` 生成的是 **user 角色**的会话缓存。
  - 但 `GSV_TOKEN` 环境变量里存放的是 **machine 角色**的设备 Token。
  - `gsv` CLI 在连接 Gateway 时，会优先读取环境变量中的 Token，而不是登录缓存的用户会话。
  - 用 machine Token 去执行需要 user 角色的命令，Gateway 直接返回 401。

- 根本原因：**`GSV_TOKEN` 环境变量中的设备 Token（machine 角色）覆盖了 `gsv auth login` 缓存的用户会话（user 角色），导致后续命令以错误的角色身份连接 Gateway，被拒绝认证。**

## 3. 解决方案

- 具体步骤：
  1. 临时清除 `GSV_TOKEN` 环境变量。
  2. 验证 `gsv auth` 系列命令是否恢复正常。
  3. 从 `~/.bashrc` 中永久删除该环境变量，避免重启后再次干扰。

- 关键命令/代码片段：
  ```bash
  # 1. 临时清除环境变量
  unset GSV_TOKEN

  # 2. 验证是否恢复正常
  gsv auth token list

  # 3. 永久删除：编辑 ~/.bashrc，删掉 export GSV_TOKEN=... 那一行
  nano ~/.bashrc
  # 删除对应行后保存，然后重新加载
  source ~/.bashrc
  ```

- 需要注意的坑：
  - `GSV_TOKEN` 环境变量的优先级高于 `gsv auth login` 缓存的用户会话，会静默覆盖用户身份。
  - machine 角色的 Token 只能用于设备驱动（`gsvd`），不能用于用户交互命令（`gsv chat`）。
  - 如果确实需要同时保留设备 Token 和用户会话，应该用不同的环境变量名区分，或者把设备 Token 写入 `gsvd` 自己的配置文件，而不是全局环境变量。

- **转折：意识到 401 不是“登录失败”，而是“用了错误的角色 Token 去连接”。**

## 4. 验证

- 验证方式：
  1. `gsv auth token list` 能正常列出 Token，不再报 401。
  2. `gsv chat` 能成功连接 Gateway，进入对话界面。
  3. 在对话中向 Agent 发送文件操作指令，能正常执行并返回结果。

- 结果：清除 `GSV_TOKEN` 环境变量后，`gsv` CLI 恢复以 user 角色连接 Gateway，认证成功，`gsv chat` 可以正常使用。

## 5. 总结与反思

- 这次学到的：
- 下次遇到类似问题，先检查什么：

## 6. 核心概念（可选）

1. **Token 认证**

   Token 是一种轻量级的身份凭证，本质上是一串加密字符串，用来证明“你是谁”。在 GSV 中，Token 分为不同角色：`user` 角色的 Token 用于用户交互（如 `gsv chat`），`machine` 角色的 Token 用于设备驱动（如 `gsvd`）。不同角色的 Token 权限不同，用错角色会被 Gateway 直接拒绝。Token 通常有有效期，过期后需要重新创建。

2. **GSV 的 Gateway**

   Gateway 是 GSV 架构中的“控制平面”，负责管理所有连接、认证和消息路由。无论是终端 CLI、网页端还是设备守护进程，所有客户端都必须先通过 Gateway 的认证，才能接入 GSV 网络。Gateway 会根据客户端提供的 Token 判断其角色和权限，认证失败时返回 401。你在终端、网页端遇到的连接问题，大多都和 Gateway 的认证环节有关。