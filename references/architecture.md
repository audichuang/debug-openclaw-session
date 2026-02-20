# OpenClaw Architecture Reference

When you need to understand a specific part of OpenClaw, read the relevant section below and then go read the actual source files mentioned.

## Table of Contents

* [How OpenClaw Starts](#how-openclaw-starts)
* [Gateway: The Central Hub](#gateway-the-central-hub)
* [How a Message Becomes a Response](#how-a-message-becomes-a-response)
* [Session Storage](#session-storage)
* [Skills System](#skills-system)
* [Config System](#config-system)
* [Model & Auth System](#model--auth-system)
* [Key Source Files Index](#key-source-files-index)

***

## How OpenClaw Starts

To understand the startup sequence, read these files in order:

1. **`src/entry.ts`** — The real entry point. Handles Node warning suppression (respawns if needed), applies CLI profile flags, then loads `src/cli/run-main.ts`
2. **`src/cli/run-main.ts`** — Calls `buildProgram()` to set up the Commander.js command tree, then `parseAsync()` to dispatch
3. **`src/cli/program.ts`** (and `src/cli/program/`) — Defines all CLI subcommands (`gateway`, `config`, `skills`, etc.)

For the gateway specifically:
4\. **`src/cli/gateway-cli/`** — The `openclaw gateway run` command handler
5\. **`src/gateway/server.impl.ts`** → `startGatewayServer()` — The main gateway startup function

***

## Gateway: The Central Hub

The gateway is an HTTP + WebSocket server. To understand its internals, read:

* **`src/gateway/server.impl.ts`** → `startGatewayServer()` — See what components it initializes:
  * HTTP server via `server-http.ts`
  * WebSocket via `ws-log.ts` and `server-ws-runtime.ts`
  * Node registry (`node-registry.ts`) — tracks connected devices
  * Channel connections (`server-channels.ts`) — Telegram, Discord, etc.
  * Config hot-reloader (`config-reload.ts`)
  * Cron scheduler (`server-cron.ts`)
  * Exec approval manager (`exec-approval-manager.ts`)

* **`src/gateway/server-http.ts`** — All HTTP route definitions. Read this to understand the REST API surface.

* **`src/gateway/server-chat.ts`** — How incoming chat messages are processed before reaching the AI provider.

***

## How a Message Becomes a Response

This is the most important flow to understand for debugging. Read these in order:

1. **Message arrives** — Through a channel (Telegram, Discord, etc.) or CLI
2. **`src/gateway/server-chat.ts`** — `handleChatMessage()` determines the session key and routes the message
3. **Session key resolution** — `src/gateway/session-utils.ts` → `resolveGatewaySessionStoreTarget()` finds the right agent and session file
4. **System prompt construction** — `src/agents/system-prompt.ts` → `buildAgentSystemPrompt()` assembles identity, tools, skills, memory, runtime info
5. **AI execution** — `src/agents/pi-embedded-runner/` directory contains the runner. Key file: `src/agents/pi-embedded-runner/runs.ts`. It:
   * Builds the API request (messages history + system prompt)
   * Selects model and auth profile
   * Sends to provider (Anthropic/OpenAI/Google/etc.)
6. **Stream processing** — `src/agents/pi-embedded-subscribe.ts` handles the streaming response:
   * Text → assistant messages
   * Tool calls → executed via tool handlers → results fed back
   * If context window is full → triggers compaction (`src/agents/compaction.ts`)
7. **Transcript append** — Results written to the session `.jsonl` file

***

## Session Storage

To understand how sessions are stored and managed:

* **Location**: `~/.openclaw/agents/<agentId>/sessions/`
* **Index file**: `sessions.json` — maps session keys (like `telegram:123456`) to session IDs
* **Transcript files**: `<session-id>.jsonl` — one JSON object per line, append-only

**To understand the JSONL format**, open any `.jsonl` file and read a few lines. Each line has:

* `type` — `"session"` (metadata) or `"message"` (conversation turn)
* `timestamp` — ISO format
* `message.role` — `"user"`, `"assistant"`, or `"toolResult"`
* `message.content[]` — array of content blocks (text, toolCall, toolResult, thinking)

**Source code:**

* `src/gateway/session-utils.ts` — Session resolution, listing, key canonicalization
* `src/gateway/session-utils.fs.ts` — File-level read/write operations
* `src/config/sessions.ts` — Session store (sessions.json) management
* `src/agents/session-write-lock.ts` — Concurrency: how write locks prevent corruption
* `src/agents/session-transcript-repair.ts` — How corrupted transcripts are repaired

***

## Skills System

To understand how skills are loaded and made available:

1. **Scanning** — `src/agents/skills/workspace.ts` → `loadSkillEntries()` scans these directories:
   * Workspace: `.agents/skills/`, `.agent/skills/`, `_agents/skills/`, `_agent/skills/`
   * Managed: `~/.openclaw/skills/` (installed via `openclaw skills install`)
   * Bundled: `skills/` in the OpenClaw distribution

2. **Parsing** — `src/agents/skills/frontmatter.ts` extracts YAML frontmatter (`name`, `description`) from each SKILL.md

3. **Filtering** — `filterSkillEntries()` applies config-level skill filters and eligibility checks

4. **Prompt injection** — `buildWorkspaceSkillsPrompt()` → formats skill metadata for the system prompt. Only `name` + `description` are always in context; the SKILL.md body is loaded on demand.

**Limits** (defined in `workspace.ts`):

* `DEFAULT_MAX_SKILLS_IN_PROMPT = 150`
* `DEFAULT_MAX_SKILLS_PROMPT_CHARS = 30_000`
* `DEFAULT_MAX_SKILL_FILE_BYTES = 256_000` (256KB per SKILL.md)
* `DEFAULT_MAX_CANDIDATES_PER_ROOT = 300`

**To debug skill loading**, read `loadSkillEntries()` in `workspace.ts` — it logs via `skillsLogger` and tracks errors per source.

***

## Config System

To understand how config is loaded:

1. **Path resolution** — `src/config/paths.ts` → `resolveConfigPath()`:
   * First checks `OPENCLAW_CONFIG_PATH` env var
   * Then `$OPENCLAW_STATE_DIR/openclaw.json`
   * Then legacy paths (`~/.clawdbot/`, etc.)

2. **Reading** — `src/config/io.ts` → uses JSON5 parser (supports comments, trailing commas)

3. **Schema validation** — `src/config/schema.ts` + `src/config/zod-schema.ts` define all valid fields

4. **Type definitions** — `src/config/types.ts` re-exports from `types.*.ts` files (one per feature area: `types.agents.ts`, `types.gateway.ts`, `types.discord.ts`, etc.)

5. **Defaults** — `src/config/defaults.ts` contains all default values

6. **Hot-reload** — `src/gateway/config-reload.ts` watches the config file and applies changes without restart

***

## Model & Auth System

To understand model selection and authentication:

1. **Selection** — `src/agents/model-selection.ts` → determines which model to use based on config, overrides, and session-level settings

2. **Authentication** — `src/agents/model-auth.ts` → resolves API key/token for the chosen provider

3. **Auth profiles** — `src/agents/auth-profiles/` directory → supports multiple API keys that rotate on failures or rate limits

4. **Provider config** — `src/agents/models-config.providers.ts` → configures base URLs, token exchange, provider-specific settings. Read this to understand custom provider setup.

5. **Fallback** — `src/agents/model-fallback.ts` → when primary model fails, selects fallback model

6. **Credentials storage** — `src/agents/cli-credentials.ts` → where and how credentials are persisted (`~/.openclaw/credentials/`)

***

## Key Source Files Index

Quick reference for finding specific functionality:

| Purpose | Read this file |
|---------|---------------|
| CLI entry | `src/entry.ts` |
| Gateway server | `src/gateway/server.impl.ts` |
| HTTP routes | `src/gateway/server-http.ts` |
| Chat handling | `src/gateway/server-chat.ts` |
| Session utilities | `src/gateway/session-utils.ts` |
| Session file I/O | `src/gateway/session-utils.fs.ts` |
| System prompt | `src/agents/system-prompt.ts` |
| AI runner | `src/agents/pi-embedded-runner/` |
| Stream handler | `src/agents/pi-embedded-subscribe.ts` |
| Compaction | `src/agents/compaction.ts` |
| Skill loading | `src/agents/skills/workspace.ts` |
| Skill frontmatter | `src/agents/skills/frontmatter.ts` |
| Config read/write | `src/config/io.ts` |
| Config paths | `src/config/paths.ts` |
| Config schema | `src/config/schema.ts`, `src/config/zod-schema.ts` |
| Config defaults | `src/config/defaults.ts` |
| Config types | `src/config/types.ts`, `src/config/types.*.ts` |
| Model selection | `src/agents/model-selection.ts` |
| Model auth | `src/agents/model-auth.ts` |
| Auth profiles | `src/agents/auth-profiles/` |
| Provider config | `src/agents/models-config.providers.ts` |
| Model fallback | `src/agents/model-fallback.ts` |
| Credentials | `src/agents/cli-credentials.ts` |
| Tool policies | `src/agents/tool-policy.ts` |
| Bash tools | `src/agents/bash-tools.exec.ts` |
| Session repair | `src/agents/session-transcript-repair.ts` |
| Session write lock | `src/agents/session-write-lock.ts` |
| Sub-agents | `src/agents/subagent-registry.ts`, `src/agents/subagent-spawn.ts` |
| Browser | `src/browser/`, `src/gateway/server-browser.ts` |
| Channels | `src/gateway/server-channels.ts` |
| Channel health | `src/gateway/channel-health-monitor.ts` |
| Config reload | `src/gateway/config-reload.ts` |
| Config validation | `src/config/validation.ts` |
| Logging | `src/logging/` |
