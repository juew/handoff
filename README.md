# handoff

**Every agent session ends. Say "hand off" and the next session picks up where this one stopped.**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-plugin-d97757)](#install)
[![Desktop app](https://img.shields.io/badge/Claude%20Code-desktop%20app-555)](#requirements)

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

The user does one thing: say "hand off" in the window being rotated. Everything else is done by the two sessions. When the app will not allow a direct start, it falls back to one click on a card; see [Three ways to start](#three-ways-to-start).

---

## The problem

Without a fixed procedure, a hand-off usually looks like this: the user opens a fresh session, pastes some context, and the new session re-derives what the old one already knew. Along the way things get lost:

- Bindings stay alive in the dead session: artifact watches, background tasks, monitors.
- Collaborating sessions keep waiting on a session that stopped answering.
- Two sessions carry the same title, and messages go to the wrong one.
- The notes are written last, from memory, after the context that produced them is gone.

## What it does

`handoff` splits the rotation between three parties and gives each a fixed list.

| Who | Does | Does not |
|---|---|---|
| **Outgoing session** | measures facts, releases bindings, writes notes, renames itself, starts the successor, stands by | archive itself, notify anyone, take new work |
| **Successor** | detaches from the predecessor, confirms its title, reads the notes, checks git, notifies collaborators, continues | re-derive conclusions the notes already settled |
| **User** | says "hand off" in the window being rotated; clicks a card or chip only when the start was not direct | anything else |

Three decisions carry most of the weight:

- **The successor does the notifying, not the outgoing session.** A session's address only exists once it is running. If the old session tried to notify on the successor's behalf it would have to wait for the successor to start and then pick it out of several sessions with the same name. The successor simply introduces itself.
- **The rename has a fixed shape**: `<name>(<timestamp> off duty)`. The sidebar sorts on it, and the original short name stops colliding, so the successor can take it back verbatim.
- **Notes before the successor, never after.** A successor whose notes are not on disk yet starts from an empty shell. The outgoing session also never archives itself: until the successor confirms it has taken over, it is the only copy of everything the notes left out.

The successor prompt is a seven-item template with no placeholders left in it. The skill ends with a table of the mistakes a session makes without it, taken from live rehearsals and real rotation records.

### Three ways to start

How the successor comes up depends on which tool the app provides and on who opened the current turn.

| Path | When | What the user does |
|---|---|---|
| **A direct start** | `start_session` is available and the user's own words to this window are in the turn | nothing |
| **B proposal card** | `start_session` is available but the turn was opened by another session's message, a scheduled task or a background notification | clicks the card, or says "start it" in this window |
| **C chip** | the app provides only `spawn_task` (the two tools are mutually exclusive) | clicks the chip |

B is a gate the app sets on purpose: without the user's own words, an agent cannot open a session by itself. Those words have to be typed on the desktop: a message sent from a phone over Remote Control does not count as the user's own input, and the card that B puts up does not show on a phone either (both tested live on 2026-10-14). The skill does not route around it; it only makes sure path A is taken when those words are there. When a controller session messages another line to rotate, that line can only take B. For zero clicks, say it in that line's own window.

## Works with ctx-watch

[ctx-watch](https://github.com/juew/ctx-watch) is the sibling skill. It measures the context watermark and its growth rate, and at 75% of the window it tells the agent to hand off at the next clean boundary. It does not perform the hand-off. `handoff` performs it and does no measuring.

| | ctx-watch | handoff |
|---|---|---|
| Answers | how big, how fast, how many calls left | how to stop this session and start the next without losing anything |
| Trigger | automatic hook, plus `ctx-audit` on demand | the user says 交接 / 换人 / hand off |
| At the 75% line | reports, names the next clean boundary | executes |
| Needs the other | no | no; uses `ctx-audit` for the watermark line in the notes when it is installed |

Neither calls the other. Crossing the rotate line is a report, not a command: whether to rotate is the user's decision, and `handoff` can be rehearsed at any watermark by asking for a `[test]` run. The usual sequence is: ctx-watch reports ROTATE → the user says hand off → `handoff` runs its six steps and starts the successor → the successor starts near zero → ctx-watch watches the successor.

One thing to know when both are installed: `ctx-audit` calibrates the window from history. On a machine where no session has come near the real 1M window it assumes 200k and fires early. Step 1 of `handoff` passes `CTX_WINDOW=1000000` in that case.

## Install

```bash
/plugin marketplace add juew/handoff
/plugin install handoff@handoff-marketplace
```

Or copy `skills/handoff/` into `~/.claude/skills/handoff/`.

### Requirements

The Claude Code **desktop app**. The skill uses the desktop session tools: `start_session` to start the successor (`spawn_task` when the app does not provide it), `detach_session` to detach from the predecessor, `set_session_title` and `get_session` for the rename, `list_sessions` for the collaborator list. The terminal CLI has no session-spawn primitive, so step 5 has no equivalent there.

The skill body is written in Chinese. Its trigger phrases include the English forms.

## Usage

In the window of the session being rotated, say 交接, 换人, 下班, or "hand off". The outgoing session runs its six steps, starts the successor directly, and ends with one sentence: the successor's session id and where the notes are. The successor detaches from its predecessor, does the rest, and reports "taken over" with the result of each notification.

When the app does not allow the direct start, the outgoing session puts up a card instead and says why: click it, or say "start it" in that window. From a phone, "hand off" does not start the successor directly, because a message sent over Remote Control does not count as the user's own input. The card does not show on a phone either, so the only way is to open a new session and paste the "successor prompt" section of the notes.

To rehearse without a real rotation, ask for a `[test]` run. The marker goes into `background` or the chip's `tldr`, never into the title.

## Verified

The first live rehearsal ran on 2026-09-02, in the repository that hosts the skill's own handoff notes. The direct-start path in 0.2.0 rests on real rotation records from 2026-10-08 to 10-14.

| | Status |
|---|---|
| Baseline, same scenario, no skill | The session shipped the chip before writing notes, invented its own rename marker, declined to notify collaborators, left bindings alive, and put no successor checklist in the chip |
| Same scenario with the skill | All five corrected |
| Successor launched from the chip | Started in a **fresh worktree**, not the main checkout; the template's git check now begins with `git worktree list` |
| Successor-side baseline, no skill | Handled the worktree, the notes write-back, the watermark hook and the pre-continue checks correctly; its one extra stop was a known-unrelated untracked directory the notes had already flagged |
| Real rotation records, 17 `start_session` calls | `user_asked` was tried 7 times: all 5 in a turn with the user's own input started directly, both of the 2 without it were refused; the other 10 calls were `own_initiative`, all in turns with no user input |
| 0.2.0 baseline, old skill with the new tools | Guessed the direct start, but the successor never detached, `background` was missing, and there was no fallback after a refusal |
| 0.2.0 with the skill, seven scenarios | Direct start, card, fallback after refusal, chip only, outgoing session in a worktree, background-task trap, successor's first turn: all answered as the skill specifies |
| Direct start from a phone over Remote Control, tested 2026-10-14 | Refused, with the same reply as a turn opened by another session's message |
| Proposal card, tested 2026-10-14 | Shows on the desktop with the old chip's `Start with worktree` main button; does not show on a phone |
| Not yet tested live | Where the card's main button lands when `use_worktree: false` was passed; where the successor lands with `use_worktree: true` from a worktree session |

## License

MIT
