# Changelog

## 0.1.2

The chip is desktop-only, and its main button starts a worktree.

- The chip's main button `Start with worktree` starts the successor in a fresh worktree (both rehearsals did). The dropdown's `Start locally` starts it in the main checkout; `Fix in this session` sends the prompt back to the outgoing session and must not be used. Step 5 and the mistakes table say so, and step 6 tells the user.
- The off-duty rename in step 4 is invisible to phone and remote session lists, which then show the old and new session under the same name. Step 6 now opens the final reply with an explicit off-duty statement so a remote user can tell them apart. Mistakes table gains the row.
- The chip renders only in the desktop app's session view; a user on the phone cannot see or click it. Handoff notes now carry a third section, the fully-filled successor prompt, so a remote user can open any new session and paste it. Step 6 tells the user to stand by or paste.

## 0.1.1

The chip title is the successor's final name.

- `set_session_title` only updates the desktop app's local store; the phone and remote session lists keep the title the session was created with, which is the chip's `title`. The outgoing session now writes `<name>` directly into the chip title and puts the `[test]` rehearsal marker in `tldr`.
- Chip template step 1 changes from "rename" to "confirm the title is already `<name>`, rename only if not".
- Mistakes table gains the row for it.

## 0.1.0

Initial release. Session hand-off for the Claude Code desktop app.

- Six steps for the outgoing session: measure facts, release bindings with proof, write handoff notes, rename itself off duty, spawn a one-click chip, report and stand by.
- Six steps for the successor, carried in the chip prompt: rename back, read the notes, locate itself with `git worktree list` and check git, notify collaborators under its own ref, report, continue from the first remaining task.
- Chip prompt template and a table of the mistakes the first rehearsal produced without the skill.
- Verified in a live rehearsal on 2026-09-02; the successor started in a fresh worktree, and step 3 of the template was amended for it.
