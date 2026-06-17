## Summary
PingBot is a tiny Telegram bot that supports three commands: /start (warm greeting), /ping (replies "pong" and increments a global counter), and /count (shows the total number of pings served across all users). The bot is intentionally minimal: no other features, no external APIs, single-process long-polling deployment, persistent global counter.

## Audience
Anyone who installs and runs the bot (public or private) — end users interact via Telegram commands.

## Core entities
- Global counter: single integer tracking total number of times /ping was invoked (across all chats/users).
- Telegram users/chats: used only to address replies; no per-user data is stored.

## Integrations & notification targets
- None. The bot does not call external APIs or send notifications outside Telegram.
- Telegram Bot API only (via a bot token provided by the operator).

## Interaction flows
- /start
  - Trigger: user sends /start to the bot (private or group where bot is present).
  - Response: a warm greeting. Exact text: "Hello! 👋 I'm PingBot — send /ping and I'll reply 'pong'."
  - No state changes.

- /ping
  - Trigger: user sends /ping.
  - Behavior: atomically increment the global counter by 1, persist the new value, then reply exactly: "pong".
  - Response: single message with the text `pong`.

- /count
  - Trigger: user sends /count.
  - Behavior: read the current global counter and return it.
  - Response: "Pings served: N" where N is the integer value of the global counter.

Notes on command handling
- The bot will respond to commands in private chats and groups (standard command handling). In groups the bot will respond to /ping and /count only when the command targets the bot or the group allows bots to receive commands (standard Telegram behavior). The bot will not implement any extra mention parsing, inline queries, or callback buttons.

## Persistence
- Storage: SQLite database file (data/pingbot.sqlite).
- Schema (single table):
  - Table `counter` with columns: `id INTEGER PRIMARY KEY CHECK (id=1)`, `value INTEGER NOT NULL`.
  - Implementation detail: keep a single row with id=1; initialize value=0 when DB is first created.
- Access pattern: on /ping run an atomic SQL update (UPDATE counter SET value = value + 1 WHERE id = 1; then SELECT value FROM counter WHERE id = 1;) inside a transaction to avoid races. Use a small async DB client (aiosqlite) for concurrency-safe access within the single-process assumption.

## Runtime & deployment
- Language & libs: Python 3.11+, aiogram for Telegram, aiosqlite for DB access. Requirements: aiogram, aiosqlite.
- Bot token: provided via environment variable TELEGRAM_BOT_TOKEN.
- Deployment mode: long polling (getUpdates) — simplest for small/minimal bot and requires no HTTPS/webhook setup.
- Process: run as a single process (systemd/service, Docker, or run manually). Concurrency within the process is handled by asyncio; SQLite chosen for minimal ops and local persistence.

## Payments
- None.

## Non-goals
- No per-user or per-chat counters — only a single global counter.
- No admin commands, no authentication, no metrics dashboards, no external integrations, and no extra UI elements (buttons or inline queries).
- No webhooks, unless operator explicitly requests switching to webhooks later.

## Assumptions & defaults
- Programming language: Python 3.11 with aiogram + aiosqlite — minimal, well-known async stack suitable for Telegram bots.
  Rationale: keeps implementation small and widely maintainable.
- Persistence: SQLite file at data/pingbot.sqlite, single-row `counter` table initialized to 0.
  Rationale: zero-ops local persistence suitable for a tiny bot and survives restarts.
- Command responses: /ping replies exactly `pong`; /count replies `Pings served: N`; /start uses a short warm greeting as specified above.
  Rationale: keep messages deterministic and minimal.
- Deployment: long-polling by default (no webhook) and single-process runtime.
  Rationale: simplest run mode requiring no HTTPS endpoint or external infra.
- Bot token provisioning: TELEGRAM_BOT_TOKEN environment variable.
  Rationale: standard secure practice and easy to inject in most runtimes.

This brief contains all decisions required to implement and deploy PingBot. Follow it exactly to keep the scope minimal.