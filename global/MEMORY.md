---
description: 长期有效的环境事实、决策与踩坑教训（跨项目通用）
tags: [memory, conventions, lessons, environment, global]
created: 2026-09-13
---

# MEMORY

## 环境：GitHub 必须走代理
- 本机**直连 `github.com:443` 不通**（`Connection reset` / 超时），`npm`/`git`/`curl` 访问 GitHub 都会失败
- 本地代理：`http://127.0.0.1:7897`（Clash 类混合端口，通常常开）
- 已固化（**只对 github.com 生效**，不影响内网/其它仓库）：
  `git config --global http.https://github.com.proxy http://127.0.0.1:7897`
- 临时用法（不改配置）：
  `git -c http.proxy=http://127.0.0.1:7897 -c https.proxy=http://127.0.0.1:7897 <命令>`
- 代理失效时先探测端口：
  `for p in 7890 7897 10809 10808 1080 8080; do (echo >/dev/tcp/127.0.0.1/$p) 2>/dev/null && echo $p; done`
- 清理配置：`git config --global --unset http.https://github.com.proxy`

## Git 身份与仓库
- Git 身份：`陈建鹏 <129568493+JianpChen@users.noreply.github.com>`（GitHub 隐私邮箱）
- 凭据管理器 `manager` 已启用，push 到 GitHub 不需要手打 token
- Windows 通用配置已设：`init.defaultBranch=main`、`core.autocrlf=true`、`core.quotepath=false`（中文文件名正常显示）、`core.longpaths=true`
- 仓库 `D:/工作学习/git的测试使用/` → `https://github.com/JianpChen/test.git`（分支 main，已建立上游追踪）

## Windows 装原生 npm 模块的通用解法
- 症状：`pi install` / `npm install` 报 `gyp ERR! find VS` → 说明该包含原生模块且需要本地编译
- 根因通常是**两件事叠加**：① 预编译二进制托管在 GitHub Releases（直连不通）；② node-pre-gyp 在 node 22 上有 `Completion callback never invoked!` 的已知 bug，退出码非 0 导致 npm 回滚整棵树
- **可靠解法**（以 `nodejieba@3.5.8` 为例，已验证）：
  1. `npm install <包> --prefix ~/.pi/agent/npm --legacy-peer-deps --ignore-scripts`（跳过编译脚本）
  2. 用代理手动下预编译包：`curl.exe -L --proxy http://127.0.0.1:7897 -o nj.tar.gz https://github.com/yanyiwu/nodejieba/releases/download/v3.5.8/nodejieba-v3.5.8-node-v127-win32-x64-unknown.tar.gz`
  3. 解包放置二进制：`tar xzf nj.tar.gz Release/nodejieba.node` → `cp Release/nodejieba.node ~/.pi/agent/npm/node_modules/nodejieba/build/Release/`
  4. 手动把包名写进 `~/.pi/agent/settings.json` 的 `packages`（install 失败时不会自动写入）
- 适用于所有「预编译包在 GitHub Releases + node-pre-gyp」的包

## pi 包管理机制（源码结论）
- `package-manager.js` 判定是否重装：`needsInstall = !existsSync(installedPath) || !installedNpmMatchesConfiguredVersion(...)`
- 而 `installedNpmMatchesConfiguredVersion` 对**没有版本号**的 spec 直接返回 `true`
- **结论：手动装的包，spec 千万别写版本号**，否则 pi 启动时会重装并再次踩同样的坑

## pi-memory-md 在 Windows 上的路径 bug（已本地修复）
- 症状：会话启动报 `Error: Directory exists but is not a git repo: C:/Users/23932/.pi/memory-md (start/pull)`
- 根因：`utils.ts` 的 `getProjectMeta()` 里 `cwd = path.resolve(...)` 是**反斜杠**，`gitRoot = git rev-parse --show-toplevel` 是**正斜杠**，而 `memory-git.ts:96/158` 和 `tools.ts:167` 用**严格相等**比较两者 → Windows 上永远不相等 → 明明是真仓库却报「不是 git 仓库」
- **不是配置问题**：改 `localPath` 的正/反斜杠写法都没用
- 修复：在 `getProjectMeta()` 里对 git 输出的路径做 `path.normalize()`（同时处理 `gitRoot` 和 `mainRoot`，保证 worktree 判断也自洽）。一处修改修好全部 3 个调用点
- 位置：`~/.pi/agent/npm/node_modules/pi-memory-md/utils.ts`；备份 `utils.ts.bak-*`
- 完整补丁 + 可复现验证脚本 + 回滚方法：`D:/工作学习/pi的记忆研究/pi-memory-md-Windows路径补丁/README.md`
- 上游 `github.com/VandeeFeng/pi-memory-md` 的 `main` 分支截至 2026-09-13 **未修复**，更新插件后需重新打补丁

## 记忆系统现状（2026-09-13 起）
- #fact 当前用 **pi-memory-md**（Markdown 文件 + git 版本管理的记忆库）；旧的两套（`pi-memory-agent` 插件、`memory-profiles` skill）已卸载删除
- 记忆库远程：`https://github.com/JianpChen/pi-memory-md.git`，本地 `C:/Users/23932/.pi/memory-md`
- 配置：`~/.pi/agent/settings.json` 顶层键 `pi-memory-md`，`memoryDir.repoUrl` **必填**；已开自动同步 `hooks.sessionStart=["pull"]`、`hooks.sessionEnd=["push"]`（默认 sessionEnd 是空数组=不自动推）
- **分区规则**：分区名 = 当前所在目录的 git 仓库目录名（不是 git 仓库则用目录名），另有 `global/` 共享区对**所有项目**可见
- 初始化命令：`bash ~/.pi/agent/npm/node_modules/pi-memory-md/skills/memory-init/scripts/memory-init.sh`（本机**无 jq**，会走 grep/sed 回退，仍可用）；斜杠命令是 `/skill:memory-init`
- #lesson 记忆文件格式硬要求：文件**必须有被 `---` 包裹的 frontmatter**；`description` 必须 string、`tags` 必须数组、`limit` 必须正数，否则**不参与检索**
- 本机已装的 skill：`cpp-defect-review`、`cpp-assign-plus-review`、`doc-to-md`（均在 `~/.pi/agent/skills/`）

## ⚠️ 写记忆的纪律（每个项目都适用，别踩）
- **项目记忆必须放在 `<分区>/core/` 下**（项目记忆用 `<分区>/core/project/`）。
  实测：放在 `core/` 之外（如 `<分区>/docs/`）的文件**既不会被注入，也搜不到**。
  因为注入根和搜索根都写死为 `<分区>/core/`。
- 但 **`memory_check` 的“Memory files (N)”会把 `core/` 之外的文件也算进去** → 这个数字会骗人，不能当作“已生效”的依据。
- **只有真正跨项目通用的事实才写进 `global/`** —— global 会注入到**每一个项目**，写错就污染全部项目。
- 写记忆没有沙箱：`memory_write` / `memory_read` / `memory_list` 在 0.1.38 里**全是注释状态**，实际只注册了 `memory_sync` / `memory_search` / `memory_check`。
  写文件靠 native `write` / `edit` 直接写，**写错分区不会报错** → 必须先用
  `bash ~/.pi/agent/npm/node_modules/pi-memory-md/skills/memory-write/scripts/memory-write.sh project-dir` 确认目标目录。
- 完整分析（4 个污染风险点 + 实测证据 + 诊断方法）：`pi的记忆研究/core/project/记忆隔离与污染分析.md`

## 本机常用路径速查
| 用途 | 路径 |
|---|---|
| 记忆库根 | `C:/Users/23932/.pi/memory-md` |
| pi 配置 | `C:/Users/23932/.pi/agent/settings.json` |
| pi 扩展包 | `C:/Users/23932/.pi/agent/npm/node_modules/` |
| skill 目录 | `C:/Users/23932/.pi/agent/skills/` |
| 学习/项目根 | `D:/工作学习/` |
| 记忆系统研究 | `D:/工作学习/pi的记忆研究/` |
