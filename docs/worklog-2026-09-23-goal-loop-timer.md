# Loop timer: a fixed cadence for Loop continuations

## What changed

`GoalSettings.loopTimerMinutes` (0 = Off, whole minutes, accepted range 0..720) gives a chat
a configured interval between one finished Loop answer and the next automatic message,
replacing the evidence-driven 2/5/10/15-minute pickup ladder for that chat's generated
continuations. The shipped default stays 0: a chat that types into itself on a clock is a
deliberate choice, not something to discover by flipping a switch that is already on.

The decision lives in `goal.ts::loopTimerMsFor()`, which answers non-zero only when the
chat's effective switch is enabled and its mode is `loop`. `bridge.ts::owedPickups` stamps
each owed Loop decision with that interval; `inspectOwedPickups` uses it instead of
`PICKUP_BACKOFF_MS[0]` when arming the watch and instead of every rung when advancing it.
`notePickupActivity` leaves a timer watch untouched, and the `continuationForSession` guard
inside `inspectOwedPickups` holds the watch while a Compact & Resume ticket is open and
re-arms one fresh full interval from the moment the ticket clears. A configured timer also
defeats `astraFinishOnly`: finish-only waits for a boundary the agent announces, which is
the opposite instruction to the interval the user named, so the chat falls back to the
ordinary after-turn boundary.

The wait is presented as the new `timer` reason of `GoalWait`. Both UIs render it — the
desktop lifecycle row (`Waiting for the Loop timer`) and the extension stage above the
composer — and the renderer deliberately keeps it out of `sharedRecoveryWait`: an
independent Loop wait, never the dock's shared silence countdown. Settings gained a
**Loop timer** row (minutes, 0..720) between "Loop response source" and the model picker,
translated in the six locale catalogs.

Authored queued input on the same chat keeps the ordinary ladder: its pickup is a stall
recovery regardless of the chat's Goal/Loop mode. The ordinary silence pass still applies
to a timer chat under its own rules; the timer owns pickup timing only.

## Why these choices

- **Loop only.** Goal exists to decide that no further message is needed, so a fixed cadence
  would contradict what it is for. `loopTimerMsFor` reads `goalSwitchFor`, so a per-chat mode
  wins over the app default exactly as ordinary continuation does, and a disabled chat has no
  delivery authority.
- **A dwell time, not a poll.** The reply obligation, its source question,
  `loopReplyHasAuthority`, the MCP-call requirement and the twelve-hour lifetime still decide
  whether anything is owed; the timer only decides when the next step is due once something
  is. Nothing polls the page on a clock.
- **Activity immunity.** Page activity does not push a timer watch out — the user asked for a
  cadence, not for a stall to be detected. Genuine revocation (new work retires the source,
  Off, block, a newer turn, replacement) still applies unchanged.
- **Compaction awareness.** A rebind can outlast a short interval, and a deadline that
  expired mid-compaction would fire the instant the replacement chat appeared — the exact
  opposite of taking the compaction into account. The guard re-arms `now + timer` on every
  sweep while the continuation is open, so the interval restarts from the only moment it
  could actually have begun.
- **The field is optional in the type**, so existing wholesale config and test literals keep
  validating; the zod schema accepts 0..720 with default 0, the IPC settings schema mirrors
  it, and the `.default({...})` literal in `config.ts` names it explicitly.

## Tests

- `test/goal.test.ts` — `loopTimerMsFor` arms only for an enabled, Loop-mode chat with a
  configured interval (Off, default 0 and Goal mode all answer zero), and a configured timer
  defeats `astraFinishOnly` while zero restores it.
- `test/ipc.test.ts` — "round-trips the Loop timer and preserves a newer value across a stale
  renderer save": default 0, persistence to disk, the three-way merge keeping a concurrent
  browser-side edit, and 0..720 bounds.
- `test/config.test.ts` — the shipped default object carries `loopTimerMinutes: 0`.
- `test/bridge.test.ts` — a timer chat arms one fresh interval from acceptance rather than
  the 2-minute opening rung, queues no Goal repair inside it, holds through a Compact & Resume
  with the deadline re-armed a full interval from the moment the ticket clears, queues
  exactly one Goal repair only after that full interval, and does not move the deadline when
  native work arrives (`turn_start` + `page_tool` observation batch).
- `test/renderer-timeline.test.ts` — the lifecycle row renders `Loop · Waiting for the Loop
  timer` with its own countdown even when a silence recovery row shares the same instant.
- `test/content-script.test.ts` — the extension stage reports `Waiting for the Loop timer`
  with the remaining time.
- `test/renderer-layout.test.ts` — the Settings number-input registry includes the new row;
  compaction still has a single threshold input.

## Checks

- `LANG=en_US.UTF-8 npm run typecheck` — pass.
- `LANG=en_US.UTF-8 npm run verify` — pass: ripgrep staged, public-history privacy check
  (206 commits, 12 tags), license notices (155 production packages, 7 catalog entries, 730
  native archives), typecheck, Electron resolve, then the full Vitest run (214 + 1 files
  passed, 13 + 1 skipped; 5,755 + 6 tests passed, 118 + 20 skipped).
- `LANG=en_US.UTF-8 npm run build` — pass (`electron-vite`, 3.58s).

All evidence above is source + tests + build only; no package, install, commit or live
browser behavior is claimed.
