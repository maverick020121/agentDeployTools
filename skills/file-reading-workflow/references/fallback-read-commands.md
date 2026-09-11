# binary 误报兜底链 — 命令模板

## 层级 1：MCP read_text_file

```
mcp__fileSystem__read_text_file(path=<绝对路径>, head=200)        # 前 N 行
mcp__fileSystem__read_text_file(path=<绝对路径>, tail=80)         # 末 N 行
mcp__fileSystem__read_multiple_files(paths=[...])                 # 批量对比
```

- 首次调用偶发同样报 binary（嗅探不稳定，schema_eval.sql 实测）→ **原样重试一次**再降级。
- 只允许白名单根 `D:\AIagent` 之下。

## 层级 2：PowerShell 按行号取区间

```powershell
$p = 'D:\AIagent\task_manage\data.html'
$lines = [IO.File]::ReadAllLines($p, [Text.Encoding]::UTF8)
$lines[579..740]        # read_file 行号 580..741（下标 = 行号-1）
$lazy = Select-String -Path $p -Pattern 'def save_|class '   # 内容搜索兜底
```

- `ReadAllLines` 是 `[IO.File]` 静态方法（返回 string[]），不是 List 的成员（实测报错）。
- 只读定位用 PS；**写操作（含中文/引号载荷）一律交 Python 脚本**（见 nonascii-file-editing）。

## 层级 3：临时 Python 脚本（内容+行号+语法三合一）

write_file 落盘 `%TEMP%\hermes-read-*.py`（显式 UTF-8），执行后删除：

```python
import io, ast, sys
p = r"D:\AIagent\task_manage\data.html"
raw = open(p, "rb").read()
has_bom = raw.startswith(b"\xef\xbb\xbf")
text = raw.decode("utf-8-sig")
lines = text.split("\n")            # 通用拆行，勿假设 CRLF
for i, l in enumerate(lines):
    if "关键词" in l:
        print(i + 1, "|", l[:200])
# .py 文件顺带验语法（BOM 已剥，否则报 U+FEFF）
try:
    ast.parse(text)
    print("AST OK")
except SyntaxError as e:
    print("AST FAIL", e.lineno, e.msg)
print("CRLF:", raw.count(b"\r\n"), "bareLF:", raw.count(b"\n") - raw.count(b"\r\n"))
```

执行（venv python 全路径、不带管道，见 windows-python-venv / nonascii-file-editing 的调用坑）：

```powershell
& "D:\tool\.venv312\Scripts\python.exe" "C:\Users\SR003122\AppData\Local\Temp\hermes-read-check.py"
```

输出同时给出：真实行号、行尾风格统计、语法状态——一次调用替代"读+嗅探+验证"三步。

## 层级 0 预防：codegraph（有索引时）

```powershell
Test-Path D:\AIagent\task_manage\.codegraph   # 无此目录 → 完全跳过 codegraph
codegraph init                                # 建索引由用户决定，勿擅自执行
```
