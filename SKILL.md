---
name: debug-openclaw
description: Debug OpenClaw session issues, gateway problems, and runtime errors. Use when troubleshooting why sessions aren't responding, gateway won't start, skills fail to load, browser automation breaks, channels disconnect, or model/auth configuration is incorrect. Provides architecture context and guides systematic investigation through config files, session logs, and source code.
---

# Debug OpenClaw Session

Guide for systematically investigating OpenClaw issues. This skill teaches you WHERE to look and WHAT to read — you analyze the content yourself.

## Investigation Flow

When debugging any OpenClaw issue, follow this order:

1. **Understand the symptom** — What is the user experiencing?
2. **Check the right files** — Use the sections below to find the relevant files
3. **Read and analyze** — Open the files, read the content, form your diagnosis
4. **Trace through source** — If needed, read the source code to understand behavior

## Architecture Overview

Read [references/architecture.md](references/architecture.md) for full details. Key concepts:

* **Gateway** — HTTP + WebSocket server (default port 18789) that manages all agent sessions
* **Session** — Each conversation is a `.jsonl` file storing the full transcript
* **Agent** — An AI agent instance with its own config, sessions, identity
* **Pi-Embedded Runner** — The core engine that sends messages to AI providers and handles responses
* **Skills** — Modular extensions loaded from SKILL.md files and injected into the system prompt

## Where to Look by Problem Type

### Session Issues

**Goal: Understand what happened in a conversation**

| What to check | Where to find it |
|----------------|-----------------|
| Session transcripts | `~/.openclaw/agents/<agentId>/sessions/*.jsonl` |
| Session index | `~/.openclaw/agents/<agentId>/sessions/sessions.json` |
| Session key mapping | Read `sessions.json` to find which `.jsonl` maps to which chat |
| What agent ID to use | System prompt `Runtime` line contains `agent=<id>` |

**How to read a session JSONL:**

* Each line is a JSON object
* `type: "session"` = session metadata (model, auth profile, timestamp)
* `type: "message"` + `message.role: "user"` = user messages
* `type: "message"` + `message.role: "assistant"` = AI responses
* `message.content[].type: "toolCall"` = tool invocations
* `message.content[].type: "toolResult"` = tool results (check `isError`)
* `message.usage.cost.total` = cost per response

**Source code to read for deeper understanding:**

* `src/gateway/session-utils.ts` — Session resolution, store loading
* `src/gateway/session-utils.fs.ts` — Session file I/O (read/write transcript)
* `src/agents/session-transcript-repair.ts` — How transcript corruption is handled
* `src/agents/session-write-lock.ts` — Session concurrency control

---

### Model & Auth Issues

**Goal: Verify model configuration and API authentication**

| What to check | Where to find it |
|----------------|-----------------|
| Main config | `~/.openclaw/openclaw.json` — look at `agents.<id>.model`, `agents.<id>.modelProvider` |
| Model overrides | `~/.openclaw/models.json` (if exists) — custom provider/model definitions |
| Auth profiles | Config `agents.<id>.authProfiles` — multiple API key rotation |
| Environment API keys | `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `GOOGLE_API_KEY` etc. |
| Credentials store | `~/.openclaw/credentials/` |
| Session-level model info | In `.jsonl` first line: `type: "session"` contains active model |

**What to look for in config:**

* `model` — The model ID (e.g., `claude-sonnet-4-20250514`)
* `modelProvider` — Provider name (e.g., `anthropic`, `openai`, `google`)
* `authProfiles` — Array of API key configs for rotation on failure
* `modelFallback` — Fallback model if primary fails

**Source code to read for deeper understanding:**

* `src/agents/model-selection.ts` — How the model is chosen
* `src/agents/model-auth.ts` — How auth is resolved for each request
* `src/agents/model-fallback.ts` — Fallback logic on provider failure
* `src/agents/auth-profiles/` — Auth profile rotation mechanism
* `src/agents/models-config.providers.ts` — Provider endpoint/URL configuration
* `src/agents/cli-credentials.ts` — Credential storage and retrieval

---

### Channel Issues

**Goal: Verify messaging channel (Telegram/Discord/Slack/etc.) connectivity**

| What to check | Where to find it |
|----------------|-----------------|
| Channel config | `~/.openclaw/openclaw.json` — `telegram`, `discord`, `slack` sections |
| Channel status | Run `openclaw channels status --probe` |
| Bot tokens | Config fields or env vars: `DISCORD_BOT_TOKEN`, `TELEGRAM_BOT_TOKEN`, etc. |
| Webhook settings | Channel-specific config sections |
| Gateway logs | `/tmp/openclaw-gateway.log` or macOS unified logs via `scripts/clawlog.sh` |

**What to look for in config:**

* Each channel has its own top-level config section (e.g., `telegram: { botToken: "..." }`)
* Check if token is present and not empty
* Check if the channel is enabled (not explicitly disabled)
* Check gateway mode: `gateway.mode` should be `"local"` for local setup

**Source code to read for deeper understanding:**

* `src/gateway/server-channels.ts` — Channel connection management
* `src/gateway/channel-health-monitor.ts` — Health check logic
* `src/telegram/`, `src/discord/`, `src/slack/` — Channel-specific code

---

### Skill Loading Issues

**Goal: Verify a skill is correctly loaded and available to the agent**

| What to check | Where to find it |
|----------------|-----------------|
| Skill status | Run `openclaw skills status` |
| SKILL.md frontmatter | Open the SKILL.md, check `name` and `description` between `---` markers |
| Skill file size | Must be ≤ 256KB per SKILL.md |
| Skill directories scanned | `.agents/skills/`, `.agent/skills/`, `_agents/skills/`, `_agent/skills/` in workspace |
| Managed skills dir | `~/.openclaw/skills/` |
| Skills config | `~/.openclaw/openclaw.json` → `skills` section |

**Rules the loader enforces:**

* SKILL.md must have valid YAML frontmatter with `name` and `description`
* `name`: lowercase, digits, hyphens only, ≤64 chars
* Max 150 skills in prompt, max 30,000 total chars
* Max 300 candidates scanned per root directory
* Skills are deduplicated by name (workspace skills win over managed/bundled)

**Source code to read for deeper understanding:**

* `src/agents/skills/workspace.ts` → `loadSkillEntries()` — The main loading pipeline
* `src/agents/skills/frontmatter.ts` — How YAML frontmatter is parsed and validated
* `src/agents/skills/types.ts` — `SkillEntry`, `SkillSnapshot` type definitions
* `src/agents/skills/config.ts` — Skill config resolution
* `src/agents/system-prompt.ts` → `buildSkillsSection()` — How skills are injected into the system prompt

---

### Gateway Issues

**Goal: Understand why the gateway won't start or behaves unexpectedly**

| What to check | Where to find it |
|----------------|-----------------|
| Gateway config | `~/.openclaw/openclaw.json` → `gateway` section |
| Default port | 18789 (check `gateway.port` in config) |
| Gateway logs | `/tmp/openclaw-gateway.log` |
| macOS logs | Run `scripts/clawlog.sh` |
| Port occupancy | `lsof -i :18789` |
| Running processes | `pgrep -f "openclaw.*gateway"` |
| Node version | `node --version` (must be 22+) |
| Doctor diagnostic | `openclaw doctor` |

**Source code to read for deeper understanding:**

* `src/gateway/server.impl.ts` → `startGatewayServer()` — Main entry point
* `src/gateway/server-http.ts` — HTTP route registration
* `src/gateway/config-reload.ts` — Config hot-reload mechanism
* `src/config/paths.ts` — How paths (config, state dir) are resolved
* `src/infra/ports.ts` — Port availability checking

---

### Browser / Playwright Issues

**Goal: Understand browser automation failures**

| What to check | Where to find it |
|----------------|-----------------|
| Chrome debug endpoint | `curl -s http://localhost:9222/json/list` |
| Chrome processes | `pgrep -f "chrome\|chromium"` |
| Target/session ID | The `/json/list` response shows current page targets |

**Key concept:** Chrome's `targetId` changes when pages navigate. The code re-fetches from `/json/list` to handle this.

**Source code to read for deeper understanding:**

* `src/browser/` — All browser automation code
* `src/gateway/server-browser.ts` — Gateway's browser session management

---

### Config System

**Goal: Understand how configuration is loaded and which file is active**

| What to check | Where to find it |
|----------------|-----------------|
| Active config | `~/.openclaw/openclaw.json` (JSON5 format, supports comments) |
| Legacy configs | `~/.clawdbot/clawdbot.json`, `~/.moldbot/moldbot.json` |
| Override env vars | `OPENCLAW_CONFIG_PATH`, `OPENCLAW_STATE_DIR`, `OPENCLAW_HOME` |
| Config audit log | `~/.openclaw/config-audit.jsonl` |
| Config schema | Read `src/config/schema.ts` for all valid config fields |

**Resolution order:**

1. `OPENCLAW_CONFIG_PATH` env var (explicit override)
2. `$OPENCLAW_STATE_DIR/openclaw.json`
3. Legacy paths (auto-migrated)

**Source code to read for deeper understanding:**

* `src/config/io.ts` — Config read/write logic, JSON5 parsing
* `src/config/paths.ts` — Path resolution logic
* `src/config/defaults.ts` — Default config values
* `src/config/validation.ts` — Config validation
* `src/config/schema.ts` + `src/config/zod-schema.ts` — Full config schema

## Useful CLI Commands

These are NOT diagnostic scripts — they just help you see raw data:

```bash
# See gateway status
openclaw channels status --probe

# Run built-in diagnostic
openclaw doctor

# List loaded skills
openclaw skills status

# Check version
openclaw --version
```
