---
name: file-reading-workflow
description: 在本机（Windows + Hermes Desktop）环境读取/搜索/定位代码文件的工具选择与兜底链。何时用：想看文件内容、找某符号在哪、问"X 怎么实现/谁调用"、read_file 误报 "Binary file cannot display"、search_files 莫名 0 命中、需按行号定位大文件（100KB+）。触发词：读文件、查代码、找函数、binary 误报、搜索0命中、分段读。编辑侧规则不在本技能（见 nonascii-file-editing）。
---

# 代码读取工作流 — 工具选择与兜底链（Windows + Hermes）

## 三条读取通道

| 通道 | 实际执行者 | 用途定位 |
|---|---|---|
| 原生 `read_file` / `search_files` | Hermes Python 运行时 + ripgrep | 日常默认（行号分页 / 正则内容搜索） |
| MCP `fileSystem`（`@modelcontextprotocol/server-filesystem`，Node/npx，白名单根 `D:\AIagent`） | 独立 MCP 进程 | binary 误报兜底；`read_multiple_files` 批量对比 |
| MCP `codegraph_explore`（codegraph CLI，读 `.codegraph/` 索引） | 独立 MCP 进程 | 理解型问题（调用链/影响面/在哪实现） |

## 决策树（先分类再选工具）

1. **理解型**（"X 怎么实现 / 谁调用它 / 改它影响哪里"）
   - 仓库根存在 `.codegraph/` → **先一次 `codegraph_explore`**：返回相关符号逐字源码+调用路径，返回即视为已 Read，不要再重复打开那些文件；细节与 CLI 等价入口见 `codegraph` 技能。
   - 无索引（如 task_manage）→ `search_files` 定位符号行号 → `read_file` offset 分段读。
2. **定位型**（"字符串 X 在哪一行/哪个文件"）
   - `search_files(pattern, target='content', path=<workspace 绝对路径>)` — **path 永远显式传**（不传按 Hermes 运行时 cwd 搜，实测误命中 2538 个 Hermes 自带文件）。
3. **直读**：`read_file`（带行号，offset/limit ≤500 行/页，.ipynb/.docx/.xlsx 自动抽取）；多文件对比用 `mcp__fileSystem__read_multiple_files`。
4. **禁做**：terminal `Get-Content`/`cat`/`type` 读文件（慢、截断、中文乱码，终端只留给 git/构建/测试）；把 100KB+ 文件整读进上下文。

## binary 误报兜底链（本机核心坑）

`read_file` 会对 **UTF-8 无 BOM 的含中文 .py/.html/.sql/.md 误报 `is_binary=true`**（实测中招：db.py、data.html、plan.md、app.py、schema_eval.sql），且 **`search_files` 内容搜索对同一批文件静默跳过 → 0 命中**。推论：**"搜不到" ≠ "文件里没有"**，凭 0 命中下"不存在"的结论前必须换通道验证。

按序降级，哪个先通停哪个（命令模板见 references/fallback-read-commands.md）：

1. `mcp__fileSystem__read_text_file`（带 head/tail；首次同样报 binary 是嗅探不稳定，**重试一次**即正常）
2. PowerShell `[IO.File]::ReadAllLines($p,[Text.Encoding]::UTF8)` 按行号取区间
3. 临时 Python 脚本 `open(p,'rb').read()` + `decode('utf-8-sig')` 一次拿到 内容+行号+`ast.parse` 语法 三件事
- 判别纪律：**工具报 binary / 搜索 0 命中 → 第一怀疑对象是工具嗅探，不是文件/代码**。

## 读后用作锚点的验证

- 读取结果将作为编辑锚点时：动手前**重读现值**（用户手改 / 并行会话改动常态，patch 报 not found 先重读别凭记忆重试）。
- `read_file` 显示行号 N = PowerShell 数组下标 **N-1**（0 基，差 1 覆写错行毁续行 import）。
- `read_file` 输出里个别行显示 `\r` 不代表全文件 CRLF，换行风格以字节统计为准。

## 编辑前置（指针，不在本技能展开）

- 编辑任何文件前先字节级检测 EOL/BOM（探测脚本：nonascii-file-editing 的 `scripts/eol_check.py`）；task_manage 内 EOL 按文件分化：plan.md/app.py=CRLF、db.py/tests=LF、README=LF。
- patch 工具在 CRLF/中文文件上的破坏形态与逐条兜底 → 全部在 `nonascii-file-editing` 技能，本技能不重复。

## Pitfalls 速查

- search_files 不传 path → 搜到 Hermes 运行时目录（已两次实锤）。
- search_files 对 binary 误报文件 0 命中 → 误判"无引用"。
- MCP fileSystem 受白名单限制，`D:\AIagent` 之外的路径会被拒。
- 大文件（data.html 300KB+ 内联 JS 型）→ 配合 `single-file-html-apps` / `webapp-restore-verify` 技能。
