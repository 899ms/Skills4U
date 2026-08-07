---
name: orchard
description: "Use the local Orchard app to interact with macOS Apple apps and services: Calendar, Reminders, Clock, Mail, Contacts, Notes, Music, Weather, Messages, Location/Maps, and Apple Shortcuts. Two execution paths reach the same running Orchard.app: the `orchard` CLI (default — use this if you have a Bash/shell tool, e.g. Claude Code, Codex CLI, Cursor) and a stdio MCP server, `orchard mcp`, exposed as prefixed MCP tools (fallback for sandboxes that cannot run the macOS CLI, e.g. Claude Cowork, bridged through Claude Desktop). Use when a task asks to read or manage local calendar events, reminders, Apple Mail, contacts, notes, iMessage/SMS, Apple Music playback/library, weather, current time/timezones, geocoding, routes, current location, or local Shortcuts."
metadata:
  version: "0.7.3"
  updated: "2026-08-07"
  tested_with:
    orchard_app: "0.6.2 (17)"
    orchard_cli: "0.6.2"
---

# Orchard

Orchard exposes 52 Apple-app tools (Calendar, Reminders, Mail, Notes, Messages, Music, Weather, Contacts, Clock, Location/Maps, Shortcuts) through one running Orchard.app, reachable over two transports that carry the same tool set. **Default to the CLI.** Read "Which Channel" below first — it costs you nothing to check and prevents wasted turns.

## Which Channel

Decide by looking at your own tool list first. Having a Bash tool is NOT enough to prove you can reach the CLI — sandboxed agents (e.g. Claude Cowork) have a Bash that runs inside an isolated Linux VM where the macOS CLI does not exist, while Orchard's MCP tools are still bridged in by the host.

- **You have a Bash/shell tool** (Claude Code, Codex CLI, Cursor, or any terminal-capable agent — the large majority of sessions) → try the **CLI first**. Continue to "Quick Start" below. It is cheaper than MCP (this file plus one on-demand reference beats preloading 52 MCP schemas), and it self-documents via `--help`.
- **You have no Bash/shell tool at all** → skip straight to `references/mcp-tools.md` and call the Orchard MCP tools (names ending in `__calendar_info`, `__mail_read`, etc.) directly. "Quick Start", "Known Pitfalls", and "Common Workflows" below are written for the CLI and do not apply to you. "Core Rules" still applies — read it, it describes the tools, not the transport.

If you have Bash but `scripts/resolve-orchard.sh --route` reports `missing_app`, check your tool list for Orchard MCP tools before concluding anything:

- **Orchard MCP tools ARE in your tool list** → your shell is sandboxed away from the user's Mac (Claude Cowork's Linux VM is the typical case), but the host has bridged Orchard's MCP server in. Switch to the MCP channel: read `references/mcp-tools.md` and use those tools. Do not tell the user anything is broken — nothing is.
- **No Orchard MCP tools either** → Orchard.app genuinely isn't installed or running on this Mac. Stop and tell the user to install/launch Orchard.app. Do not hunt further — every channel proxies to that same app.

## Quick Start (CLI)

Resolve the CLI path first, then call that path for every command.

Run the bundled resolver first. If your current working directory is not this skill directory, use the absolute path to `scripts/resolve-orchard.sh`.

```bash
ORCHARD_BIN="$(./scripts/resolve-orchard.sh --bin)"
ORCHARD_ROUTE="$(./scripts/resolve-orchard.sh --route)"
ORCHARD_APP_PATH="$(./scripts/resolve-orchard.sh --app)"
"$ORCHARD_BIN" --version
```

Route rules:

- `global_cli`: use the globally installed `orchard` command. Do not reject it because it points at a Debug app; development machines often do that.
- `bundle_cli`: use the bundled `orchard-cli` inside `Orchard.app`. Continue the task. Optionally tell the user they can install the CLI tool from Orchard later for shell-wide `orchard` access.
- `missing_app`: see "Which Channel" above — check your tool list for Orchard MCP tools before telling the user anything is broken.

Prefer `--json` for machine-readable output. Orchard returns an outer JSON envelope:

```json
{"output":"...string, object, or array...","success":true}
```

The `output` value may already be a JSON object/array, plain text, JSON text, or prose followed by JSON such as `Found 2 events:\n{...}`. If `output` is a string that contains JSON, strip text before the first JSON object/array and parse the nested payload before reasoning over records. (This envelope is a CLI-only convenience — see `references/mcp-tools.md` if you're on the MCP channel instead, where results look different.)

For the full command matrix, read `references/commands.md`.

## Known Pitfalls (CLI)

- Use `"$ORCHARD_BIN" <domain> <command> ...`, not a hard-coded `orchard`, unless `command -v orchard` was the selected route.
- Check leaf-command help with `"$ORCHARD_BIN" <domain> <command> --help` or `"$ORCHARD_BIN" help <domain> <command>`; both print the same output.
- In zsh loops, never pass a multi-word command through a scalar like `"$ORCHARD_BIN" $cmd`; zsh keeps it as one argument. Use an array, or `eval` only for trusted static command strings.
- Negative numeric values must use the equals form: `--lng1=-122.4194`, `--lon=-117.1201`. With a space (`--lng1 -122.4194`) the parser reads the value as a new flag and fails with a usage error; shell quoting does not help because quotes never reach argv.
- `orchard mail read` supports `--date-from`/`--date-to` (ISO 8601) and `--offset` for time windows and pagination. If the installed CLI rejects these flags (older builds), fall back to `--limit` plus local `date_sent` filtering and state possible truncation when results hit the limit.
- Treat help output and the installed CLI as source of truth. If this skill and CLI disagree, run `"$ORCHARD_BIN" <domain> <command> --help` and adapt.

## Core Rules

These describe the tools themselves, not either transport — they apply whether you're on the CLI or the MCP channel.

- Use ISO 8601 timestamps for all date ranges. Include timezone offsets when the user means local time, e.g. `2026-06-03T08:00:00+08:00`.
- Convert relative dates before calling Orchard. "Yesterday 08:00" must become a concrete timestamp.
- Before creating calendar events or reminders, list calendars/lists and choose the right writable target. Do not dump everything into a default list/calendar unless the user explicitly asks.
- Before update/delete/mark/cancel operations, read the target and capture its ID.
- Treat destructive operations as real local mutations. For deletes, bulk updates, sending mail/messages, and scheduled sends, confirm intent unless the user explicitly requested the action.
- Before running an Apple Shortcut, list or open the matching shortcut first. Use `shortcuts run` only with `--confirm` and `--reason` (CLI) / `confirm`+`reason` (MCP), and confirm with the user unless they explicitly requested that exact run.
- Never print huge email bodies or private data unnecessarily. Fetch summaries first, then read full content only for messages that matter.

## Common Workflows (CLI)

### Daily Context

```bash
"$ORCHARD_BIN" clock time --timezone Asia/Shanghai --json
"$ORCHARD_BIN" weather get --location Jinan --granularity daily --start-date YYYY-MM-DD --end-date YYYY-MM-DD --json
"$ORCHARD_BIN" calendar info --type events --from YYYY-MM-DDT00:00:00+08:00 --to YYYY-MM-DDT00:00:00+08:00 --json
"$ORCHARD_BIN" reminder info --type reminders --status incomplete --json
```

When summarizing reminders, filter stale/completed/noisy records yourself if Orchard returns more than requested.

### Mail Triage

```bash
"$ORCHARD_BIN" mail refresh --json
"$ORCHARD_BIN" mail read --type list --limit 100 --json
"$ORCHARD_BIN" mail read --type list --date-from 2026-07-01T00:00:00+08:00 --date-to 2026-07-15T00:00:00+08:00 --limit 100 --json
"$ORCHARD_BIN" mail read --type content --message-id MESSAGE_ID --json
```

Mail listing supports `--limit`, `--mailbox`, `--account`, `--offset` for pagination, and `--date-from`/`--date-to` (ISO 8601) for time windows. If the result count equals the limit, page with `--offset` or say the scan may be truncated instead of pretending coverage is complete. On older builds that reject the date/offset flags, fall back to `--limit` plus local `date_sent` filtering.

Only fetch full email bodies for likely important mail: accounts, billing, support, customer, legal, platform review, security, collaboration, or anything the user asked to inspect.

### Calendar And Reminder Writes

```bash
"$ORCHARD_BIN" calendar info --type calendars --json
"$ORCHARD_BIN" calendar create --title "Call" --start 2026-06-03T15:00:00+08:00 --end 2026-06-03T15:30:00+08:00 --calendar-id CALENDAR_ID --alarms 15 --json

"$ORCHARD_BIN" reminder info --type lists --json
"$ORCHARD_BIN" reminder create --title "Follow up" --list-id LIST_ID --due-date 2026-06-03T18:00:00+08:00 --priority 5 --json
```

Priority is `0`-`9`, but **lower is more urgent**: `0` = none, `1` = high, `5` = medium, `9` = low. Mark complete with:

```bash
"$ORCHARD_BIN" reminder update --reminder-id REMINDER_ID --completed true --json
```

### Notes And Contacts

Search before creating duplicates:

```bash
"$ORCHARD_BIN" contacts search --query "Alice" --limit 10 --json
"$ORCHARD_BIN" notes search --query "project name" --limit 20 --json
```

Notes content is HTML for create/update. If the user gives Markdown, convert it to simple HTML first.

### Messages

```bash
"$ORCHARD_BIN" messages read --type chats --query "search term" --limit 20 --json
"$ORCHARD_BIN" messages read --type messages --chat "+15551234567" --limit 50 --json
"$ORCHARD_BIN" messages send --to "+15551234567" --text "Text here" --service iMessage --json
```

Confirm before sending unless the user directly instructs sending exact text.

### Shortcuts

```bash
"$ORCHARD_BIN" shortcuts list --summary --json
"$ORCHARD_BIN" shortcuts open --name "Shortcut name" --json
"$ORCHARD_BIN" shortcuts run --name "Shortcut name" --input "Text" --confirm --reason "User requested this exact shortcut run" --json
```

Prefer `shortcuts list --summary` before running. Use `--input-path` and `--output-path` for file handoff.

## Troubleshooting (CLI)

- If a command asks for macOS privacy permissions, tell the user which app/service needs permission and retry after permission is granted.
- You normally never see `Orchard.app is not running`: CLI 0.6.2+ auto-launches the app on demand (`open -b tech.5km.orchard`, polls up to ~10s, retries once — both channels share this). If a command reports the app could not be started or did not become ready, or an older CLI still returns the plain "not running" error, start `"$ORCHARD_APP_PATH"` when it is non-empty (otherwise `open -a Orchard`), wait a few seconds, then retry once with the same `ORCHARD_BIN`. Set `ORCHARD_NO_AUTOLAUNCH=1` only when a script needs a fast, definitive failure instead of the launch wait.
- For persistent or confusing failures, run `"$ORCHARD_BIN" doctor` — read-only diagnostics covering the socket, the CLI symlink, Skill installs, MCP config entries, and CLI/app version drift (exits 1 when something fails). `"$ORCHARD_BIN" status --json` reports login/Pro state, per-feature switches, and macOS permission status in one call; neither command launches the app.
- If a tool errors with "'X' is currently disabled", the switch can be flipped with `"$ORCHARD_BIN" features enable <feature>` — ask the user before changing their settings. Pro-tier features still require a Pro subscription; the server enforces this either way.
- If a command returns `Invalid response from Orchard.app`, treat it as an Orchard.app bridge/permission/runtime issue, not necessarily a CLI syntax issue. Retry once, then ask the user to check that Orchard.app is running and has the relevant macOS privacy permissions.
- If Apple Mail, Calendar, or Reminders data looks stale, run the relevant refresh/list command again and state the timestamp of the scan.
- If `--json` output contains a long HTML email body, summarize only the relevant parts; do not paste the entire body.
- If a flag is not accepted, trust `"$ORCHARD_BIN" <domain> <command> --help` for the installed version and adapt.

## MCP Channel (No Bash — e.g. Claude Cowork)

If "Which Channel" routed you here (no Bash tool, or Bash is sandboxed away from this Mac while Orchard MCP tools are present): read `references/mcp-tools.md` now. It is self-contained — real tool names, required parameters, parameter-shape gotchas — because this channel has no `--help` to discover syntax from. Two things worth knowing before you start:

- **Tool name prefix varies by host.** Your tool list will show these tools as `mcp__<something>__<tool_name>` — e.g. `mcp__orchard__calendar_info` under a manual config, or `mcp__plugin_orchard_orchard__calendar_info` when Orchard ships as a plugin (Cowork's likely case). Never hardcode the middle segment: match on whatever ends in `__<tool_name>` against the names in `references/mcp-tools.md`.
- **Results are content blocks, not the CLI's JSON envelope.** `{"output":...,"success":...}` is a CLI-only wrapper. On MCP you get a content array plus a separate `isError` flag. When a tool emits a human-readable prefix followed by JSON, the two arrive as separate text blocks — the last block is clean, parseable JSON, so parse that one directly instead of stripping prose off the front. Plain-text results pass through as a single block unchanged.

Everything in **Core Rules** above still applies — it describes the tools, not the transport.
