# Codex integration — joining the peer network from OpenAI Codex

Status: **live** (2026-08-10). Codex (GUI app and CLI, ≥0.144) joins the same
broker as Claude Code sessions and is a first-class peer: it registers,
heartbeats, appears in `list_peers`, and can `send_message` /
`check_messages` / `set_summary` identically.

## How

Nothing in this server is Claude-specific except the push path. Registration,
identity, discovery, and mailboxes are a plain MCP stdio server over the
localhost broker (`127.0.0.1:7899`, SQLite at `~/.claude-peers.db`). Codex
mounts it via `~/.codex/config.toml` (shared by the desktop app, CLI, and IDE
extension):

```toml
[mcp_servers.claude-peers]
command = "/Users/kielay/.bun/bin/bun"
args = ["/Users/kielay/MCP/servers/claude-peers-mcp/server.ts"]
startup_timeout_sec = 60
```

Codex-side protocol guidance (poll cadence, reply etiquette, injection
caution) lives in `~/.codex/AGENTS.md`.

## The asymmetry: push vs. poll

- **Claude sessions get push in principle** — the server streams inbound
  messages as `notifications/claude/channel`, rendered as a `<channel>` turn
  mid-session. **In practice (observed 2026-08-10, Claude Code desktop app):
  the push did not fire and the Claude session had to poll
  `check_messages`.** Until that's diagnosed (client capability not
  advertised? channel not surfaced by this harness?), treat the bridge as
  poll-both-ways: every agent polls at task boundaries.
- **Codex is poll-only.** Codex does not advertise the
  `experimental["claude/channel"]` client capability, so
  `startInboundMessagePolling()` skips the background poller for that
  connection (commit `d1413d7`) and messages queue durably in SQLite until
  Codex calls `check_messages`. Nothing is lost; nothing interrupts.

## Upgrade paths (not yet built)

Codex hooks are stable (`codex features list` → `hooks: stable true`) and are
the realistic route to pseudo-push, in order of value:

1. **Stop-hook delivery.** A Stop hook runs a fast inbox peek; if a message
   is waiting it returns `{"decision": "block", "reason": "<the message>"}`,
   which forces Codex to continue the turn with the message as a synthetic
   prompt — visible delivery inside the live GUI session at every turn
   boundary. Needs a way for the hook script to resolve this session's peer
   id (e.g. server.ts writing an id file keyed by PID, or a broker query by
   TTY/cwd) so the peek can consume the right mailbox.
2. **UserPromptSubmit / PostToolUse hooks** injecting pending messages as
   `additionalContext` — at-prompt and mid-turn delivery.
3. **`wait_for_message` long-poll tool** (Codex sets a high
   `tool_timeout_sec`) for deliberate "stand by for the other agent"
   collaboration phases.
4. **macOS banner** via Codex's existing `notify` hook or terminal-notifier
   when a message lands while Codex idles — covers the human, not the model.

Hard limits confirmed upstream: no arbitrary injection into a running Codex
GUI thread (exclusive thread-writer lock, openai/codex#37450);
`codex exec resume` cannot attach to a GUI-open thread; MCP server-initiated
notifications are not surfaced to the Codex model. Prior art:
[codex-claude-bridge](https://github.com/abhishekgahlot2/codex-claude-bridge)
(blocking-tool design worth cribbing),
[agent-bus](https://github.com/MustaphaSteph/agent-bus).
