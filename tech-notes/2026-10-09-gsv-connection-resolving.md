# 标题：修复 GSV 云端安装身份失效导致的 WebSocket 连接失败
# 涉及层次：环境、配置、代码

## 1. 背景

- 目标：恢复 GSV 的网页端和设备端连接，让 Agent 能正常使用。
- 遇到的具体问题：
  1. 主机和笔记本的 GSV 网页端同时弹出 `WebSocket connection failed`。
  2. Cloudflare Worker 日志显示 `[Gateway] Kernel installation identity check failed`。
  3. 尝试用 `deploy.gsv.space` 的 UPDATE 修复，卡在 WhatsApp 通道的 Durable Object 迁移报错。
  4. DELETE 后重新部署，Worker 报 `Error 1101`，日志显示 `Installation directory is not configured`。
  5. 设备注册时 gsvd 反复报 401 `Authentication failed`。
- 为什么这个问题值得记录：这是 GSV 从“云端身份失效”到“部署器漏组件”再到“设备 token 不一致”的完整故障链。排查过程中发现了 GSV 最新版部署器的 Bug，以及 `gsv daemon install` 缓存旧配置的行为。这些经验以后遇到同类问题可以直接复用。

## 2. 排查过程

- 现象：
  - 网页端报 `WebSocket connection failed`，控制台显示 `wss://europeanskyfab.xyz/ws` 握手返回 503。
  - Worker 日志：`[Gateway] Kernel installation identity check failed`。
  - 重新部署后：`Error 1101`，日志 `Installation directory is not configured`。
  - gsvd 日志：`sys.connect failed (code 401): Authentication failed`。
- 尝试过的方法（包括失败的）：
  1. 用无痕模式访问网页端 → 无效，排除浏览器缓存问题。
  2. 检查笔记本网络环境 → `curl` 和 `nslookup` 都正常，排除网络问题。
  3. 用 `deploy.gsv.space` 的 UPDATE 修复 → 卡在 `channel-whatsapp` 的 Durable Object 迁移报错。
  4. DELETE 后重新部署 → 域名 DNS 记录被清空，Worker 报 `Error 1101`。
  5. 手动绑定子域名 `gsv.europeanskyfab.xyz` → 域名能解析，但 Worker 仍报 1101。
  6. 查 GitHub 源码 → 确认 `INSTALLATION_DIRECTORY` 绑定的是 `gsv-installations` 服务。
  7. 检查 Workers 列表 → 发现 `gsv-installations` 不存在。
  8. 降级到 v0.5.0 重新部署 → `gsv-installations` 出现，Web 端能登录。
  9. 设备注册 → CLI 报 503 `Installation unavailable`。
  10. 通过 Web 端 `CONNECT MACHINE` 生成 token → gsvd 报 401 `Authentication failed`。
  11. 检查 `~/.config/gsv/config.toml` → token 是对的，但 gsvd 服务读的是旧缓存。
  12. `gsv daemon uninstall` + `gsv daemon install` → 终于连上。
- 关键线索：
  1. Worker 日志里的 `Installation directory is not configured`，直接指向缺失的绑定。
  2. GitHub 源码里 `{ "binding": "INSTALLATION_DIRECTORY", "service": "gsv-installations" }`，确认了组件名。
  3. Workers 列表里没有 `gsv-installations`，确认部署器漏部署。
  4. 配置文件里的 token 是对的，但 gsvd 报 401，说明服务没读到最新配置。
- 根本原因：
  1. **GSV 最新版部署器（v0.6.5 / v0.6.6）漏部署了 `gsv-installations` 组件**，导致 Gateway 找不到安装目录，身份校验失败。
  2. **DELETE 操作清空了域名下的 DNS 记录**，导致子域名解析失败。
  3. **`gsv daemon install` 缓存了旧的设备 token**，改了 CLI 配置后，gsvd 服务没有重新读取。

## 3. 解决方案

- 具体步骤：
  1. **降级部署**：在 `deploy.gsv.space` 的 Advanced options 里选 **v0.5.0** 重新部署，绕开最新版部署器的 Bug。
  2. **补绑子域名**：在 Cloudflare 的 `gsv` Worker → 域 → Add Custom Domain，绑定 `gsv.europeanskyfab.xyz`。
  3. **Web 端登录并初始化**：打开 `https://gsv.europeanskyfab.xyz`，完成首次用户设置。
  4. **Web 端生成设备 token**：在 `CONNECT MACHINE` 页面走完流程，拿到新的 token。
  5. **WSL 里配置并注册设备**：
     ```bash
     gsv config --local set gateway.url "wss://europeanskyfab.xyz/ws"
     gsv config --local set gateway.username "europeanskyfab"
     gsv config --local set node.token "<新token>"
     gsv daemon install --id "esf_computer" --workspace "$HOME"
     ```
  6. **如果 gsvd 报 401，卸载重装**：
     ```bash
     gsv daemon uninstall
     gsv daemon install --id "esf_computer" --workspace "$HOME"
     ```
  7. **验证连接**：
     ```bash
     gsv daemon status
     gsv chat "hello"
     ```
- 关键命令/代码片段：

降级部署后检查 `gsv-installations` 是否存在（Cloudflare Workers 列表里看）。

修复 gsvd 401 的关键命令：

```bash
gsv daemon uninstall
gsv daemon install --id "esf_computer" --workspace "$HOME"
gsv daemon status
```

GitHub 源码里的关键绑定定义：

```json
{ "binding": "INSTALLATION_DIRECTORY", "service": "gsv-installations" }
```

- 需要注意的坑：
  1. **GSV 的 Web 部署器不可靠**：最新版漏部署 `gsv-installations`，降级到 v0.5.0 才能正常部署。部署后先检查 Workers 列表有没有这个组件。
  2. **DELETE 会连带删除 DNS 记录**：删除 GSV 安装后，记得检查域名下的 DNS 记录，手动补回子域名绑定。
  3. **`gsv config set` 和 `gsv daemon install` 读的配置可能不一致**：改了 CLI 配置后，gsvd 服务可能还读旧值。改完配置要 `daemon uninstall` + `daemon install`。
  4. **WSL 里用 `gsv`，PowerShell 里用 `gsv.exe`**：不要混用。gsvd 跑在 WSL 里，CLI 也应该在 WSL 里装。
  5. **设备 id 要和 Web 端注册时一致**：Web 端显示 `esf_computer`，CLI 里也用 `esf_computer`，不要写成 `esf-computer`。
- **转折：从“域名问题”到“部署器漏组件”
  一开始所有报错都指向“WebSocket 连接失败”和“域名解析失败”，我花了大量时间在 DNS、hosts、浏览器缓存上。直到发现 Worker 日志里的 `Installation directory is not configured`，去 GitHub 查源码，才确认根因是 `gsv-installations` 组件没被部署。这个转折点把排查方向从“网络/域名”彻底转向了“部署器 Bug”。

## 4. 验证

- 验证方式：
  1. `gsv daemon status` 显示 `connected: yes`。
  2. `gsv chat "hello"` 收到 Agent 回复。
  3. Web 端的 `MACHINES` 区域出现 `esf_computer`。
  4. 网页端能正常对话，Agent 状态显示 `RUNNING`。
- 结果：
  - gsvd 状态正常，不再报 401。
  - CLI 能正常收发消息。
  - Web 端设备列表显示已连接。
  - Agent 回复虽然稍慢，但功能正常。

## 5. 总结与反思

- 这次学到的：GSV这种新项目容易出现内部问题，在遇到棘手问题时应考虑更换版本
- 下次遇到类似问题，先检查什么：先分析是个人配置、网络问题，还是项目自身bug导致的问题

## 6. 核心概念


1. WebSocket

  WebSocket 是一种**全双工通信协议**，建立在 TCP 之上。它先通过一次 HTTP 请求完成“握手”（Upgrade），之后这条连接就变成持久的、双向的通道，双方可以随时互发消息。

  和普通 HTTP 的区别：

  - **HTTP**：客户端问一次，服务器答一次，连接就断了。服务器不能主动推消息。
  - **WebSocket**：一次握手，长期保持。服务器可以主动推消息给客户端。

  GSV 网页端用的就是 WebSocket。浏览器打开页面后，会和 Cloudflare 上的 Gateway 建立一条 `wss://` 连接。之后 Agent 的回复、状态更新，都通过这条连接实时推送。所以一旦 WebSocket 握手失败（比如返回 503），整个页面就无法正常工作了。


2. 域名与子域名

  **域名**是互联网上的“门牌号”，比如 `europeanskyfab.xyz`。

  **子域名**是在主域名前面加前缀，比如 `gsv.europeanskyfab.xyz`。

  | 名称 | 例子 | 说明 |
  |:---|:---|:---|
  | 主域名（根域名） | `europeanskyfab.xyz` | 你注册的那个域名 |
  | 子域名 | `gsv.europeanskyfab.xyz` | 主域名下的一个分支 |

  在 Cloudflare 里，主域名和子域名是**两条独立的 DNS 记录**：

  - `europeanskyfab.xyz` → 一条记录（可能是 A 或 CNAME）
  - `gsv.europeanskyfab.xyz` → 另一条记录

  **它们不共享绑定关系。** 你把 Worker 绑到主域名上，子域名不会自动跟着绑。反之亦然。

  这也是这次问题的原因之一：DELETE GSV 安装时，主域名和子域名的 DNS 记录都被清空了。重新部署后只恢复了主域名，子域名需要**手动重新绑定**，否则解析不出来。
