# 可复现环境与后台任务

> 用途：让 agent 跑长任务不超时、让环境可重建、让"成功"可验证。

## 长任务必须后台化

任何预计超过 1 分钟的命令，**不要在前台同步等**。agent 调用有超时上限，
被 kill 会留下半成品状态（例如数据库里一行 `status=running` 的僵尸记录）。

```powershell
$p = Start-Process -FilePath $Python `
    -ArgumentList @("-X","utf8","-m","your.module","scan") `
    -WorkingDirectory $Root -WindowStyle Hidden `
    -RedirectStandardOutput $Log -RedirectStandardError ($Log + ".err") `
    -PassThru
$p.Id | Set-Content "$Rootdataun.pid"
```

### 验证结果要看权威源，不看进程

进程没了不等于成功。**查数据库/文件系统**：

```sql
SELECT id,started_at,finished_at,status FROM scan_runs ORDER BY started_at DESC LIMIT 3;
```

### PowerShell 重定向会吞退出码

`*>` 合并 stdout/stderr 时 `$LASTEXITCODE` 判断可能失误，**误报失败**（实际成功）。
分开重定向，或直接查权威源，别只信退出码。

## 环境纪律

| 规则 | 理由 |
| --- | --- |
| **禁止装全局 Python** | 污染系统状态，且 agent 之间互相破坏 |
| 有 `.venv` 用现有的，没有则 `uv venv` | 保持隔离 |
| 始终显式调用解释器路径 | 不依赖激活状态，脚本才可复现 |
| 依赖记 `requirements.txt` | 一次性调试安装不必记 |
| stdlib-only 的脚本不需要 venv | 减少无谓开销 |
| 全程显式 UTF-8（`PYTHONUTF8=1` / `-X utf8`） | Windows 默认编码会毁掉中文 |

### Windows 特有

- PowerShell 5.1 执行含中文的 `.ps1` 会按 GBK 解析导致乱码/失败 -> **中文批处理改用 Python**。
- Harness 里 PowerShell 单行给**变量赋值可能被剥离**（`$x = "..."` 整段丢失）->
  把完整路径字面量写进命令参数，不要用变量拼目标。
- `Move-Item` 对超长路径**静默失败** -> 用 `robocopy /E /MOV`。
- 递归体积统计在部分卷上会超时 -> `-Depth 2` + 逐目录 + 结果增量落盘。
- MSYS 会把 `/Switch` 形式的参数当 POSIX 路径改写 ->
  `MSYS_NO_PATHCONV=1` / `MSYS2_ARG_CONV_EXCL="*"` / `//Switch`；更稳：改用 PowerShell。

## 二进制溯源

每次装工具，记录五项，方便以后判断"这个 exe 从哪来、要不要留"：

```text
路径 / 版本 / 体积 / SHA-256 / Authenticode 验签结果
```

```powershell
(Get-FileHash $exe -Algorithm SHA256).Hash
Get-AuthenticodeSignature $exe | Select-Object Status,SignerCertificate
winget show <id>   # 与清单版本比对
```

## 配置订正

工具配置里的路径经常在换机/换账户后失效。排查方法：

1. 把配置里每个路径逐个 `Test-Path`，标出失效项。
2. 区分"源数据缺失"和"生成产物未生成"（见 `../method/memory-stack.md`）。
3. 备份原配置（带时间戳），再改。
4. 改完跑一次真实命令验证，不要只看配置内容。

## 自检清单

- [ ] 长任务后台化了吗？
- [ ] 结果查的是权威源（DB/文件系统），不是进程在不在？
- [ ] 没装全局 Python？解释器是显式路径？
- [ ] 新装工具有哈希和验签记录？
- [ ] 配置里的路径都 `Test-Path` 过了？
