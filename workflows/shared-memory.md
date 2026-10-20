# 跨 Agent 共享记忆

> 用途：让多个 agent 读写同一批"跨会话仍成立"的事实。
> 形态：一个 SQLite 库 + 一个 stdio JSON-RPC 服务（MCP 协议），**纯标准库零依赖**。

## 设计约束

建之前先明确消费方要什么。最稳的做法是**照现有只读方的 SQL 反推 schema**，
这样它们一行代码都不用改就能重新读到数据。

本例中两个只读消费方要求：

```sql
-- 图谱扫描器
SELECT id,namespace,memory_key,content,tags_json,source_agent,provenance,confidence,updated_at
FROM memories WHERE status='active' ORDER BY updated_at DESC

-- 同步器
SELECT id,memory_key,content FROM memories WHERE status='active' AND memory_key IS NOT NULL
```

并调用 `memory_add` / `memory_supersede` 两个写工具。

## Schema

```sql
CREATE TABLE IF NOT EXISTS memories (
    id TEXT PRIMARY KEY,
    namespace TEXT NOT NULL DEFAULT 'global',
    memory_key TEXT,            -- 可空；有值则全局唯一（active 状态下）
    content TEXT NOT NULL,
    tags_json TEXT NOT NULL DEFAULT '[]',
    source_agent TEXT,
    provenance TEXT,
    confidence REAL NOT NULL DEFAULT 1.0,
    status TEXT NOT NULL DEFAULT 'active',   -- active / deleted（软删除）
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL
);

CREATE INDEX idx_memories_key ON memories(memory_key) WHERE memory_key IS NOT NULL;
CREATE INDEX idx_memories_status ON memories(status);
CREATE INDEX idx_memories_updated ON memories(updated_at);
```

## 六个工具

| 工具 | 作用 | 关键行为 |
| --- | --- | --- |
| `memory_add` | 新增 | **同 key 已 active 时拒绝**（防竞争副本） |
| `memory_supersede` | 更新 | 按 id 或 key；改内容不改 key；保留审计链 |
| `memory_get` | 读 | 按 id 或 key |
| `memory_list` | 列 | active，可按 namespace |
| `memory_delete` | 删 | 软删除（status=deleted），不物理删 |
| `memory_search` | 搜 | content / memory_key |

## 写入门控（最重要的一条）

```python
if key:
    dup = c.execute("SELECT id FROM memories WHERE memory_key=? AND status='active'", (key,)).fetchone()
    if dup:
        raise ValueError("memory_key already active: " + key)
```

这一条把"悄悄新建一条矛盾的"变成了"必须显式替换旧的"，替换时还强制留下原因。

## 已知坑（实测，按隐蔽程度排序）

### 1. `PYTHONHOME` 污染 —— 最难定位

**现象**：调用方报 `JSONDecodeError: Expecting value: line 1 column 1`，看起来像协议问题。

**真相**：调用方跑在虚拟环境里，其环境带 `PYTHONHOME` / `UV_INTERNAL__PYTHONHOME` 指向某个
Python 版本。它 spawn 本服务（另一个 Python 版本）时，该变量泄漏进子进程，
**覆盖子进程的 site-packages 解析**，导致模块路径文件被忽略 -> `ModuleNotFoundError` ->
stdout 输出空行 -> 调用方解析 JSON 失败。

**为什么难**：直接手动跑 `python.exe` **完全正常**，只在"从虚拟环境内 spawn"时出现。

**修法**：加一个启动器，先清变量再启动。

```bat
@echo off
set "PYTHONHOME="
set "UV_INTERNAL__PYTHONHOME="
set "PYTHONUTF8=1"
"<python.exe>" -X utf8 -m your_module.server %*
```

### 2. 别用 `Path.home()`

某些宿主（agent 沙箱）不设 `HOME`/`USERPROFILE`，此时 `Path.home()` 抛 `RuntimeError`。
改用 `LOCALAPPDATA` -> `USERPROFILE` -> `SystemDrive` 逐级回退。

### 3. `.pth` 可能不在你以为的 site-packages

用户级 Python 的 site-packages 就在**安装根目录**，不是 `Lib\site-packages`。
用 `site.getsitepackages()[0]` **现场取**，不要硬编码。

### 4. `executescript` 的 SQL 必须带分号

不像 `execute` 会自动补；数组拼 SQL 时每行结尾都要显式 `;`，否则报 `near "CREATE": syntax error`。

### 5. Python 相邻字符串字面量会隐式拼接

```python
# 错：三段被拼成一个参数，而元组是第二个 -> "expected at most 2 arguments"
c.execute(
    "UPDATE t SET a=?,",
    " b=? WHERE id=?",
    (x, y, ident),
)

# 对：用逗号连接成一个字符串
c.execute("UPDATE t SET a=?, b=? WHERE id=?", (x, y, ident))
```

### 6. 连接用 `isolation_level=None`

否则隐式 BEGIN 与 DDL 冲突，`executescript` 报错。

## 与图谱工具联动

- 图谱扫描器把本库的 active 记忆导入为"记忆节点" + `describes` 边。
- 反向：把双链笔记里标记 `share_memory: true` 的条目 supersede 进本库。
- 形成 **笔记（人读）<-> 共享记忆（agent 读）** 的双向链路，两侧都不手改对方。

## 纪律

1. 只写"跨会话仍成立"的事实/约束/决定/方法。
2. **不写**凭据、原始聊天、真实姓名、涉密原文。
3. key 命名：`native.<agent>.<project>.<topic>` / `<agent>.curated.<topic>.v<n>`。
4. namespace：跨项目用 `global`，项目内用 `project:<slug>`。
5. 冲突时以**证件/原件/最新显式指令**为准，然后 supersede 并写清依据。
6. **自动映射可疑时，宁可降级为"待核对"标记，也不要留错内容。** 正确内容稍后按源头补。
