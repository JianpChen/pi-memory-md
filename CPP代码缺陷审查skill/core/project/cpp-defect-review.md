---
description: cpp-defect-review skill 的能力、目录结构、37 条缺陷规则分类与误报抑制手段
tags: [cpp, 代码审查, skill, 静态扫描, sarif]
created: 2026-09-13
---

# cpp-defect-review：清单驱动的 C/C++ 缺陷审查 skill

## 用途
用户给一份**问题清单**（崩溃、结果不对、内存涨、卡死等）→ 反向定位 C/C++ 源码里的缺陷 → **输出 Markdown 缺陷清单**（默认文件名 `缺陷清单.md`）。

## 位置与形态
- 源目录：`D:/工作学习/CPP代码缺陷审查skill/`
- 已安装：`C:/Users/23932/.pi/agent/skills/cpp-defect-review`
- 打包：`cpp-defect-review.zip`（11 个文件）
- 技术选型：**纯 Python 3 标准库、零第三方依赖、无需联网、离线可跑**，整个目录可整体复制移植

## 三个脚本

| 脚本 | 职责 |
|---|---|
| `scan_defects.py` | 37 条规则的扫描器，支持 `md` / `sarif` / `json` 三种输出 |
| `triage_issues.py` | 问题清单 → 症状归类 → 缺陷假设 + 检索式；加 `--code` 可对代码做粗筛命中 |
| `symptom_map.py` | 14 类症状的关键词映射表，前两者共用 |

## 37 条规则分四类
| 前缀 | 类别 | 编号 |
|---|---|---|
| `CPP-LOG` | 逻辑语义 | 001 ~ 010 |
| `CPP-DAT` | 数据流 / 内存 | 001 ~ 012 |
| `CPP-API` | 函数接口 | 001 ~ 008 |
| `CPP-SYN` | 语法类型 | 001 ~ 007 |

## 关键设计
- **词法掩码**：扫描前先跳过字符串字面量、注释、原始字符串（raw string），避免误报
- 每条发现都带 `why` / `fix` / `snippet` / **人工确认要点**
- `--issues` 参数生成「问题 → 候选代码行」的关联章节，把清单和代码对应起来

## 误报抑制手段（经 fmt 库实测：609 条 → 20 条）
这些是让它可用的关键，每条都值得保留：
- 函数识别支持 **initializer list**（`ctor : a(x), b(y) {}`）
- 区分 **构造函数 / 析构函数**
- **C++17 条件内声明**（`if (auto x = f())`）跳过
- `&var` 视为**被调方写入**（不算未初始化读）
- 类名宏前缀（如 `GTEST_API_`）要剥掉再比对
- 数组按**声明行 / 作用域**匹配
- `static` 局部变量直接返回不算悬垂
- 比较器里含 `&&` / `||` / `?` 的跳过
- `printf` 位置参数 `%1$s` 要识别
- **本文件自有同名函数**不算「不安全函数」（别把自家 `strcpy` 当 libc）

## 运行控制
- 默认跳过 `build/`、`third_party/`、`gtest/`、`gmock/` 等目录
- `--fail-on` 控制退出码（便于接 CI）
- 编译期补充防线：`-Wall -Wextra -Wconversion -Wshadow`

## 相关
- 同目录另有 `cpp-assign-plus-review` skill：检测 `+=` 误写成 `=+`、`-=` 误写成 `=-` 这类复合赋值手误，输出 SARIF 2.1.0
- 两者都在 `~/.pi/agent/skills/` 下，都是纯 Python 3 零依赖
