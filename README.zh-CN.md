# handoff

**每个 agent 会话都会结束。这个 skill 让下一个会话离「接着干」只差一次点击。**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-plugin-d97757)](#安装)
[![Desktop app](https://img.shields.io/badge/Claude%20Code-desktop%20app-555)](#前提)

[English](README.md) · [中文](README.zh-CN.md)

```
outgoing session                       user       successor
─────────────────────────────────────  ─────────  ──────────────────────────────────
1  measure facts (git, watermark)
2  release bindings, keep the proof
3  write the handoff notes
4  rename itself  "<name>(<time> off duty)"
5  spawn a task chip  ───────────────► click ───► 1  rename back to <name>
6  report, then stand by                          2  read the handoff notes
                                                  3  git worktree list, git status
                                                  4  notify collaborators (own ref)
                                                  5  report "taken over"
                                                  6  continue from remaining task #1
```

用户只做一件事：点芯片。其余全部由两个会话完成。

---

## 问题

Claude Code 桌面版不允许 agent 自己开新会话。于是换人通常是：用户手动新建会话、贴一段上下文、新会话把旧会话早已知道的东西再推导一遍。一路上会丢东西：

- 会话级绑定留在死会话里：Artifact watch、后台任务、Monitor。
- 协作会话还在等一个不再应答的会话。
- 两个会话同名，消息发错人。
- 交接文档最后才写、凭记忆写，写的时候产生它的上下文已经没了。

## 它做什么

`handoff` 把换人拆给三方，各自一份固定清单。

| 谁 | 做什么 | 不做什么 |
|---|---|---|
| **旧会话** | 现跑事实 → 释放绑定 → 写交接文档 → 改名下班 → 出芯片 → 待命 | 不归档自己；不通知协作方；不接新任务 |
| **继任者** | 改回原名 → 读交接文档 → 定位自己并核对 git → 通知协作方 → 续跑 | 不重新推导旧会话已定的结论 |
| **用户** | 点芯片 | 其余全不用管 |

三个设计决定承担了大部分分量：

- **由继任者通知协作方，不是旧会话。** 一个会话的地址只在它启动后才存在。旧会话去通知就得等继任者起来、再从几条同名会话里挑出它。继任者自报家门没有这个问题。
- **改名格式固定**：`<原名>(<时间戳>下班)`。侧边栏靠它排序，原短名不再碰撞，继任者可以原样拿回。
- **先写文档，再出芯片，顺序不能反。** 文档没落盘，芯片就是空壳。旧会话也不归档自己：芯片被点之前，它是交接文档之外一切细节的唯一副本。

芯片 prompt 是一个六项模板，每个占位符都要填成具体值。skill 末尾是一张「没有它时会犯的错」表，来自首次演练。

## 与 ctx-watch 的合作

[ctx-watch](https://github.com/juew/ctx-watch) 是它的姊妹 skill。ctx-watch 测上下文水位和增速，过了窗口的 75% 就提示 agent「在下一个干净边界换人」，但它不执行换人。`handoff` 执行换人，不做任何测量。

| | ctx-watch | handoff |
|---|---|---|
| 回答的问题 | 多大、涨多快、还剩几次调用 | 这个会话怎么停、下一个怎么起、什么都不丢 |
| 触发 | 自动 hook，加按需跑 `ctx-audit` | 用户说 交接 / 换人 / 下班 |
| 到 75% 线 | 报告，并指出下一个干净边界 | 执行 |
| 依赖对方 | 否 | 否；装了 ctx-watch 时用 `ctx-audit` 给交接文档里的水位一行取值 |

两者互不调用。跨过换人线是报告，不是命令：换不换是用户的决定，`handoff` 也可以在任何水位演练，给芯片标题加 `[test]` 前缀即可。常见顺序是：ctx-watch 报 ROTATE → 用户说换人 → `handoff` 跑完六步留下芯片 → 继任者从接近零的上下文起步 → ctx-watch 接着看继任者。

两者同装时有一个要知道的坑：`ctx-audit` 按历史峰值自校准窗口。机器上没有会话接近过真实的 1M 窗口时，它按 200k 算，换人线会早报。这种情况下 `handoff` 第 1 步加 `CTX_WINDOW=1000000`。

## 安装

```bash
/plugin marketplace add juew/handoff
/plugin install handoff@handoff-marketplace
```

或把 `skills/handoff/` 复制到 `~/.claude/skills/handoff/`。

### 前提

Claude Code **桌面版**。skill 用到桌面版的会话工具：出芯片用 `spawn_task`，改名用 `set_session_title` 和 `get_session`，对接名单用 `ListAgents`。终端 CLI 没有拉起会话的原语，第 5 步在那里没有对应物。

## 使用

说 交接、换人、下班 或 hand off。旧会话跑完六步，最后只说一句：芯片已出、点了会发生什么、交接文档在哪。

点芯片。继任者做完其余的事，报「接手完成」并附每条通知的结果。

想演练而不真换人，说明这是 `[test]`，芯片标题会带上前缀。

## 已验证

首次实机演练在 2026-09-02，就在托管这个 skill 自己交接文档的仓库里做的。

| | 状态 |
|---|---|
| 同情景基线，不带 skill | 先出芯片再写文档、自创改名标记、拒绝通知协作方、不释放绑定、芯片里没有继任者清单 |
| 同情景带 skill | 五项全部纠正 |
| 点芯片启动的继任者 | 启动在**新 worktree** 而非主 checkout；模板第 3 项现在以 `git worktree list` 开头 |
| 继任者侧基线，不带 skill | worktree、写回文档、水位 hook、续跑前置四问全对；唯一多停的一下是交接文档已标明无关的一个未跟踪目录 |

## 许可

MIT
