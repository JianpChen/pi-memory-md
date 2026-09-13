---
description: 自研记忆插件 pi-memory-agent 的设计、命令与验证结果（已于 2026-09-13 卸载，保留作参考）
tags: [pi, 记忆插件, pi-memory-agent, 历史项目, 多智能体]
created: 2026-09-13
---

# pi-memory-agent 插件（历史项目，已卸载）

> ⚠️ **状态：已废弃**。2026-09-13 用户要求删除，插件源目录与 settings.json 注册项均已移除，
> 精确备份在 `D:/工作学习/pi的记忆研究/pi-memory-agent.zip`（28.4 KB，4 文件）。
> 现役方案是 `pi-memory-md`，见同目录 `pi-memory-md-现状.md`。

## 项目概况
- 位置：`D:/工作学习/pi的记忆研究/pi-memory-agent/`（版本 0.2.0）
- 技术选型：**纯 TypeScript + Node 标准库，零 npm 依赖**
- 解决的问题（原版 pi-memory 的两个痛点）：
  1. **全局单例记忆**导致所有项目上下文相互污染 → 内置 `memory_agent` 工具，支持多独立记忆体（`list` / `create` / `switch` / `current` / `rename` / `remove` / `import`）
  2. `memory_search` 依赖 `qmd` 需联网下载 embedding 模型 → 改为内置**离线检索**

## 关键设计
- 记忆目录**每次动态解析**，支持运行中切换记忆体，无需重启
- 设 `PI_MEMORY_DIR` 环境变量则退化为单目录模式
- 首次运行自动把旧 `~/.pi/agent/memory` 复制为 `default` 记忆体（**无损**）
- 记忆体名校验拒绝 `../x` 这类路径穿越

## 快捷命令（用 `pi.registerCommand` 实现，复用现有 `memFile` / `forgetBlocks` / `writeRecoveryRecord`）

| 命令 | 作用 |
|---|---|
| `/memo <内容>` | 写长期记忆 |
| `/forget <关键词>` | 删除记忆，**可恢复**（写 recovery 记录） |
| `/magents` | 列出所有记忆体 |
| `/magent list\|create <名字>\|current\|<名字>` | 记忆体管理与切换 |
| `/memlast [memo\|both] [edit]` | 记忆**上一轮对话**，写成 `### 对话记忆 #N HH:MM` 区块；默认写当天 `daily/`，`memo` 写 `MEMORY.md`，`both` 两处都写，`edit` 先弹编辑器 |
| `/memturns [数量]` | 列出最近 N 轮对话（每条用户消息 = 一个任务，默认 10、上限 30）→ 选择一轮 → 再选写入位置 |
| `/memturns save <编号> [memo\|both]` | 无 TUI 环境（RPC / 微信）直接记忆第 N 轮 |
| `/memdate` | 列出有记录的日期供选择 |
| `/memdate today\|yesterday\|7d\|30d\|2026-09-08\|2026-09\|2026-09-01..2026-09-08\|all [关键词]` | 按日期/日期范围检索，可叠加关键词 |

## 实现要点
- **轮次提取**：从 `ctx.sessionManager.getBranch()` 取 user/assistant 文本与工具名；助手多条回复合并；工具名去重成 `_工具：bash、edit_`；单轮内容超限按 2000/4000 字符中间截断
- **日期来源**：`daily/YYYY-MM-DD.md` 文件名 + `MEMORY.md` / `SCRATCHPAD.md` 里每条记忆的 `<!-- 时间戳 [sid] -->`
- `splitStampEntries()`：按**时间戳行**切分（而非按空行），因此「一条记忆 = 一条结果」，不会出现「戳一行、正文一行」的碎片
- **长结果展示**：`/memdate` 结果 >1500 字符时用 `ctx.ui.editor` 打开全文（零新增依赖，只用 core UI API），>14K 字符自动截断
- 所有命令都带 `getArgumentCompletions` 参数补全

## 验证方式（可复用）
用 **jiti 加载真实源码 + 模拟 pi/ctx 对象**做离线实测，不启动 pi：
- 7 个命令、8 个工具全部注册成功
- 三种搜索模式命中、`forget` 恢复记录、记忆体名校验（拒绝 `../x`）均通过
- 写盘结果正确

## 无网机器移植
- 打包产物（放在 `D:/工作学习/pi的记忆研究/`）：
  - `pi-memory-agent.zip` — 改进版，28.4 KB / 4 文件，**多智能体 + 全离线搜索**
  - `pi-memory-original.zip` — 原版 pi-memory v0.4.2，34 KB / 6 文件，核心读写可用但 `memory_search` 需 qmd 联网
- 移植方式：目标机器上 `pi install ./解压目录`——**本地路径安装不跑 npm install**，零联网
- 原版限制：`memory_search` 需 qmd 联网下载 embedding 模型；离线时只有 keyword 模式可用，semantic/deep 失败

## 本机切换验证记录
- 已卸载 `npm:pi-memory`、改为本地路径安装 `pi-memory-agent`
- 验证结果：semantic/deep 从「missing embeddings / 超时」变为**全部离线可用**
