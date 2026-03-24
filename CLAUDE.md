# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a Claude Code Skill that bridges personal WeChat to local Claude Code. It allows users to chat with Claude via WeChat, including text, image analysis, permission approvals, slash commands, and launching any installed Claude Code skill.

## Common Commands

### Development
- `npm run build` – Compile TypeScript to `dist/`
- `npm run dev` – Watch mode, auto-compile on changes
- `npm run setup` – First-time setup: generate QR code to bind WeChat account and configure working directory
- `npm run start` – Run the compiled main.js (starts the daemon)
- `npm run daemon -- <command>` – Manage the cross-platform daemon:
  - `npm run daemon -- start` – Start the service (macOS: launchd, Linux: systemd/nohup fallback, Windows: background process)
  - `npm run daemon -- stop` – Stop the service
  - `npm run daemon -- restart` – Restart after code updates
  - `npm run daemon -- status` – Check running status
  - `npm run daemon -- logs` – View recent logs

### Environment Variables
- `ANTHROPIC_API_KEY`, `ANTHROPIC_BASE_URL` – For Claude SDK (supports third‑party providers)
- `WCC_DATA_DIR` – Override data directory (default: `~/.wechat-claude-code/`)

## Architecture

The system is a Node.js daemon that long‑polls the WeChat ilink bot API and forwards messages to Claude Code via `@anthropic‑ai/claude‑agent‑sdk`.

### Core Components

1. **WeChat API (`src/wechat/api.ts`)** – HTTP client for ilink bot endpoints (getUpdates, sendMessage, getUploadUrl)
2. **Monitor (`src/wechat/monitor.ts`)** – Polls for new messages, deduplicates, handles session expiry and backoff
3. **Message Handler (`src/main.ts`)** – Processes incoming WeChat messages:
   - Routes slash commands via `src/commands/router.ts`
   - Manages session state (`idle`, `processing`, `waiting_permission`)
   - Downloads images and forwards text/images to Claude
4. **Permission Broker (`src/permission.ts`)** – Manages pending tool‑use approvals with 120‑second timeout
5. **Session Store (`src/session.ts`)** – Persists session data (SDK session ID, working directory, model, permission mode, chat history) per account
6. **Claude Provider (`src/claude/provider.ts`)** – Wraps the SDK’s `query()` function, adapts images, bridges permission callbacks
7. **Command Handlers (`src/commands/handlers.ts`)** – Implement `/help`, `/clear`, `/model`, `/permission`, `/status`, `/skills`, etc.
8. **Skill Scanner (`src/claude/skill‑scanner.ts`)** – Discovers installed Claude Code skills for the `/skills` command

### Data Flow

```
WeChat message → Monitor.poll() → handleMessage() → command router → (if slash command) → handler
                                                → (if normal text/image) → sendToClaude()
                                                                         → Claude SDK query()
                                                                         → permission broker (if needed)
                                                                         → response split & sent back via WeChat API
```

### State Management

- Each WeChat account has a separate session file (`~/.wechat‑claude‑code/sessions/{accountId}.json`)
- Session includes: `sdkSessionId` (for resuming), `workingDirectory`, `model`, `permissionMode`, `state`, `chatHistory`
- State machine: `idle` ↔ `processing` ↔ `waiting_permission`

### Platform‑Specific Daemon Management

- **macOS**: launchd agent (auto‑start, auto‑restart) – managed by `scripts/daemon.sh`
- **Linux**: systemd user service (falls back to `nohup` with PID file)
- **Windows**: background process with PID file (requires Git Bash or compatible shell)
- The daemon script automatically detects the platform and sets up the appropriate service.

## Data Storage

All persistent data lives in `~/.wechat‑claude‑code/` (configurable via `WCC_DATA_DIR`):

```
~/.wechat‑claude‑code/
├── accounts/          # WeChat account credentials (JSON per account)
├── config.env         # Global config: workingDirectory, model, permissionMode
├── sessions/          # Session data (JSON per account)
├── get_updates_buf    # Message polling sync buffer
└── logs/              # Rotating logs (stdout.log, stderr.log, bridge‑*.log)
```

## Development Notes

- Written in TypeScript (ES2022, Node16 module resolution)
- No unit tests currently; manual testing via WeChat
- The `postinstall` script runs `npm run build` automatically
- The project is designed to be installed as a Claude Code Skill in `~/.claude/skills/wechat‑claude‑code/`
- When adding new slash commands, update `src/commands/router.ts` and `src/commands/handlers.ts`
- Permission modes: `default` (manual), `acceptEdits` (auto‑approve edits), `plan` (read‑only), `auto` (auto‑approve all tools)
- Images are downloaded, converted to base64 data URIs, and passed to the Claude SDK as image content blocks