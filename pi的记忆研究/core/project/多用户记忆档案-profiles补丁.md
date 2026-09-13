---
description: pi-memory-md 多用户档案（profiles）补丁：多人在同一台 pi 上各用独立记忆仓库，含 /memory-profile 命令、切换器脚本、验证与部署方法
tags: [pi-memory-md, 多用户, profiles, 补丁, 记忆隔离, 实测]
created: 2026-09-14
updated: "2026-09-14"
---

# pi-memory-md 多用户档案（profiles）补丁

> 需求背景：实验室设备紧张，**多人共用一个 pi**。每人要有**自己独立的记忆仓库（独立 git 链接）**，
> 并且要方便配置管理（一条命令就能看到"现在用的是谁的仓库"）。

## 结论（已验证）

给 `pi-memory-md@0.1.38` 加了 **profiles（档案）** 机制：

- 每个档案 = 一个 `repoUrl` + 一个 `localPath`；**记忆正文、项目分区、`global/`、Tape 数据全部分开**
- 斜杠命令 `/memory-profile`（别名 `/memory-user`）查看/切换/新增/删除
- 另有零依赖的 pi 外部切换器 `tools/pi-memory-profile.mjs`（`.cmd` 双击出交互菜单）
- 47 项离线断言全通过：`node _verify-profiles.mjs`（现为 49 项）
- **完全向后兼容**：没配 `profiles` 时与原插件行为、注入头部格式一模一样（已用真实配置实测）

## 配置格式

```jsonc
// ~/.pi/agent/settings.json
"pi-memory-md": {
  "profile": "alice",          // 无状态文件/环境变量时的默认档案
  "profilesDir": "~/.pi",      // 档案默认 localPath 的父目录
  "profiles": {
    "alice": { "repoUrl": "https://github.com/alice/memory-md.git" },
    "bob": { "repoUrl": "https://github.com/bob/memory-md.git", "localPath": "~/.pi/memory-md-bob" }
  }
}
```

- 档案字段覆盖顶层同名字段，未写的继承顶层；`localPath` 默认 `<profilesDir>/memory-md-<名字>`
- **档案定义只从全局 settings.json 读**，项目 `.pi/settings.json` 不能定义/切换档案（防劫持）
- 生效优先级：`PI_MEMORY_PROFILE` 环境变量 > 状态文件 `~/.pi/agent/memory-md-profile` > `settings.profile` > 旧的顶层配置

## 常用命令

```
/memory-profile            # 当前用户 + 仓库链接 + 本地路径 + git origin 校验 + 全部档案
/memory-profile repo       # 一行：当前用户 + 仓库链接（最快捷）
/memory-profile use bob    # 切换：写状态文件 + 立即 pull/clone + 热重载（无需重启 pi）
/memory-profile use none   # 清除档案，回退顶层配置
/memory-profile add carol https://github.com/carol/memory-md.git
/memory-profile remove carol   # 只删定义，不删本地克隆
```

pi 外部：`node tools/pi-memory-profile.mjs use bob`，或双击 `tools/pi-memory-profile.cmd` 出菜单。

### 提示语义（FAQ，少踩坑）

| 提示 | 含义 | 要紧吗 |
|---|---|---|
| `Warning: No memory profile configured. Use /memory-profile add <name> <repoUrl>` | 只有 `/memory-profile list`（别名 `/memory-user list`）在 `profiles` 为空时给，源码 `index.ts:711-715` | **不是错误**：还没定义任何档案，插件走顶层单仓库模式（`memoryDir.repoUrl` + `localPath`），记忆功能正常 |
| `/memory-profile`（不带参数）显示 `(none - using top-level repoUrl/localPath)` | 同上——未启用档案，但会列出当前仓库链接、本地路径、git origin | 正常 |
| `Unknown memory profile "x". Configured: ...` | `use <名字>` 的名字不在 `profiles` 里 | 真错误，拼错或还没 `add` |
| `Warning: local clone origin is ... but this profile expects ...` | 本地目录被别的用户的仓库占着 | 切换时会**跳过同步**，需手动处理该目录 |

只有一个用户在用这台 pi 时，**不需要配置 profiles**，上面第一条警告可以直接无视。

## 关键技术点（下次改插件可复用）

- 新增 `profiles.ts`：档案解析 + 状态文件读写 + `settings.json` **安全改写**（保留其它顶层键、写前备份 `.bak`、tmp+rename 原子替换）
- `loadSettings()` 里 `applyActiveProfile()` 做「档案 over 顶层」的 deep merge；`memoryDir.globalMemory` 需要**镜像**到嵌套结构才能被 `getGlobalMemoryDir()` 读到
- 运行中切换靠 `refreshSettingsFromDisk()` **原地清空+赋值** `settings` 对象 —— 所有闭包持的是同一个引用，所以无需重启 pi
- 切换后必须：`state.activeTapeRuntime?.service.detachSessionTree()` + 置 `null`，并重置 `initialMemoryContext` / `hasDeliveredInitialContext`，否则还是旧仓库的上下文
- 记忆上下文头部新增 `profile="名字"` 属性与一行说明；未配 profiles 时不出现（格式兼容）

## 相关文件

| 内容 | 路径 |
|---|---|
| 补丁说明 + 部署 + 验证方法 | `D:/工作学习/pi的记忆研究/pi-memory-md-多用户档案补丁/README.md` |
| 验证脚本（49 项断言，全离线） | `同目录/_verify-profiles.mjs`（配 `_stub-pi-*.mjs`） |
| 离线一键安装包（含补丁，5.9 MB） | 同目录 `pi-memory-md-offline-v0.1.38-profiles.zip`（sha256 `b5deed7d…a49e`） |
| **中文离线安装指南** | `D:/工作学习/pi的记忆研究/pi-memory-md离线安装指南.md`（打包脚本会把它写成包内 `OFFLINE-INSTALL.md`） |
| 原始文件留档 | 同目录 `index.ts.原始版`、`memory-core.ts.原始版`、`types.ts.原始版` |
| 补丁后完整源码（重打补丁用） | 同目录 `pi-memory-md-patched-0.1.38-profiles.zip` |
| pi 外部切换器 | `D:/工作学习/pi的记忆研究/tools/pi-memory-profile.mjs` / `.cmd` |
| 改动的插件文件 | `~/.pi/agent/npm/node_modules/pi-memory-md/`（`profiles.ts` 新增；`types.ts`/`memory-core.ts`/`index.ts`/`package.json`/`README.md` 修改；`skills/memory-profile/` 新增） |

## 注意

- **package.json 版本号故意保持 0.1.38**：pi 用 `installedNpmMatchesConfiguredVersion` 判断是否重装，改版本号会被判定不匹配 → 覆盖 `node_modules` 里的补丁
- 多人**同时**用 pi 时，"当前档案"（状态文件）是机器级共享状态会互抢 → 给每人一个启动脚本 `set PI_MEMORY_PROFILE=alice` 再 `pi`
- 切换档案后，本会话**之前**注入的旧档案记忆仍在上下文里 → 要彻底隔离请开新会话
- 切换时如果本地目录的 git origin 与档案仓库不一致（目录被别的用户克隆占用），插件会**拒绝同步并告警**
- 插件升级 / `npm install` 会冲掉补丁 → 用 zip 重新覆盖
