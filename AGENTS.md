# AGENTS.md — 本项目协作规范

> 给 Codex / 其他 AI 助手。按这份规则工作，风格与之前的协作保持一致。

## 1. 项目是什么

嵌入式 / 秋招面试笔记仓库，同时是 Obsidian vault（`notes/`）。

- 主题：C 语言（volatile / static / 指针）、FreeRTOS 内核机制、Cortex-M 异常模型
- 源码依据：`D:\Users\86137\Desktop\2027秋招\FreeRTOS\Despacito\006\FreeRTOS`
  （`port.c` 移植层、`src/tasks.c`、`src/queue.c`）
- 目标读者：准备嵌入式岗位面试的本人

## 2. 硬性规则（都是踩过的坑）

1. **改已有文件前必须先读**。禁止不看内容直接 `write` 覆盖整个文件 —— 曾因此丢掉一整份笔记。
2. **只改 AI 自己新增的内容**。用户原有段落必须逐字节保留；改完要用 `Contains` 或 hash 证明原段落没变。
3. **大改前先备份到工作区**（不要备份进仓库），命名 `<文件名>_<原因>.bak`。
4. **写 Markdown 用 UTF-8 无 BOM**。PowerShell 5.1 下禁止用 `>` 或 `echo >` 写 `.md`（会写成 UTF-16，Obsidian 读不了）。
5. **不主动 `git add` / `commit` / `push`**，除非用户明确要求。
6. 中文文件名在 git 里会转义，需要 `git config --global core.quotepath false`。
7. 说明性内容优先放进对话；只有用户明确说"写进笔记"才改文件。

## 3. 写作风格

- **结论先行**，再讲机制，再给证据（源码行号 / 编译出的汇编 / 仿真数据），最后给面试话术。
- **表格优先**，少写长段落；每节尽量一张能扫完的表。
- 代码块必须标语言（```c / ```asm / ```text）。
- 表格后必须空一行；表头不允许空单元格；表格分隔线用 `| --- | --- |`。
- 压缩时优先删**例子和重复**，保留表格、关键代码、面试速答。
- 术语要精确：Cortex-M 优先级**数值越小越高**，别说"优先级低"这种歧义表述。
- 说错了要**主动承认并纠正**，说明错在哪、正确的是什么。
- 中文回答，代码注释用中文。

## 4. 改完 Markdown 必跑的结构校验

```powershell
$f = '<目标文件>'
$lines = Get-Content $f -Encoding UTF8
$fences = ($lines | Where-Object { $_ -match '^```' }).Count
"围栏 $fences 行，平衡 $(if ($fences % 2 -eq 0) {'✓'} else {'✗'})"
$bad = 0
for ($i = 0; $i -lt $lines.Count - 1; $i++) {
    if ($lines[$i] -match '^\|' -and $lines[$i+1] -match '^\|[\s\-:|]+\|$') {
        $c1 = ($lines[$i].Split('|').Count - 2); $c2 = ($lines[$i+1].Split('|').Count - 2)
        if ($c1 -ne $c2) { "行 $($i+1): $c1 vs $c2 ✗"; $bad++ }
        if ($lines[$i] -match '^\|\|') { "行 $($i+1): 空表头 ✗"; $bad++ }
    }
}
"表格校验: $(if ($bad -eq 0) {'✓'} else {'✗'})"
$b = [IO.File]::ReadAllBytes($f)[0..2]
"BOM: $(if ((($b | ForEach-Object { $_.ToString('X2') }) -join ' ') -eq 'EF BB BF') {'有 ✗'} else {'无 ✓'})"
```

## 5. 环境事实

| 项 | 值 |
| --- | --- |
| 系统 | Windows，PowerShell 5.1（**没有 `pwsh`**） |
| git | 2.46.2，`user.name=MyAngelMyrtle` |
| gh | 2.96.0，已登录 `MyAngelMyrtle` |
| 交叉编译器 | `arm-none-eabi-gcc` 13.3（`C:\msys64\mingw64\bin`） |
| 调试工具 | 有 `openocd`、`arm-none-eabi-gdb`；**没有 QEMU** |
| 网络 | GitHub 直连被干扰；代理 `127.0.0.1:7890`；git 已配代理；gh 需 `$env:HTTPS_PROXY` |

## 6. 笔记现状

| 文件 | 状态 |
| --- | --- |
| `notes/C语言基础.md` | **用户自己维护，不要改** |
| `notes/FREERTOS内核机制.md` | 主体，含 7 节（列表 / SysTick / 调度器 vs 临界区 / 任务创建 / PendSV / xPendedTicks / 同步原语） |
| `notes/Hardfault异常.md` | 空 |
| `notes/通信协议知识.md` | 空 |

**待补**：任务状态机、`vTaskDelay` vs `xTaskDelayUntil`、空闲任务与 `configIDLE_SHOULD_YIELD`。

## 7. 备份位置（工作区，不在仓库）

`D:\Users\86137\Documents\deepseek-harness\default-workspace\`

- `FREERTOS内核机制_粘贴原版.md.bak`（5229 B）
- `FREERTOS内核机制.md.bak`（18411 B，AI 生成的完整版）
- `FREERTOS内核机制_压缩前.md.bak`（30513 B，压缩前）
- `FREERTOS内核机制_成品455.md.bak`（18189 B，只压 AI 内容的版本）

## 8. 回答模板

用户问技术问题时，按这个顺序：

1. **结论**（一到两句，先给判断）
2. **机制**（分步或表格，讲清因果）
3. **证据**（源码行号 / 反汇编 / 实测数据；能在本机跑就跑）
4. **边界与坑**（什么时候不成立、常见误答）
5. **面试话术**（一段可以直接背的话）
6. 必要时问一句"要不要写进笔记"

对"我笔记里这段有问题吗"这类提问，先判断对错并说明理由，再提供修正文本。
