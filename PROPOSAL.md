# cc-tmux-lifeboat: capture tmux panes and restore them anywhere

Status: proposal. Nothing here is built.

## What it is

cc-tmux-lifeboat is an Electron app that attaches to a tmux server and shows its panes live. You mark a pane for capture and restore. The app records that pane continuously until the server dies or you stop capturing it. Later you restore the capture into any location on any server, new or existing. Claude Code panes are detected automatically and restored as resumed Claude Code sessions.

**The unit of capture and restore is the tmux pane.** A capture is not tied to the session, window, pane id or server it came from. Sessions and windows appear in the UI only as a way to find panes and to choose a destination.

## Why Electron

The app must stay connected for as long as the server lives, notice when it dies, reconnect when a new one appears, and write captures to disk throughout. That is a long-lived process with filesystem access, which is what an Electron main process is. A browser GUI would need a separate bridge process for the same work. The library already ships `./electron/main` and `./electron/renderer` connectors for this split, and `examples/web-multiplexer/electron` in the library repo shows the shape.

## Architecture

Built on `@promptctl/tmux-control-mode-js` and the layering this repo already uses:

- The main process owns the `TmuxClient`, the recorder, and the on-disk store. All tmux access goes through the library's client and its structured results. The app does not shell out to `tmux` or parse raw `list-panes` text.
- The renderer is UI only. It talks to main through the library's Electron renderer connector and imports types only from the library, never protocol code.
- The library is a normal package dependency; this repo is separate from it.

## Behavior

1. **Attach and live view.** Choose a tmux server to attach to. The UI shows its sessions, windows and panes, updating as the server changes.
2. **Mark a pane for capture/restore.** The user marks a pane. A marked pane is recorded from then on; an unmarked pane is not. The user never declares what runs in a pane. Marking mints a capture id (a UUID) that is the capture's only identity. The live tmux pane id is a temporary link to the running pane while it is being recorded; it is dropped when the capture ends and is never part of the capture's identity or stored location.
3. **What a capture records:**
   - The launch command line (argv) and working directory.
   - The pane's history (scrollback), with colors.
   - The pane's size.
4. **Claude Code detection.** The app detects Claude Code in a marked pane and additionally records:
   - The current Claude Code session id, as an attribute of the capture, not its key. A pane's session id changes during its life (for example when the conversation is cleared), so the recorder re-reads it continuously and always holds the latest.
   - Where it was launched from, which decides how restore starts it (see Restore). A worktree at `<repo>/.claude/worktrees/<name>` is a standard worktree: its capture records the repo root and the worktree name. Any other directory, including worktrees in non-standard locations, is recorded as just its working directory.
5. **Capture lifetime.** Recording continues until the server dies, the pane closes, the user stops capturing the pane, or the app stops running. The capture then remains in the store as a finished, restorable record.
6. **Server restart.** The user restarts the server out of band. The app detects the loss of the connection, keeps every capture, and reconnects automatically when a new server is available. Captures from the old server are not touched by the new one.
7. **Interactive restore.** In the UI the user picks a capture and a session on the connected server. The capture's origin plays no part. Restore:
   - creates a new window in the chosen session, holding the restored pane, with the recorded working directory,
   - loads the recorded history into it as real scrollback,
   - for a Claude Code capture in a standard worktree, starts Claude Code from the recorded repo root with `--resume <session id> --worktree <name>`,
   - for any other Claude Code capture, including worktrees in non-standard locations, runs Claude Code in the recorded working directory with `--resume <session id>` and no `--worktree`,
   - for any other capture, relaunches the recorded command line.

   A restored pane is a new live pane and is not captured automatically. If the user marks it, that is a new capture with a new capture id.
8. **Failures are loud and specific.** A pane that cannot be captured or restored is shown as failed, with the reason.
9. **Captures stay readable without the app.** History is stored as plain files that `cat` or `less -R` can display, and the manifest is versioned JSON. A capture is never half-written.

## Findings and unverified assumptions

Observed on this machine:

- Each running Claude Code instance writes `~/.claude/sessions/<pid>.json` containing its `pid`, current `sessionId`, `cwd`, a `tmux` field naming the pane it runs in (for example `rich-js:@516.%523`), and a `formerNames` list recording earlier session ids. That is the detection and session-id source: match a marked pane to the instance whose `tmux` field names it. Session transcripts live under `~/.claude/projects/<encoded path>/`.

Not yet verified:

- That the library's client exposes what capture needs through its structured API: pane topology, history capture with escapes, live pane output, and the pane's process. Any gap is a library change, not a workaround in the app.
- How best to load recorded history into a new pane as real scrollback, so the user scrolls up and sees the old output above the live session.

## Build order

1. App scaffold on the library's Electron connector, with a test harness that runs against an isolated tmux socket.
2. Attach and live view of sessions, windows and panes.
3. Marking panes, and continuous capture into the store.
4. Detecting server loss and reconnecting automatically.
5. Claude Code detection: session id and launch location (standard worktree or not).
6. Interactive restore into a new window of a chosen session.
