# Changelog

## 0.2.0

The successor starts without a click when the user asks for the rotation in the session's own window.

- Step 5 is now "start the successor" with three paths. A: `start_session` with `initiation: "user_asked"`, `context: "fresh"`, starts it at once when the user's own words to this window are in the turn. B: the same call as `own_initiative` puts up a proposal card when the turn was opened by another session's message or an automated event, or when A is refused. C: the `spawn_task` chip, kept for app builds that provide it instead of `start_session` (the two tools are mutually exclusive).
- The gate behind B is the app's own and is not routed around: no retries, no `offer`, no asking another session to start it. A controller session that messages another line to rotate will always get a card there.
- The successor template gains a first step, `detach_session("self")`. A successor started by `start_session` is a side session of its predecessor and wakes it at the end of every turn until it detaches. The template is now seven items.
- `use_worktree` depends on where the outgoing session is: `false` from the main checkout, `true` from a worktree, where a successor that shared the worktree could not detach and would lose edit access once the predecessor is archived. Uncommitted changes do not travel with `true`, so the outgoing session reports them to the user first.
- The start must happen in the turn that carries the user's words. Background tasks and background subagents before step 5 move it into a notification-opened turn, where it is refused.
- `hand_off_to_session` and `context: "fork"` are named as mistakes: they copy the whole conversation into the successor.
- The collaborator list records `local_…` session ids from `list_sessions`, and the successor addresses them by id.
- The `[test]` marker goes into the first line of `background` (or the chip's `tldr`), never the title.
- Tested live on 2026-10-14: "hand off" sent from a phone over Remote Control does not pass the gate; the refusal is the same as for a turn opened by another session's message. Zero clicks therefore needs the user at the desktop, and the paste-the-prompt fallback in the notes stays for remote use.
- The proposal card is the same component as the old chip, `Start with worktree` main button included, and it does not show on a phone (both seen live on 2026-10-14). Step 6 of path B tells the user to pick `Start locally` from its menu, as for the chip. From a phone neither the direct start nor the card works; pasting the successor prompt into a new session is the only remote path.
- Not yet tested live: where the card's main button lands when `use_worktree: false` was passed, and where the successor lands with `use_worktree: true`.

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
