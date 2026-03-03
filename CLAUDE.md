# NanoClaw

Personal Claude assistant accessible via WhatsApp. Single Node.js process connects to WhatsApp, routes messages to Claude Agent SDK running in containerized Linux VMs (Apple Container or Docker). Each group gets isolated filesystem, memory, and sessions.

See [README.md](README.md) for philosophy and setup. See [docs/REQUIREMENTS.md](docs/REQUIREMENTS.md) for architecture decisions. See [docs/SECURITY.md](docs/SECURITY.md) for the security model.

## Architecture Overview

```
WhatsApp (baileys) --> SQLite --> Polling loop --> Container (Claude Agent SDK) --> Response
                                      |
                                 IPC watcher <--> Container (MCP tools)
                                      |
                                 Task scheduler --> Container (scheduled tasks)
```

The host process handles WhatsApp connectivity, message storage, and container lifecycle. Agents run inside containers with the Claude Agent SDK, communicating back via filesystem-based IPC and an MCP server (`ipc-mcp-stdio.ts`).

## Key Files

### Host Process (runs on your machine)

| File | Purpose |
|------|---------|
| `src/index.ts` | Orchestrator: state machine, message polling loop, agent invocation, graceful shutdown |
| `src/channels/whatsapp.ts` | WhatsApp connection via baileys, auth, send/receive, typing indicators, group sync |
| `src/container-runner.ts` | Builds volume mounts, spawns container processes, streams output via sentinel markers |
| `src/group-queue.ts` | Per-group queue with global concurrency limit (`MAX_CONCURRENT_CONTAINERS`), retry with backoff |
| `src/ipc.ts` | Filesystem IPC watcher: processes messages and task commands from containers |
| `src/router.ts` | Message formatting (XML), outbound routing, `<internal>` tag stripping |
| `src/task-scheduler.ts` | Polls for due tasks, runs them in containers, updates next_run |
| `src/db.ts` | SQLite operations: messages, chats, tasks, sessions, router state, registered groups |
| `src/config.ts` | All constants: trigger pattern, paths, intervals, container settings |
| `src/types.ts` | TypeScript interfaces: RegisteredGroup, NewMessage, ScheduledTask, Channel, mount types |
| `src/env.ts` | Reads `.env` file without polluting `process.env` (keeps secrets off child processes) |
| `src/mount-security.ts` | Validates additional mounts against external allowlist at `~/.config/nanoclaw/mount-allowlist.json` |
| `src/logger.ts` | Pino logger with pretty printing, uncaught exception handlers |
| `src/whatsapp-auth.ts` | Standalone auth script: QR code or pairing code authentication |

### Container (runs inside Linux VM)

| File | Purpose |
|------|---------|
| `container/Dockerfile` | Node 22 + Chromium + agent-browser + claude-code, non-root user |
| `container/build.sh` | Build script for Apple Container (`container build`) |
| `container/agent-runner/src/index.ts` | Agent runner: reads stdin JSON, runs Claude Agent SDK queries in a loop, streams results via markers |
| `container/agent-runner/src/ipc-mcp-stdio.ts` | MCP server providing `send_message`, `schedule_task`, `list_tasks`, `pause_task`, `resume_task`, `cancel_task`, `register_group` tools |
| `container/skills/agent-browser/SKILL.md` | Browser automation skill using `agent-browser` CLI with Chromium |

### Data & State

| Path | Purpose |
|------|---------|
| `groups/{name}/` | Per-group workspace: CLAUDE.md memory, logs/, conversations/ |
| `groups/{name}/CLAUDE.md` | Per-group persistent memory (the agent reads and writes this) |
| `groups/global/CLAUDE.md` | Shared memory readable by all non-main groups (read-only) |
| `store/messages.db` | SQLite database (messages, tasks, sessions, state) |
| `store/auth/` | WhatsApp session credentials (never mounted into containers) |
| `data/ipc/{group}/` | Per-group IPC: messages/, tasks/, input/ directories |
| `data/sessions/{group}/.claude/` | Per-group Claude sessions and settings |
| `.env` | Secrets: `ANTHROPIC_API_KEY` or `CLAUDE_CODE_OAUTH_TOKEN`, `ASSISTANT_NAME` |

## Skills

Skills are Claude Code skills (`.claude/skills/*/SKILL.md`) that transform the codebase. Users run them with `/skill-name`.

| Skill | When to Use |
|-------|-------------|
| `/setup` | First-time installation: dependencies, auth, container setup, service config |
| `/customize` | Adding channels, integrations, changing behavior (interactive) |
| `/debug` | Container issues, logs, troubleshooting |
| `/add-telegram` | Add Telegram as a channel (alongside or replacing WhatsApp) |
| `/add-telegram-swarm` | Add agent swarm support to Telegram (each subagent gets its own bot) |
| `/add-gmail` | Add Gmail as a tool or full channel |
| `/add-voice-transcription` | Add WhatsApp voice message transcription via Whisper |
| `/convert-to-docker` | Switch from Apple Container to Docker (enables Linux support) |
| `/x-integration` | X (Twitter) integration: post, like, reply, retweet |

## Development

Run commands directly — don't tell the user to run them.

```bash
npm run dev          # Run with hot reload (tsx)
npm run build        # Compile TypeScript
npm run test         # Run tests (vitest)
npm run test:watch   # Run tests in watch mode
npm run typecheck    # Type-check without emitting
npm run format       # Format with prettier
npm run format:check # Check formatting
```

### Container rebuild

```bash
./container/build.sh                 # Build agent container image
```

The agent-runner source (`container/agent-runner/src/`) is mounted at runtime and recompiled on container startup via the entrypoint script. Host-side code changes take effect immediately without rebuilding the container. Only rebuild when changing the Dockerfile (system deps, global packages).

### Apple Container build cache

Apple Container's buildkit caches aggressively. `--no-cache` alone does NOT invalidate COPY steps. To force a truly clean rebuild:

```bash
container builder stop && container builder rm && container builder start
./container/build.sh
```

Verify after rebuild:
```bash
container run -i --rm --entrypoint wc nanoclaw-agent:latest -l /app/src/index.ts
```

### Service management (macOS)

```bash
launchctl load ~/Library/LaunchAgents/com.nanoclaw.plist
launchctl unload ~/Library/LaunchAgents/com.nanoclaw.plist
```

## Code Conventions

- **ESM modules** — `"type": "module"` in package.json, `.js` extensions in imports
- **TypeScript strict mode** — target ES2022, NodeNext module resolution
- **Single quotes** — enforced by prettier (`.prettierrc: { "singleQuote": true }`)
- **Pino logging** — structured JSON logs with pino-pretty, use `logger.info/warn/error/debug`
- **No configuration sprawl** — behavior changes go in code, not config files
- **Tests** — vitest, files colocated as `*.test.ts`, in-memory SQLite via `_initTestDatabase()`
- **Secrets handling** — secrets read from `.env` via `readEnvFile()`, never loaded into `process.env`, passed to containers via stdin JSON and deleted immediately

## How the Message Flow Works

1. WhatsApp messages arrive via baileys, stored in SQLite via `storeMessage()`
2. Polling loop in `startMessageLoop()` checks for new messages every 2s (`POLL_INTERVAL`)
3. Messages grouped by chat JID, trigger pattern checked for non-main groups
4. If an active container exists for the group, message is piped via IPC file; otherwise queued
5. `GroupQueue` manages concurrency (max 5 containers by default)
6. `processGroupMessages()` formats messages as XML, calls `runContainerAgent()`
7. Container receives input via stdin JSON (including secrets), runs Claude Agent SDK
8. Results stream back via `OUTPUT_START_MARKER`/`OUTPUT_END_MARKER` sentinel pairs in stdout
9. Host sends results to WhatsApp, strips `<internal>` tags

## IPC Protocol

Containers communicate with the host via filesystem-based IPC:

- **Outbound messages:** Container writes JSON to `/workspace/ipc/messages/` -> host sends to WhatsApp
- **Task operations:** Container writes JSON to `/workspace/ipc/tasks/` -> host processes (schedule, pause, resume, cancel, register_group)
- **Inbound messages:** Host writes JSON to `data/ipc/{group}/input/` -> container polls and feeds to active query
- **Close sentinel:** Host writes `_close` file -> container exits its query loop

## Security Model

- **Container isolation** — agents run in Linux VMs, can only access explicitly mounted directories
- **Mount allowlist** — stored at `~/.config/nanoclaw/mount-allowlist.json`, outside project root, never mounted into containers
- **Session isolation** — each group has its own `.claude/` directory, preventing cross-group data access
- **IPC authorization** — non-main groups can only send messages to their own chat and manage their own tasks
- **Credential handling** — only `CLAUDE_CODE_OAUTH_TOKEN` and `ANTHROPIC_API_KEY` exposed to containers; WhatsApp auth never mounted
- **Bash sanitization** — `PreToolUse` hook strips secret env vars from Bash commands inside containers
- **Non-root execution** — containers run as `node` user (uid 1000)

## Database Schema

SQLite at `store/messages.db`:
- `chats` — JID, name, last_message_time
- `messages` — id, chat_jid, sender, sender_name, content, timestamp, is_from_me, is_bot_message
- `scheduled_tasks` — id, group_folder, chat_jid, prompt, schedule_type, schedule_value, context_mode, next_run, status
- `task_run_logs` — task_id, run_at, duration_ms, status, result, error
- `router_state` — key/value pairs (last_timestamp, last_agent_timestamp)
- `sessions` — group_folder to session_id mapping
- `registered_groups` — jid, name, folder, trigger_pattern, container_config, requires_trigger

## Common Debugging

- Check container logs: `groups/{name}/logs/container-*.log`
- Check IPC errors: `data/ipc/errors/`
- View database: `sqlite3 store/messages.db`
- Set verbose logging: `LOG_LEVEL=debug npm run dev`
- Run `/debug` skill for guided troubleshooting

## Environment Variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `ASSISTANT_NAME` | `Andy` | Trigger word and message prefix |
| `ASSISTANT_HAS_OWN_NUMBER` | `false` | Skip prefix when bot has dedicated phone |
| `CONTAINER_IMAGE` | `nanoclaw-agent:latest` | Container image name |
| `CONTAINER_TIMEOUT` | `1800000` (30min) | Hard timeout per container |
| `IDLE_TIMEOUT` | `1800000` (30min) | Close container after this idle period |
| `MAX_CONCURRENT_CONTAINERS` | `5` | Max simultaneous containers |
| `LOG_LEVEL` | `info` | Pino log level |
| `TZ` | system | Timezone for scheduled tasks |
