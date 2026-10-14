# handoff

**每个 agent 会话都会结束。这个 skill 让你说一句「换人」，下一个会话就接着干。**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-plugin-d97757)](#安装)
[![Desktop app](https://img.shields.io/badge/Claude%20Code-desktop%20app-555)](#前提)

[English](README.md) · [中文](README.zh-CN.md)

```
outgoing session                       user              successor
─────────────────────────────────────  ────────────────  ──────────────────────────────────
1  measure facts (git, watermark)      says "hand off"
2  release bindings, keep the proof    in this window
3  write the handoff notes
4  rename itself  "<name>(<time> off duty)"
5  start the successor  ─────────────► (nothing) ──────► 1  detach from the predecessor
6  report, then stand by                                 2  confirm the title is <name>
                                                         3  read the handoff notes
                                                         4  git worktree list, git status
                                                         5  notify collaborators (own id)
                                                         6  report "taken over"
                                                         7  continue from remaining task #1
```

用户只做一件事：在要换人的窗口说一句「换人」。其余全部由两个会话完成。应用不放行直接启动时退为点一张卡片，见[三条启动路径](#三条启动路径)。

---

## 问题

没有固定流程时，换人通常是：用户手动新建会话、贴一段上下文、新会话把旧会话早已知道的东西再推导一遍。一路上会丢东西：

- 会话级绑定留在死会话里：Artifact watch、后台任务、Monitor。
- 协作会话还在等一个不再应答的会话。
- 两个会话同名，消息发错人。
- 交接文档最后才写、凭记忆写，写的时候产生它的上下文已经没了。

## 它做什么

`handoff` 把换人拆给三方，各自一份固定清单。

| 谁 | 做什么 | 不做什么 |
|---|---|---|
| **旧会话** | 现跑事实 → 释放绑定 → 写交接文档 → 改名下班 → 启动继任者 → 待命 | 不归档自己；不通知协作方；不接新任务 |
| **继任者** | 脱离前任 → 确认标题 → 读交接文档 → 定位自己并核对 git → 通知协作方 → 续跑 | 不重新推导旧会话已定的结论 |
| **用户** | 在要换人的窗口说一句「换人」；没直接起来时点一下卡片或芯片 | 其余全不用管 |

三个设计决定承担了大部分分量：

- **由继任者通知协作方，不是旧会话。** 一个会话的地址只在它启动后才存在。旧会话去通知就得等继任者起来、再从几条同名会话里挑出它。继任者自报家门没有这个问题。
- **改名格式固定**：`<原名>(<时间戳>下班)`。侧边栏靠它排序，原短名不再碰撞，继任者可以原样拿回。
- **先写文档，再启动继任者，顺序不能反。** 文档没落盘，继任者拿到的就是空壳。旧会话也不归档自己：继任者确认接手之前，它是交接文档之外一切细节的唯一副本。

继任 prompt 是一个七项模板，每个占位符都要填成具体值。skill 末尾是一张「没有它时会犯的错」表，来自实机演练与真实换人记录。

### 三条启动路径

继任者怎么起来，取决于应用给了哪个工具、以及这一轮是谁开的。

| 路径 | 条件 | 用户要做什么 |
|---|---|---|
| **A 直接启动** | 有 `start_session`，且本轮里有用户本人在这个窗口说的话 | 无 |
| **B 提议卡片** | 有 `start_session`，但本轮由别的会话的消息、定时任务或后台通知开启 | 点卡片，或回这个窗口说一句「启动」 |
| **C 芯片** | 应用只给了 `spawn_task`（两个工具互斥） | 点芯片 |

B 是应用有意设的闸：没有用户本人的话，agent 不能自己开会话。这句话要在桌面端说：从手机经远程控制发来的话，应用不算作本人输入；退而出的卡片在手机上也看不到（均为 2026-10-14 实测）。skill 不绕它，只保证有用户那句话时走 A。主控会话发消息叫另一条线换人，那条线只能走 B；想零点击，就去那条线的窗口说一句。

## 与 ctx-watch 的合作

[ctx-watch](https://github.com/juew/ctx-watch) 是它的姊妹 skill。ctx-watch 测上下文水位和增速，过了窗口的 75% 就提示 agent「在下一个干净边界换人」，但它不执行换人。`handoff` 执行换人，不做任何测量。

| | ctx-watch | handoff |
|---|---|---|
| 回答的问题 | 多大、涨多快、还剩几次调用 | 这个会话怎么停、下一个怎么起、什么都不丢 |
| 触发 | 自动 hook，加按需跑 `ctx-audit` | 用户说 交接 / 换人 / 下班 |
| 到 75% 线 | 报告，并指出下一个干净边界 | 执行 |
| 依赖对方 | 否 | 否；装了 ctx-watch 时用 `ctx-audit` 给交接文档里的水位一行取值 |

两者互不调用。跨过换人线是报告，不是命令：换不换是用户的决定，`handoff` 也可以在任何水位演练，说明这是 `[test]` 即可。常见顺序是：ctx-watch 报 ROTATE → 用户说换人 → `handoff` 跑完六步并启动继任者 → 继任者从接近零的上下文起步 → ctx-watch 接着看继任者。

两者同装时有一个要知道的坑：`ctx-audit` 按历史峰值自校准窗口。机器上没有会话接近过真实的 1M 窗口时，它按 200k 算，换人线会早报。这种情况下 `handoff` 第 1 步加 `CTX_WINDOW=1000000`。

## 安装

```bash
/plugin marketplace add juew/handoff
/plugin install handoff@handoff-marketplace
```

或把 `skills/handoff/` 复制到 `~/.claude/skills/handoff/`。

### 前提

Claude Code **桌面版**。skill 用到桌面版的会话工具：启动继任者用 `start_session`（应用没给它时用 `spawn_task`），脱离前任用 `detach_session`，改名用 `set_session_title` 和 `get_session`，对接名单用 `list_sessions`。终端 CLI 没有拉起会话的原语，第 5 步在那里没有对应物。

## 使用

在要换人的那个会话窗口说 交接、换人、下班 或 hand off。旧会话跑完六步，直接把继任者启动起来，最后只说一句：继任者的会话 id、交接文档在哪。继任者脱离前任、做完其余的事，报「接手完成」并附每条通知的结果。

应用没放行直接启动时，旧会话会改出一张卡片并说明原因：点它，或回那个窗口说一句「启动」。人在手机上时，说「换人」不会直接启动，因为远程发来的话不算本人输入。卡片在手机上也看不到，只能新建一个会话，把交接文档里的「继任者启动 prompt」贴进去。

想演练而不真换人，说明这是 `[test]`。标记进 `background` 或芯片的 `tldr`，不进标题。

## 已验证

首次实机演练在 2026-09-02，就在托管这个 skill 自己交接文档的仓库里做的。0.2.0 的直接启动路径依据 2026-10-08 至 10-14 的真实换人记录。

| | 状态 |
|---|---|
| 同情景基线，不带 skill | 先出芯片再写文档、自创改名标记、拒绝通知协作方、不释放绑定、芯片里没有继任者清单 |
| 同情景带 skill | 五项全部纠正 |
| 点芯片启动的继任者 | 启动在**新 worktree** 而非主 checkout；模板的 git 核对一项以 `git worktree list` 开头 |
| 继任者侧基线，不带 skill | worktree、写回文档、水位 hook、续跑前置四问全对；唯一多停的一下是交接文档已标明无关的一个未跟踪目录 |
| 真实换人记录，17 次 `start_session` 调用 | `user_asked` 试 7 次：本轮有用户本人输入的 5 次全部直接启动，没有的 2 次全部被拒；另 10 次 `own_initiative` 全在没有用户输入的一轮里 |
| 0.2.0 基线，旧 skill 配新工具 | 能猜到直接启动，但继任者不脱离前任、漏 `background`、被拒后没有退路 |
| 0.2.0 带 skill，七个场景 | 直接启动、卡片、被拒退路、仅有芯片、身处 worktree、后台任务陷阱、继任者首轮，全部按 skill 作答 |
| 手机经远程控制直接启动，2026-10-14 实测 | 被拒，原话与「别的会话发消息开启的一轮」相同 |
| 提议卡片，2026-10-14 实测 | 桌面端显示，主按钮同旧芯片为 `Start with worktree`；手机端看不到 |
| 尚未实测 | 卡片主按钮在传了 `use_worktree:false` 时的落点；旧会话身处 worktree 时 `use_worktree:true` 的实际落点 |

## 许可

MIT
