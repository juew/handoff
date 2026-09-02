# handoff

**Every agent session ends. This one leaves the next session a single click away from where it stopped.**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-plugin-d97757)](#install)
[![Desktop app](https://img.shields.io/badge/Claude%20Code-desktop%20app-555)](#requirements)

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

The user does one thing: click the chip. Everything else is done by the two sessions.

---

## The problem

The Claude Code desktop app does not let an agent open a new session. So a hand-off usually looks like this: the user opens a fresh session, pastes some context, and the new session re-derives what the old one already knew. Along the way things get lost:

- Bindings stay alive in the dead session: artifact watches, background tasks, monitors.
- Collaborating sessions keep waiting on a session that stopped answering.
- Two sessions carry the same title, and messages go to the wrong one.
- The notes are written last, from memory, after the context that produced them is gone.

## What it does

`handoff` splits the rotation between three parties and gives each a fixed list.

| Who | Does | Does not |
|---|---|---|
| **Outgoing session** | measures facts, releases bindings, writes notes, renames itself, spawns the chip, stands by | archive itself, notify anyone, take new work |
| **Successor** | renames back, reads the notes, checks git, notifies collaborators, continues | re-derive conclusions the notes already settled |
| **User** | clicks the chip | anything else |

Three decisions carry most of the weight:

- **The successor does the notifying, not the outgoing session.** A session's address only exists once it is running. If the old session tried to notify on the successor's behalf it would have to wait for the successor to start and then pick it out of several sessions with the same name. The successor simply introduces itself.
- **The rename has a fixed shape**: `<name>(<timestamp> off duty)`. The sidebar sorts on it, and the original short name stops colliding, so the successor can take it back verbatim.
- **Notes before the chip, never after.** A chip whose notes are not on disk yet is an empty shell. The outgoing session also never archives itself: until the chip is clicked, it is the only copy of everything the notes left out.

The chip prompt is a six-item template with no placeholders left in it. The skill ends with a table of the mistakes a session makes without it, taken from the first rehearsal.

## Works with ctx-watch

[ctx-watch](https://github.com/juew/ctx-watch) is the sibling skill. It measures the context watermark and its growth rate, and at 75% of the window it tells the agent to hand off at the next clean boundary. It does not perform the hand-off. `handoff` performs it and does no measuring.

| | ctx-watch | handoff |
|---|---|---|
| Answers | how big, how fast, how many calls left | how to stop this session and start the next without losing anything |
| Trigger | automatic hook, plus `ctx-audit` on demand | the user says 交接 / 换人 / hand off |
| At the 75% line | reports, names the next clean boundary | executes |
| Needs the other | no | no; uses `ctx-audit` for the watermark line in the notes when it is installed |

Neither calls the other. Crossing the rotate line is a report, not a command: whether to rotate is the user's decision, and `handoff` can be rehearsed at any watermark by prefixing the chip title with `[test]`. The usual sequence is: ctx-watch reports ROTATE → the user says hand off → `handoff` runs its six steps and leaves the chip → the successor starts near zero → ctx-watch watches the successor.

One thing to know when both are installed: `ctx-audit` calibrates the window from history. On a machine where no session has come near the real 1M window it assumes 200k and fires early. Step 1 of `handoff` passes `CTX_WINDOW=1000000` in that case.

## Install

```bash
/plugin marketplace add juew/handoff
/plugin install handoff@handoff-marketplace
```

Or copy `skills/handoff/` into `~/.claude/skills/handoff/`.

### Requirements

The Claude Code **desktop app**. The skill uses the desktop session tools: `spawn_task` for the chip, `set_session_title` and `get_session` for the rename, `ListAgents` for the collaborator list. The terminal CLI has no session-spawn primitive, so step 5 has no equivalent there.

The skill body is written in Chinese. Its trigger phrases include the English forms.

## Usage

Say 交接, 换人, 下班, or "hand off". The outgoing session runs its six steps and ends with one sentence: the chip is out, what clicking it will do, and where the notes are.

Click the chip. The successor does the rest and reports "taken over" with the result of each notification.

To rehearse without a real rotation, ask for a `[test]` run; the chip title carries the prefix.

## Verified

The first live rehearsal ran on 2026-09-02, in the repository that hosts the skill's own handoff notes.

| | Status |
|---|---|
| Baseline, same scenario, no skill | The session shipped the chip before writing notes, invented its own rename marker, declined to notify collaborators, left bindings alive, and put no successor checklist in the chip |
| Same scenario with the skill | All five corrected |
| Successor launched from the chip | Started in a **fresh worktree**, not the main checkout; the template's step 3 now begins with `git worktree list` |
| Successor-side baseline, no skill | Handled the worktree, the notes write-back, the watermark hook and the pre-continue checks correctly; its one extra stop was a known-unrelated untracked directory the notes had already flagged |

## License

MIT
