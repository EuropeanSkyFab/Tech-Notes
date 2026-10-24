# 标题：为 N.E.K.O. 猫娘 Agent 接入自定义 MCP Server，实现文件操作能力

## 1. 背景

- 目标：让猫娘 Agent（N.E.K.O.）能够操作本机文件，实现读取、写入、列出目录等能力，作为 Agent 控制外部世界的第一步。
- 遇到的具体问题：
  1. 不清楚 MCP Server 到底是什么，也不知道从哪里入手。
  2. 最初尝试的 MCP Python SDK 示例代码基于 1.x 版本，本机安装的是 2.x，导入 FastMCP 直接报 ModuleNotFoundError。
  3. MCP Adapter 插件最初加载后显示“当前上下文没有暴露 add_server 动作，无法使用自动表单”，无法添加 Server。
  4. 导入 Server 配置后一直卡在 Connecting...，无法完成握手。
- 为什么这个问题值得记录：这是第一次真正打通“猫娘 Agent → MCP 协议 → 自定义工具”的完整链路。MCP 是 Agent 连接外部能力的标准协议，跑通一次之后，后续接入任何硬件、服务都只是换工具函数，不用再重走一遍配置流程。

## 2. 排查过程

- 现象：
  - 代码报错：ModuleNotFoundError: No module named 'mcp.server.fastmcp'，提示这是 mcp 2.x，FastMCP 已改名为 MCPServer。
  - 插件面板：add_server 动作不可用，表单和 JSON 导入按钮均为灰色。
  - 日志显示：Timeout connecting to MCP server 'example'，导入的示例 Server 一直连不上。
- 尝试过的方法（包括失败的）：
  1. 直接按 1.x 教程写代码，导入 FastMCP，失败。
  2. 考虑降级到 mcp<2，但决定改用 2.x 的新写法，保持依赖最新。
  3. 重启插件、重载插件，add_server 动作一开始不可用，后来重载后恢复正常。
  4. 导入教程里的示例配置（example、remote-api），一直卡在 Connecting...。
  5. 把 command 从 python 改成 Python 完整路径，重新导入，依然报 already exists with different config。
- 关键线索：
  1. 报错信息直接指出 mcp 2.x 的改名问题，并附带了迁移指南链接。
  2. 2.x 的迁移指南明确：FastMCP 改名为 MCPServer，导入路径变成 mcp.server.mcpserver。
  3. 日志里出现 Saving mcp_servers config: ['file-server'] 和 Connected to MCP server 'file-server' with 3 tools，说明 Server 本身能连上。
  4. 手动在终端跑 python E:\NEKO\MCP_Server.py 不报错、能挂着，说明 Server 本身没问题。
- 根本原因：
  1. mcp 包版本是 2.x，API 已从 FastMCP 改为 MCPServer，导入路径也变了。解决方式是改用 2.x 的新写法：from mcp.server.mcpserver import MCPServer。
  2. 最初 add_server 不可用是插件加载时动作未注册，重载插件后恢复。
  3. 卡在 Connecting... 是因为导入的是教程示例，不是真实可用的 Server。导入真实的 file-server 配置后，日志显示连接成功并发现了 3 个工具。

## 3. 解决方案

- 具体步骤：
  1. 编写 2.x 版本的 MCP Server：创建 E:\NEKO\MCP_Server.py，用 MCPServer 定义三个工具：read_file、write_file、list_files，工作区限定在 ~/mcp_workspace，并通过 _safe_path 做路径越权检查。
  2. 手动验证 Server：在终端执行 python E:\NEKO\MCP_Server.py，确认不报错、能挂着等待输入。
  3. 重载 MCP Adapter 插件：在 N.E.K.O. 插件面板点“停止”再点“重载”，让 add_server 动作恢复可用。
  4. 导入 Server 配置：在 MCP Adapter 的 JSON 导入框粘贴配置，勾选“导入后连接”，点“导入 JSON”。
  5. 刷新状态：点“刷新状态”，确认 Server 已连接、Tools 已发现。
- 关键命令/代码片段：

python E:\NEKO\MCP_Server.py

MCP Server 核心代码（2.x 写法）：

```bash

from mcp.server.mcpserver import MCPServer
import os

mcp = MCPServer("file-server")
WORKSPACE = os.path.expanduser("~/mcp_workspace")
os.makedirs(WORKSPACE, exist_ok=True)

def _safe_path(relative_path: str) -> str:
    full = os.path.abspath(os.path.join(WORKSPACE, relative_path))
    if not full.startswith(os.path.abspath(WORKSPACE)):
        raise ValueError("路径越权，只允许操作工作区内的文件")
    return full

@mcp.tool()
def read_file(path: str) -> str:
    """读取工作区内指定文件的内容。"""
    with open(_safe_path(path), "r", encoding="utf-8") as f:
        return f.read()

@mcp.tool()
def write_file(path: str, content: str) -> str:
    """向工作区内指定文件写入内容。"""
    full = _safe_path(path)
    os.makedirs(os.path.dirname(full), exist_ok=True)
    with open(full, "w", encoding="utf-8") as f:
        f.write(content)
    return f"已写入 {len(content)} 个字符到 {path}"

@mcp.tool()
def list_files(dir: str = ".") -> str:
    """列出工作区内某个目录下的所有文件和文件夹。"""
    entries = os.listdir(_safe_path(dir))
    return "\n".join(entries) if entries else "(空目录)"

if __name__ == "__main__":
    mcp.run()
```

JSON 导入配置：

```bash
{
  "name": "file-server",
  "transport": "stdio",
  "command": "python",
  "args": ["E:/NEKO/MCP_Server.py"],
  "enabled": true,
  "auto_connect": true
}
```

- 需要注意的坑：
  1. mcp 2.x 的导入路径是 mcp.server.mcpserver，类名是 MCPServer，不是 1.x 的 mcp.server.fastmcp.FastMCP。
  2. stdio 模式不能乱打印：stdout 是 MCP 协议的通信线路，代码里不能用 print() 输出调试信息，否则会破坏 JSON-RPC 消息。
  3. add_server 动作不可用：插件加载时动作未注册，重载插件后恢复。
  4. 重复导入报错：Server 'file-server' already exists with different config 不是错误，只是配置已存在，不覆盖。
  5. 工作区安全：_safe_path 防止 Agent 越权读写工作区外的文件，不要直接把 WORKSPACE 设成根目录。

## 4. 验证

- 验证方式：
  1. 在 MCP Adapter 面板确认：已连接 Server = 1，已发现 Tools = 3，Error 列为空。
  2. 在猫娘对话里说“帮我看看工作区里有哪些文件”，观察她是否调用 list_files。
  3. 说“在工作区建一个 note.txt，写入 hello world”，再让她“读一下 note.txt”，验证写入和读取。
- 结果：
  - MCP Adapter 面板显示：已连接 Server = 1，已发现 Tools = 3，工具名 read_file, write_file, list_files，Connected = 是，Error 列为空。
  - Gateway 在线，连接率 100%。

## 5. 总结与反思

- 这次学到的：MCP Server的本质与基本配置思路
- 下次遇到类似问题，先检查什么：MCP版本
