# cc-tmux-lifeboat

An Electron app that attaches to a tmux server, records the panes you mark, and restores any recorded pane into any session on any server. Claude Code panes are detected and restored as resumed Claude Code sessions.

This document is the project's founding commitments. Code that disagrees with it is wrong; if the commitment is wrong, change it here first.

## The unit is the pane

A capture is a recording of one tmux pane. It is not tied to the session, window, pane id, or server it came from. Sessions and windows appear in the UI only to find panes and to choose a restore destination.

## Invariants

These hold by construction. The types carry them; nothing downstream re-checks them.

- **A capture's only identity is its capture id**, a UUID minted when the pane is marked. The live tmux pane id is a link to the running pane while recording; it is dropped when the capture ends and is never stored as identity or location.
- **A capture has exactly one authoritative record**, the manifest on disk. The UI, the recorder, and restore all read it; only the recorder writes it.
- **A Claude Code session id is an attribute of a capture, never its key.** It changes during a pane's life (clearing the conversation, for one). The recorder re-reads it continuously and the manifest holds the latest.
- **A capture is never half-written.** Every write to the store is atomic; a reader sees the previous complete state or the next one.
- **Captures outlive servers.** Server loss, reconnect, and a new server never modify an existing capture.
- **A restored pane is a new pane.** It is not captured. Marking it mints a new capture id.
- **The launch location is a closed set**: a standard worktree at `<repo>/.claude/worktrees/<name>`, recorded as repo root plus worktree name, or any other directory, recorded as its working directory. There is no third shape.

## What a capture records

- The launch command line (argv) and working directory.
- The pane's scrollback, with colors, as plain files that `cat` or `less -R` display.
- The pane's size.
- For a Claude Code pane: the current session id and the launch location.
- The capture's own history: when recording started, why it ended, and every failure, with its reason.

The manifest is versioned JSON. The store is readable without the app.

## Architecture

Built on `@promptctl/tmux-control-mode-js`, a normal package dependency.

- **Main process** owns the `TmuxClient`, the recorder, and the store. It is the only process with filesystem and tmux access. It runs for as long as the app does, watches the server, and writes captures throughout.
- **Renderer** is UI only. Everything it exchanges with main, tmux traffic and the app's own API alike, crosses on the typed RPC package over Electron IPC.
- **Typed RPC is its own package**, built in this repo and depending on nothing tmux-specific. Transports plug in beneath it; Electron IPC is the first. A channel is declared as one request union and one push union. The package owns framing, correlation ids, the typed error envelope, the single parse at the trust boundary, the proxy the compiler checks against the union, and push streams with backpressure. The app's channel is one declaration on it; the library's tmux bridge becomes another.
- **Dependencies flow one way**: renderer depends on main's channel declaration, main depends on the library, and both depend on the RPC package, which depends on neither. Nothing points back.
- **All tmux access is through the library's structured results.** The app does not shell out to `tmux` and does not parse raw tmux text. A gap in the library is a library change, not a workaround here.
- **Decisions are pure.** What to record, whether a pane is Claude Code, how to restore: each is a function from structured inputs to a description of the action. tmux calls and disk writes happen at the edge, where the description is performed.

## Behavior

1. **Attach and live view.** Choose a server. The UI shows its sessions, windows, and panes, updating as the server changes.
2. **Mark a pane.** A marked pane is recorded from then on; an unmarked pane is not. The user never declares what runs in a pane.
3. **Claude Code detection.** Each running Claude Code instance writes `~/.claude/sessions/<pid>.json` with its `pid`, `sessionId`, `cwd`, a `tmux` field naming its pane, and `formerNames`. A marked pane is Claude Code when an instance's `tmux` field names it. That file is the source of the session id and the launch location.
4. **Capture lifetime.** Recording ends when the server dies, the pane closes, the user stops it, or the app exits. The capture stays in the store as a finished, restorable record, with the reason it ended.
5. **Server restart.** The app detects connection loss, keeps every capture, and reconnects when a server is available.
6. **Restore.** The user picks a capture and a session on the connected server. Restore creates a new window in that session with the recorded working directory, loads the recorded scrollback into it, then:
   - a Claude Code capture in a standard worktree starts Claude Code from the repo root with `--resume <session id> --worktree <name>`;
   - any other Claude Code capture runs Claude Code in the working directory with `--resume <session id>`;
   - any other capture relaunches the recorded command line.
7. **Failures are loud and specific.** A pane that cannot be captured or restored is shown as failed, with the reason, and the reason is in the capture's record. Nothing falls back to a different source or defaults past an error.

## Observability

Nothing happens unseen. It is a property of the code's shape, built in the first commit, and no sink can add it later.

- **One work layer in main.** Every unit of work, an attach, a recorder tick, a restore, a store write, runs through one wrapper. The wrapper opens the event when the unit begins and closes it when the unit ends, on success, on failure, and on the exception nobody caught.
- **One event per unit of work**, as a type. Its default fields: `event`, `trace_id`, `service`, `started_at`, `duration_ms`, `outcome`, `error`, `sink`, `sink_error`, and `counts`. The project adds fields and never renames or duplicates one of these. `counts` holds every count a unit has, including the zeros: bytes written, panes seen, reconnect attempts, session-id re-reads.
- **One annotate call.** Code inside a unit adds a fact to the current event. It never emits. Call sites do not log.
- **One export edge.** Events leave the process through one checkpoint that speaks OTLP to an address read from configuration. With no address, or an unreachable one, it appends the event as JSONL to a local file, and when the address was set and failed, the record carries `sink_error`. The record always exists. Redaction happens at this edge and nowhere else.
- **Decisions are on the record.** Which launch-location shape was taken, which restore command was chosen, which server the client attached to, and why a capture ended.
- **The recorder has a heartbeat.** A recorder that never ran and a recorder that recorded nothing are different facts, and both are visible from the outputs alone.

## Testing

Suite speed is an architecture property. The seams below exist so almost every behavior is proven in process, in milliseconds.

- **Decision apart from effect.** What to record, whether a pane is Claude Code, which restore command to issue: each is a pure function from structured inputs to a description of the action. The shell loads inputs, calls the core, and performs the description. An `if` in the shell is logic that belongs in the core.
- **Ports the app owns**, in the app's vocabulary: the tmux server (topology, scrollback, live output, window creation), the Claude Code instance directory, the clock, the store, and the event export edge. Only adapters import the library, the filesystem, or the clock.
- **One real adapter and one in-memory fake per port**, kept beside the port in production code. The fake tmux server can lose its connection and come back. The fake clock advances on command.
- **One contract suite per port**, run against the fake in the unit layer and against the real adapter in the integration layer. The fake is trusted only while both pass.
- **One composition root.** The same assembly function builds the app from production ports or from fakes. Nothing constructs its own dependencies.
- **Layers.** Unit tests cover every behavior in this document against the fakes, with no socket, no disk, and a frozen clock. Integration tests exercise one adapter against a real tmux server on an isolated socket started once per run, and the real store against a temp directory. End-to-end smoke is capped at a handful: the app boots on the production root, attaches, marks, captures, and restores one pane.
- **The event is asserted like a return value.** A unit of work that runs under test produced its record with the counts the code claims.
- **The store format is tested from outside**: a written capture is read back with the plain-file tools the store promises to support.

A claim in this document with no test is not yet true.

## Not yet verified

- That the library's structured API exposes pane topology, scrollback with escapes, live output, and the pane's process. Any gap is a library change.
- How to load recorded scrollback into a new pane as real scrollback, so scrolling up shows the old output above the live session.

## Build order

Vertical slices. Each one cuts through every layer, ports, core, event layer, renderer, and tests, and leaves a working app behind. None finishes a layer ahead of the slice that needs it.

1. **Walking skeleton.** Boot on the composition root, attach, show panes live, one event per unit of work landing in a file.
2. **Capture and restore a plain pane.** Mark, record to the store, stop with a reason, restore by relaunching the command line into a chosen session.
3. **Claude Code panes.** Detect, track the session id, classify the launch location, restore with `--resume` and `--worktree`.
4. **Server loss and reconnect.** Captures finish with the reason and survive; the new server touches none of them.
5. **Real scrollback on restore.** Spike the mechanism, then replace the first approximation.
