name: codegraph
description: "Use when exploring/locating/understanding code in repos that have a `.codegraph/` directory (pre-indexed knowledge graph: call paths, blast radius, verbatim source). Triggers: 找代码、调用链、谁调用了、影响面、codegraph、explore."
version: 1.0.0
author: installed from colbymchenry/codegraph v1.6.0
license: MIT
platforms: [windows]
metadata:
  hermes:
    tags: [codegraph, code-search, mcp, cli]


CodeGraph — 预索引代码知识图谱

来源仓库：https://github.com/colbymchenry/codegraph （npm 包 @colbymchenry/codegraph，本机 v1.6.0，2026-09-03 安装）。

何时用

仓库根目录存在 .codegraph/ 时，理解/定位代码优先用它而不是 grep/find 或盲目读文件：一次调用返回相关符号的逐字源码 + 它们之间的调用路径（含 grep 跟不了的动态派发跳转），省 token 省工具调用。

- 回答"这个函数被谁调用 / 它调用了谁"、"改动 X 会影响哪些地方"、"Y 功能在哪个文件实现"类问题。
- name: codegraph 需要跨仓索引查询：调用链、dead code、类型层级等。
- 没有 .codegraph/ 目录 → 完全跳过 CodeGraph（是否索引是用户决定的事）。

两种入口

1. MCP 工具（本会话已配置）：mcp_codegraph_explore — 语义与 CLI 相同，优先用。
2. CLI（始终可用）：codegraph explore "<符号名或问题>"，在仓库目录内执行。

常用命令

    codegraph version          # 查看版本
    codegraph init             # 在当前项目建索引（一次；之后文件变更自动同步）
    codegraph explore "符号名"  # 查询（MCP 工具不可用时的等价入口）
    codegraph upgrade          # 升级（自动识别 npm/bundle 安装方式）
    codegraph uninstall --keep-cli   # 仅移除各 agent 配置，保留 CLI

维护与配置现状

- MCP server 已写入 $HERMES_HOME/config.yaml 的 mcp_servers.codegraph（codegraph serve --mcp），另有 platform_toolsets.cli 含 mcp-codegraph。
- MCP 工具变更需新会话生效（会话启动时连接发现）。

常见陷阱（Windows 实测）

- npm 装的是 codegraph.CMD shim（%APPDATA%\npm\codegraph.CMD）。Python subprocess 用裸名 "codegraph" spawn 会 WinError 2；Hermes 自身 MCP 启动器能解析 .CMD（fileSystem server 的 npx 同理），自己写探针脚本时用 shutil.which("codegraph") 的完整路径。
- PS 管道结尾报 exit_code=4294967295/-1：codegraph explore ... | Select-Object 输出正确但退出码不可信（PS 5.1/管道老毛病），以输出内容判断成败。
- 安装器写 config.yaml 的位置难看但语义正确：codegraph 子块被插到文件尾部注释区之后（仍解析进 mcp_servers，PyYAML 已验证）。
- 索引是每项目一份（.codegraph/），全局 install 一次即可，init 每项目跑一次。

验证记录（2026-09-03）

1. codegraph version → 1.6.0 ✓
2. codegraph install --target=hermes --yes → 写入 config.yaml ✓（PyYAML 解析确认 codegraph ∈ mcp_servers）
3. stdio MCP 握手探针（initialize + tools/list）→ server codegraph 1.6.0，工具 codegraph_explore ✓
4. realTimeMonitoring codegraph init → 30 文件 / 494 节点 / 1486 边 ✓
5. codegraph explore "run_check_isolated" → 返回逐字源码 + blast radius ✓
