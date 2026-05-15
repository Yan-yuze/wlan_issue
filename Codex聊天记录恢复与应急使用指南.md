# Codex 聊天记录“消失”问题排查与恢复指南

> 适用场景：Codex 软件重启后，项目名称还在，但项目下方显示“暂无对话”；本地旧会话文件仍然存在，但 Codex 软件左侧聊天列表无法正常显示历史记录。

---

## 一、问题现象

我曾在 PowerShell 中运行过类似命令：

```powershell
Get-Process Codex,codex -ErrorAction SilentlyContinue | Stop-Process
```

之后重启 Codex，发现：

- Codex 软件里以前做过的 project / 项目文件名还在；
- 但项目下面的历史聊天记录全部消失；
- 左侧聊天区域显示“暂无对话”；
- 通过本地目录检查，发现旧的会话文件仍然存在。

这说明：**聊天记录不一定真的丢了，更可能是 Codex 的本地会话索引或软件列表显示出问题。**

---

## 二、初步判断

这句命令本身不是删除命令，它只是强制结束 Codex 相关进程：

```powershell
Get-Process Codex,codex -ErrorAction SilentlyContinue | Stop-Process
```

问题可能出在：Codex 当时还在写入本地会话索引、数据库或状态文件，进程被强制结束后，导致：

- 会话文件还在；
- 项目目录还在；
- 但 Codex 软件的左侧历史列表无法正确读取；
- `resume --all` 或软件列表只显示部分会话，甚至不显示。

---

## 三、最重要原则

出现这种情况后，先不要做这些事：

- 不要卸载 Codex；
- 不要清缓存；
- 不要删除 `.codex` 文件夹；
- 不要手动修改 `state_5.sqlite`、`logs_2.sqlite`、`session_index.jsonl`；
- 不要继续用 `Stop-Process` 强杀 Codex；
- 不要在 Codex 正在生成、运行命令、改文件时直接关窗口。

第一步应该是：**先备份本地 Codex 数据。**

---

## 四、备份 Codex 本地数据

在 PowerShell 中运行：

```powershell
$stamp = Get-Date -Format "yyyyMMdd-HHmmss"
$backup = "$env:USERPROFILE\Desktop\codex-backup-$stamp"
New-Item -ItemType Directory -Force $backup | Out-Null

$items = [ordered]@{
  "UserDotCodex"       = "$env:USERPROFILE\.codex"
  "AppDataCodex"       = "$env:APPDATA\Codex"
  "LocalAppDataCodex"  = "$env:LOCALAPPDATA\Codex"
  "AppDataOpenAI"      = "$env:APPDATA\OpenAI"
  "LocalAppDataOpenAI" = "$env:LOCALAPPDATA\OpenAI"
}

foreach ($name in $items.Keys) {
  $p = $items[$name]
  if (Test-Path $p) {
    Copy-Item $p -Destination "$backup\$name" -Recurse -Force -ErrorAction SilentlyContinue
  }
}

Write-Host "备份完成：$backup"
```

备份完成后，桌面会出现类似文件夹：

```text
C:\Users\你的用户名\Desktop\codex-backup-日期时间
```

---

## 五、检查旧会话文件是否还在

运行：

```powershell
Get-ChildItem "$env:USERPROFILE\.codex\sessions" -Recurse -File -Filter "rollout-*.jsonl" |
  Sort-Object LastWriteTime -Descending |
  Select-Object LastWriteTime, Length, FullName |
  Format-List
```

如果能看到大量类似文件：

```text
C:\Users\你的用户名\.codex\sessions\2026\05\14\rollout-2026-05-14T22-47-34-019e26f5-0b9c-7e43-89fe-e14e42347e37.jsonl
```

说明旧聊天记录大概率还在。

重点看：

- `rollout-*.jsonl` 文件是否存在；
- 文件大小是否不为 0；
- 是否有多个日期的会话文件；
- 是否有几百 KB、几 MB 的文件。

文件越大，里面包含完整聊天上下文的可能性越高。

---

## 六、当前结论

如果旧的 `rollout-*.jsonl` 文件仍在，那么：

```text
聊天内容：大概率还在。
Codex 软件左侧列表：可能坏了。
一键恢复到软件左侧原样：目前不稳定，不保证。
```

也就是说，问题不是“聊天完全丢失”，而更像是：

```text
原始会话文件存在，但 Codex 软件的历史索引/列表没有正确显示它们。
```

---

## 七、恢复旧会话的最稳方法

目前最稳的方法是：**绕过 Codex 软件左侧列表，直接用会话 ID 恢复旧聊天。**

每个 `rollout-*.jsonl` 文件名里都包含一个会话 ID。

例如这个文件：

```text
rollout-2026-05-14T22-47-34-019e26f5-0b9c-7e43-89fe-e14e42347e37.jsonl
```

其中会话 ID 是：

```text
019e26f5-0b9c-7e43-89fe-e14e42347e37
```

可以运行：

```powershell
codex resume 019e26f5-0b9c-7e43-89fe-e14e42347e37
```

---

## 八、批量生成可恢复会话命令清单

为了不用手动复制每个 ID，可以运行下面命令，在桌面生成一个恢复清单：

```powershell
$recovery = "$env:USERPROFILE\Desktop\codex-resume-list.txt"

Get-ChildItem "$env:USERPROFILE\.codex\sessions" -Recurse -File -Filter "rollout-*.jsonl" |
  Sort-Object LastWriteTime -Descending |
  ForEach-Object {
    if ($_.BaseName -match 'rollout-\d{4}-\d{2}-\d{2}T\d{2}-\d{2}-\d{2}-(?<id>[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12})$') {
      "{0:yyyy-MM-dd HH:mm:ss}  {1,10} bytes  codex resume {2}" -f $_.LastWriteTime, $_.Length, $Matches.id
    }
  } | Set-Content -Encoding UTF8 $recovery

notepad $recovery
```

生成的文件是：

```text
Desktop\codex-resume-list.txt
```

里面会有很多类似命令：

```powershell
codex resume 019e26f5-0b9c-7e43-89fe-e14e42347e37
codex resume 019e26db-6958-70b3-8671-739496a05da0
codex resume 019e1142-b58b-7593-9db1-666233edb3eb
```

之后可以按时间和文件大小，优先恢复最重要的会话。

---

## 九、恢复时如何选择目录

执行：

```powershell
codex resume <会话ID>
```

之后，Codex 可能会问：

```text
1. Use session directory (...)
2. Use current directory (C:\WINDOWS\system32)
```

此时应选择：

```text
1. Use session directory
```

原因：

- `Use session directory` 是这条旧会话当时所在的项目目录；
- Codex 需要回到原项目目录，才能看到当时的文件、代码和上下文；
- 不要选 `Use current directory`，尤其是当前目录是 `C:\WINDOWS\system32` 时，否则会把会话恢复到错误目录。

通常光标默认停在第 1 项时，直接按 Enter 即可。

---

## 十、恢复成功后第一件事：生成 RECOVERY_SUMMARY.md

进入旧会话后，不要急着继续开发。

先让 Codex 把当前会话上下文固化成项目文件：

```text
请根据当前会话上下文，总结这个项目目前的进展、已修改的文件、关键决策、未完成任务和下一步计划，并在项目根目录生成 RECOVERY_SUMMARY.md。
```

这样做的目的：

- 即使 Codex 软件左侧聊天列表以后仍然不显示；
- 也可以通过 `RECOVERY_SUMMARY.md` 把旧会话的核心上下文保存到项目里；
- 以后开新聊天时，只要让 Codex 先阅读这个文件，就能继续工作。

---

## 十一、如何使用 RECOVERY_SUMMARY.md 继续新对话

如果旧聊天无法回到 Codex 软件左侧列表，可以这样继续项目：

1. 打开 Codex 软件；
2. 进入对应项目目录；
3. 新建一个聊天；
4. 第一句话发送：

```text
请先阅读项目根目录下的 RECOVERY_SUMMARY.md，不要急着改代码。读完后请结合当前项目文件，告诉我：
1. 这个项目目前做到哪一步了；
2. 已经完成了哪些功能；
3. 目前可能存在什么问题；
4. 接下来最应该做哪三件事。
```

等 Codex 理解后，再发：

```text
很好。请根据 RECOVERY_SUMMARY.md 和当前项目文件，继续完成“下一步任务”。在改动前先告诉我你准备修改哪些文件。
```

这样就可以用 `RECOVERY_SUMMARY.md` 作为“项目记忆文件”，让新聊天接上旧项目。

---

## 十二、以后每次结束前都更新 RECOVERY_SUMMARY.md

为了避免以后再次丢上下文，每次准备关闭 Codex 前，让它更新总结文件：

```text
请更新 RECOVERY_SUMMARY.md，记录本轮完成了什么、修改了哪些文件、遇到什么问题、下一次应该从哪里继续。
```

下一次重新进入项目时，直接发：

```text
请阅读 RECOVERY_SUMMARY.md，并从“下一次应该继续的任务”开始工作。
```

---

## 十三、Codex 软件如何正常退出

这里说的是 **Codex 桌面软件**，不是单纯的 CLI。

不要优先使用：

```powershell
Get-Process Codex,codex -ErrorAction SilentlyContinue | Stop-Process
```

这属于强制杀进程，容易导致本地索引或状态没有写完。

更稳的退出方式：

1. 先确认 Codex 没有正在运行任务、没有正在生成、没有正在改文件；
2. 在 Codex 软件里找真正的退出入口，例如：
   - `File -> Exit`
   - `Codex -> Quit`
   - 右下角系统托盘 Codex 图标右键 `Quit / Exit`
3. 退出后再检查后台进程：

```powershell
Get-Process Codex,codex -ErrorAction SilentlyContinue |
  Select-Object Id, ProcessName, Path, StartTime
```

如果没有输出，说明 Codex 已经完全退出。

如果仍然有进程残留，并且确认 Codex 没有在写文件、没有在执行任务，最后才考虑强制结束：

```powershell
Get-Process Codex,codex -ErrorAction SilentlyContinue |
  Stop-Process -Force
```

注意：强制结束只能作为最后手段。

---

## 十四、关于“能不能恢复到 Codex 软件左侧列表”

目前结论：

```text
可以恢复旧聊天内容：大概率可以。
可以继续旧项目：可以。
可以通过 RECOVERY_SUMMARY.md 接续上下文：可以。
能否把所有旧聊天重新显示到 Codex 软件左侧列表：不保证。
```

不要把“左侧列表不显示”等同于“聊天记录彻底丢失”。

如果本地还有：

```text
C:\Users\你的用户名\.codex\sessions\...\rollout-*.jsonl
```

那么旧聊天通常还有恢复价值。

---

## 十五、推荐的最终处理流程

建议按这个顺序处理：

1. 立刻备份 `.codex`、AppData、LocalAppData 里的 Codex/OpenAI 相关目录；
2. 检查 `rollout-*.jsonl` 是否还在；
3. 生成 `codex-resume-list.txt`；
4. 按时间和文件大小，优先恢复重要会话；
5. 每恢复一个重要会话，就让 Codex 生成该项目的 `RECOVERY_SUMMARY.md`；
6. 之后新聊天都先阅读 `RECOVERY_SUMMARY.md`；
7. 每次关闭前更新 `RECOVERY_SUMMARY.md`；
8. 尽量从软件菜单或托盘正常退出 Codex；
9. 避免再次用 `Stop-Process` 强杀正在运行的 Codex。

---

## 十六、一句话总结

这次问题的核心不是“项目文件没了”，而是：

```text
Codex 本地会话文件仍然存在，但软件左侧聊天列表/会话索引可能损坏或没有正确加载。
```

目前最稳的补救方式是：

```text
用 codex resume <会话ID> 逐个救回重要旧会话，
再让 Codex 在项目根目录生成 RECOVERY_SUMMARY.md，
以后用这个文件作为项目记忆来继续新对话。
```
