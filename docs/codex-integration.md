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

## Hook-based delivery (BUILT 2026-08-10) — both harnesses

`hooks/inbox-hook.ts` drains this session's mailbox and injects messages
into model context. Two modes: `context` (print messages on stdout —
UserPromptSubmit adds them to context) and `stop` (emit
`{"decision":"block","reason":<messages>}` so the harness runs one more
turn to handle them; silent when the mailbox is empty).

**Self-addressing:** server.ts now writes
`~/.claude-peers/sessions/<harness-pid>.json` at registration (removed on
clean exit). The hook walks its own ancestor PIDs to find that file — hook
and MCP server share the harness as a common ancestor. `--peer-id <id>`
bypasses resolution for testing. Draining uses `/poll-messages`
(single-consumption), so hooks and `check_messages` never double-deliver.

**Wiring:** Claude Code — `~/.claude/settings.json` hooks on
`UserPromptSubmit` (context) and `Stop` (stop), exec-form, 10 s timeout.
Codex — `~/.codex/config.toml` `[[hooks.UserPromptSubmit]]` and
`[[hooks.Stop]]`. Restart sessions to activate: the session-map file only
exists for servers started after this change, and Codex prompts once to
trust the new hooks.

**Loop caution:** two agents with stop-block hooks replying unconditionally
can ping-pong; the injected envelope instructs reply-only-when-needed.

Remaining (not built): a `wait_for_message` long-poll tool for deliberate
stand-by phases; macOS banner on message arrival while idle (the human's
notification, via terminal-notifier or Codex `notify`).

Hard limits confirmed upstream: no arbitrary injection into a running Codex
GUI thread (exclusive thread-writer lock, openai/codex#37450);
`codex exec resume` cannot attach to a GUI-open thread; MCP server-initiated
notifications are not surfaced to the Codex model. Prior art:
[codex-claude-bridge](https://github.com/abhishekgahlot2/codex-claude-bridge)
(blocking-tool design worth cribbing),
[agent-bus](https://github.com/MustaphaSteph/agent-bus).
