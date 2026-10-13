# 问题记录：GSV 网页端登录时弹出 WebSocket connection failed

## 基础信息
- **日期**：2026-10-02
- **项目阶段**：GSV 部署完成后，笔记本网页端登录测试
- **状态**：根因不明，暂时绕过
- **排查耗时**：约 2 小时（当日未解决）

## 现象描述

> **异常现象**：在笔记本上打开 `europeanskyfab.xyz` 登录 GSV 时，页面弹出 `WebSocket connection failed`，无法进入配置界面。
> **复现频率**：必现（笔记本 100%）

## 触发条件

- [x] 在笔记本上访问 GSV 网页端并尝试登录
- [x] 浏览器控制台报错：`WebSocket connection to 'wss://europeanskyfab.xyz/ws' failed: Error during WebSocket handshake: Unexpected response code: 503`
- [x] Cloudflare Worker 日志显示 `[Gateway] Kernel installation identity check failed`
- [x] 无痕模式下同样报错（排除浏览器缓存/Cookie 问题）
- [x] 用同一个账号在主机上可以正常登录，笔记本不行

## 排查过程

1. **检查笔记本浏览器扩展和隐私设置** → 结果：用无痕模式打开，问题依旧，排除扩展干扰。
2. **检查笔记本的网络环境** → 结果：网络正常，`curl -I https://gsv.space` 返回 200，`nslookup` 能解析出 IP。
3. **清除笔记本的浏览器站点数据** → 结果：问题依旧。
4. **检查 Cloudflare Worker 的 Observability 日志** → 结果：发现 `[Gateway] Kernel installation identity check failed`，说明笔记本的连接请求被 Gateway 拒绝。
5. **对比主机和笔记本的访问差异** → 结果：同一账号、同一时间，主机能登录，笔记本不能，说明问题与设备侧的凭据或网络路径有关。
6. **尝试用笔记本访问 `wss://europeanskyfab.xyz/ws`** → 结果：WebSocket 握手返回 503，和主机侧同一时间的请求表现不同。

## 当前假说

- **假说 A（高概率）**：笔记本浏览器持有的旧凭据（Session/Cookie）与云端安装身份不一致，导致 Gateway 拒绝 WebSocket 握手。
- **假说 B（中概率）**：笔记本所在的网络环境对 WebSocket 长连接有额外限制，导致握手失败。
- **假说 C（低概率）**：Cloudflare 边缘节点对笔记本的请求做了不同的调度，触发了身份校验的边界情况。

## 临时绕过方案

> 1. 在笔记本上**彻底清除 `europeanskyfab.xyz` 的站点数据**（不是无痕模式，是手动删 Cookie 和 Storage）。
> 2. 如果还不行，**改用主机访问 GSV 网页端**，笔记本暂时通过 CLI 或其他方式接入。
> 3. 后续如果云端安装身份再次变化，需要**所有设备同步重新登录**，避免凭据不一致。

## 复盘备注

> 暂无