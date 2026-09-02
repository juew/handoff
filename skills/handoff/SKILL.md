---
name: handoff
description: Use when the user says 交接 / 换人 / 下班 / 把工作交给新 agent / 接手, when a Claude Code desktop session must hand its work to a fresh session in the same project, or when rehearsing that rotation at any context watermark. Independent of ctx-watch — no threshold needs to be crossed to run it.
---

# handoff · 会话换人

**换人 = 旧会话收口 + 继任者启动。** 桌面版不允许 agent 自己开会话，所以 agent 把一切准备到「用户只需点一次」：写交接文档、给自己改名下班、用 `spawn_task` 出一个芯片，用户点芯片，继任者在同项目启动并自动完成接手。

与 `ctx-watch` 相互独立：那边只管报水位，这里只管执行换人。任何水位都可以跑，用于演练时在芯片标题前加 `[test]`。

## 三方分工

| 谁 | 做什么 | 不做什么 |
|---|---|---|
| **旧会话** | 现跑事实 → 释放绑定 → 写交接文档 → 改名下班 → 出芯片 → 待命 | 不归档自己；不通知协作方；不接新任务 |
| **继任者** | 改回原名 → 读交接文档 → `git worktree list` 定位自己 → 核对 git → 通知协作方 → 续跑 | 不重新推导旧会话已定的结论 |
| **用户** | 点芯片 | 其余全不用管 |

**为什么由继任者通知协作方**：继任者的会话 ref 只在它启动后才存在，旧会话去通知就得等它起来再从同名会话里挑——实测过「同名三条、必须带 ref 寻址」的坑。继任者自报家门没有这个问题。

## 旧会话：六步

1. **现跑事实，不引用记忆。** 水位（装了 [ctx-watch](https://github.com/juew/ctx-watch) 时）`node <ctx-watch>/scripts/ctx-audit.mjs`，真实窗口是 1M 而机器上没有会话接近过它时加 `CTX_WINDOW=1000000`；`git status --short | head`、`git log --oneline -5`；报告里任何计数都要有产生它的命令。
2. **释放会话级绑定并留证。** Artifact watch（`status` → `unwatch`）、后台任务、Monitor、锁、看板订阅——逐项实测释放，把命令与结果写进交接文档。拿不到证据的写「未确认」，不写「已释放」。
3. **写或更新交接文档。** 位置按项目或全局 `CLAUDE.md` 里既定的交接文档约定，没有约定时放 `<项目根>/docs/handoff-<项目名>.md`；该会话已在维护的文档直接更新，同一项目多角色并行时按角色分文件 `交接-<项目名>-<角色>.md`，各角色文件相互独立，不需要互相引用或统一索引。必含四项（已改文件/函数、剩余任务、验证证据含 EXIT 码、踩坑清单）之外，本 skill 再加两节：
   - **对接名单**：现跑 `ListAgents`，列出协作会话的名字 + ref + 各自正在等你什么
   - **继任者启动清单**：第 3 步之后继任者要做的事，按顺序，可勾选
   密钥、Token、连接串只写变量名与获取途径。
4. **改名下班。** `get_session("self")` 取原名，再 `set_session_title("self", "<原名>(<时间戳>下班)")`，时间戳用 `date '+%Y年%m月%d日%H:%M:%S'`，例：`M·AI小程序PM(2026年09月02日09:51:09下班)`。
5. **出芯片。** `spawn_task`：`title` = `接手 <原名>`，`cwd` = 项目根目录，`prompt` 用下面的模板逐项填满，`tldr` 一句话。
6. **告诉用户并待命。** 一句话：芯片已出、点击后会发生什么、交接文档路径。之后不再接新任务，不循环等待，不 `archive_session("self")`——归档由用户在继任者确认接手后决定。

## 芯片 prompt 模板

```
你是「<原名>」的继任者，原会话已改名为「<原名>(<时间戳>下班)」并停止接活。
1. set_session_title("self", "<原名>")
2. 读交接文档：<绝对路径>。以它为准，不重新推导已定结论
3. 先 git worktree list 确认自己启动在主 checkout 还是新 worktree（芯片可能在 fresh worktree 里启动）。再核对：git -C <主 checkout 绝对路径> status --short；git -C <主 checkout 绝对路径> log --oneline -3；主 checkout 分支应为 <branch>，自己所在 worktree 的分支不同不算异常。把所在位置写回交接文档
4. 按文档「对接名单」逐个 SendMessage：「我是 <原名> 的继任者，ref <本会话 ref>，原会话已下班；你手上等我的 <事项> 由我接」（每条 ≤400 字，summary 必填）
5. 向用户报一句「接手完成」，附对接名单通知结果
6. 从「剩余任务」第一项续跑
```

模板里每个 `<…>` 都必须由旧会话填成具体值，不留占位符。

## 常见错误

| 错误 | 纠正 |
|---|---|
| 先出芯片、事后再补交接文档 | 文档没落盘，芯片就是空壳。第 3 步在第 5 步之前 |
| 「用户没说通知谁，所以不通知」 | 通知是接手的一部分，由继任者按「对接名单」做 |
| 改名成自创标记（如「【已收口】」） | 固定格式 `(<时间戳>下班)`，侧边栏靠它按时间排 |
| 旧会话自己去 ListAgents 找继任者 | 不找。继任者自报家门 |
| 出完芯片就 `archive_session("self")` | 芯片没被点前归档，交接文档之外的一切随之消失 |
| 「看板订阅应该已经释放了」 | 没有命令与结果就写「未确认」 |
| 芯片 prompt 只写「读交接文档继续」 | 六项清单缺一不可，尤其改名与通知 |
| 继任者把「分支应为 X」套在自己所在的 worktree 上 | 芯片可能在新 worktree 启动（2026-09-02 首演实证）。git 核对一律带 `-C <主 checkout>`，先 `git worktree list` 定位自己 |
