# OpenClaw Developer Guide

> A comprehensive guide for developers who want to understand and contribute to OpenClaw

## Table of Contents

1. [What is OpenClaw?](#what-is-openclaw)
2. [Architecture Overview](#architecture-overview)
3. [Project Structure](#project-structure)
4. [Core Components](#core-components)
5. [Development Setup](#development-setup)
6. [Development Workflow](#development-workflow)
7. [Testing](#testing)
8. [Plugin System](#plugin-system)
9. [Key Architectural Patterns](#key-architectural-patterns)
10. [Contributing Guidelines](#contributing-guidelines)
11. [Resources](#resources)

---

## What is OpenClaw?

**OpenClaw** is a **personal AI assistant** that runs on your own devices and answers you through the messaging channels you already use.

### Key Characteristics

- **Multi-channel AI Gateway**: Connects to 20+ messaging platforms (WhatsApp, Telegram, Slack, Discord, Signal, iMessage, Matrix, etc.)
- **Local-first**: Runs on your devices with your rules
- **Privacy-focused**: You control where your data goes
- **Extensible**: Plugin-based architecture for channels, AI providers, and tools
- **Cross-platform**: Gateway (Node.js), macOS/iOS/Android apps
- **Real capabilities**: Browser automation, canvas rendering, voice wake, file operations, cron jobs

### The Product Philosophy

From `VISION.md`:
> OpenClaw is the AI that actually does things. It runs on your devices, in your channels, with your rules.

OpenClaw started as a personal playground and evolved through several iterations (Warelay → Clawdbot → Moltbot → OpenClaw). The focus is on building a genuinely useful assistant with:
- Security and safe defaults
- Wide platform support
- Privacy respect
- Real computer-use capabilities

---

## Architecture Overview

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        Gateway (Node.js)                     │
│  ┌──────────────┐  ┌──────────────┐  ┌─────────────────┐  │
│  │   Channels   │  │   Agents     │  │    Providers    │  │
│  │  (Plugins)   │  │   (Core)     │  │    (Plugins)    │  │
│  └──────────────┘  └──────────────┘  └─────────────────┘  │
│         ↕                  ↕                    ↕           │
│  ┌──────────────────────────────────────────────────────┐  │
│  │          WebSocket Gateway Protocol (RPC)            │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                            ↕
        ┌───────────────────┴───────────────────┐
        ↓                   ↓                    ↓
   ┌─────────┐         ┌─────────┐        ┌──────────┐
   │   CLI   │         │ WebChat │        │  Nodes   │
   │         │         │   UI    │        │ (iOS/Mac/│
   │         │         │         │        │ Android) │
   └─────────┘         └─────────┘        └──────────┘
```

### Core Architectural Principles

1. **Gateway-Centric**: Single long-lived Gateway daemon owns all messaging surfaces
2. **WebSocket Control Plane**: All clients (CLI, apps, nodes) connect via WebSocket
3. **Plugin-Based**: Channels and providers are plugins; core stays lean
4. **Strict Boundaries**: Extensions use `plugin-sdk`, core stays extension-agnostic
5. **Security-First**: Default to safe; expose clear knobs for trusted workflows

### Key Flows

#### Message Flow (Inbound)
```
Channel (WhatsApp/Telegram)
  → Channel Plugin
  → Routing Engine
  → Agent
  → AI Provider (Anthropic/OpenAI/etc)
  → Tool Execution
  → Response
  → Channel Plugin
  → User
```

#### Command Flow (Operator)
```
CLI/App
  → WebSocket Gateway
  → RPC Handler
  → Action (send/agent/status)
  → Response
  → CLI/App
```

---

## Project Structure

### Root Layout

```
openclaw/
├── src/                      # Core TypeScript codebase
│   ├── agents/              # Agent runtime, tools, inference loop
│   ├── channels/            # Channel core abstractions
│   ├── cli/                 # CLI commands
│   ├── config/              # Configuration system
│   ├── gateway/             # WebSocket gateway server & protocol
│   ├── plugin-sdk/          # Public plugin SDK (third-party contract)
│   ├── plugins/             # Plugin loader, registry, contracts
│   ├── infra/               # Infrastructure (auth, network, storage)
│   ├── sessions/            # Session management
│   └── [many more...]       # See below for details
│
├── extensions/              # Bundled plugins (100+ plugins)
│   ├── anthropic/          # Anthropic provider
│   ├── openai/             # OpenAI provider
│   ├── telegram/           # Telegram channel
│   ├── whatsapp/           # WhatsApp channel
│   ├── discord/            # Discord channel
│   └── [100+ more]         # Providers, channels, tools
│
├── apps/                    # Platform-specific apps
│   ├── macos/              # macOS menu bar app (Swift/SwiftUI)
│   ├── ios/                # iOS app (Swift/SwiftUI)
│   └── android/            # Android app (Kotlin)
│
├── docs/                    # Documentation
│   ├── channels/           # Channel-specific docs
│   ├── concepts/           # Architecture, sessions, models
│   ├── cli/                # CLI command reference
│   ├── gateway/            # Gateway, protocol, security
│   └── plugins/            # Plugin development guides
│
├── ui/                      # Web UI components
├── packages/                # Shared packages
├── test/                    # Test infrastructure & helpers
├── scripts/                 # Build, release, dev scripts
├── skills/                  # Bundled skills (slash commands)
├── AGENTS.md               # Agent/LLM instructions for contributors
└── CLAUDE.md → AGENTS.md   # Symlink
```

### Core `src/` Directories (Detailed)

| Directory | Purpose |
|-----------|---------|
| `agents/` | Agent runtime, tools system, inference loop, prompt management |
| `acp/` | ACP (Agent Context Protocol) implementation |
| `auto-reply/` | Auto-reply rules and matching |
| `bootstrap/` | Application initialization |
| `canvas-host/` | Canvas rendering host (HTML/CSS/JS workspace) |
| `channels/` | Channel core abstractions (not implementations) |
| `cli/` | CLI command implementations |
| `commands/` | Chat commands (`/status`, `/new`, `/reset`, etc.) |
| `config/` | Configuration schema, validation, loading |
| `context-engine/` | Context management for agent prompts |
| `cron/` | Cron job scheduling |
| `daemon/` | Daemon management (launchd/systemd) |
| `flows/` | Workflow automation (ClawFlow) |
| `gateway/` | WebSocket gateway server, protocol, RPC handlers |
| `hooks/` | Hook system (bundled hooks included) |
| `infra/` | Infrastructure (auth, network, storage, outbound) |
| `media/` | Media handling (images, video, audio) |
| `media-understanding/` | Media analysis capabilities |
| `mcp/` | MCP (Model Context Protocol) integration |
| `memory-host-sdk/` | Memory plugin SDK |
| `node-host/` | Node (device) management |
| `pairing/` | Device pairing system |
| `plugin-sdk/` | **Public plugin SDK** (the contract for all plugins) |
| `plugins/` | Plugin loader, registry, lifecycle |
| `routing/` | Channel/agent routing |
| `secrets/` | Secrets management |
| `security/` | Security policies, sandboxing |
| `sessions/` | Session management (conversation state) |
| `tasks/` | Task management |
| `tts/` | Text-to-speech |
| `tui/` | Terminal UI components |
| `wizard/` | Onboarding wizard |

---

## Core Components

### 1. Gateway

**Location**: `src/gateway/`

The Gateway is the heart of OpenClaw:
- Single long-lived daemon process
- Manages all messaging channel connections
- Exposes WebSocket RPC API for clients
- Handles authentication, pairing, and authorization
- Routes messages to appropriate agents
- Manages provider connections

**Key Files**:
- `src/gateway/server.ts` - WebSocket server
- `src/gateway/protocol/` - Protocol definitions (TypeBox schemas)
- `src/gateway/server-methods/` - RPC method handlers

### 2. Agent Runtime

**Location**: `src/agents/`

The agent system handles AI interactions:
- Inference loop (prompt → model → tools → response)
- Tool execution framework
- Context management
- Streaming responses
- Multi-agent routing

**Key Concepts**:
- **Agent**: Isolated workspace with its own config, sessions, tools
- **Session**: Conversation thread with message history
- **Tool**: Executable capability (bash, browser, canvas, etc.)

### 3. Plugin System

**Location**: `src/plugins/` (core), `src/plugin-sdk/` (public API), `extensions/` (implementations)

OpenClaw uses a sophisticated plugin architecture:

- **Plugin SDK** (`src/plugin-sdk/`): Public contract for plugins
- **Bundled Plugins** (`extensions/`): 100+ built-in plugins
- **Plugin Types**:
  - **Channel Plugins**: Messaging platforms (Telegram, Discord, WhatsApp, etc.)
  - **Provider Plugins**: AI model providers (Anthropic, OpenAI, Gemini, etc.)
  - **Tool Plugins**: Additional capabilities

**Boundary Rules** (Critical):
- Plugins import ONLY from `openclaw/plugin-sdk/*`
- Core must stay extension-agnostic
- No deep imports across boundaries
- Extensions export through `api.ts` / `runtime-api.ts`

### 4. Channel System

**Location**: `src/channels/` (core), `extensions/*/` (implementations)

Channels are the adapters between messaging platforms and the agent:
- Each platform is a plugin (Telegram, WhatsApp, Discord, etc.)
- Unified message format
- Presence, typing indicators, reactions
- Group support, threads, attachments
- DM policies and pairing

**Supported Channels** (25+):
WhatsApp, Telegram, Slack, Discord, Google Chat, Signal, iMessage, BlueBubbles, IRC, Microsoft Teams, Matrix, Feishu, LINE, Mattermost, Nextcloud Talk, Nostr, Synology Chat, Tlon, Twitch, Zalo, WeChat, QQ, WebChat

### 5. Configuration System

**Location**: `src/config/`

Comprehensive configuration management:
- Schema-based validation (TypeBox/Zod)
- User config: `~/.openclaw/config.json`
- Workspace config support
- Environment variable overrides
- Migration system
- Doctor/repair tools

**Key Areas**:
- Gateway settings (port, auth, TLS)
- Channel configs (credentials, policies)
- Agent configs (models, tools, prompts)
- Security policies (sandboxing, DM policies)

### 6. Security & Sandboxing

**Location**: `src/security/`, `src/gateway/auth.ts`

Security is a core focus:
- **DM Pairing**: Require approval for unknown senders
- **Sandboxing**: Docker/SSH/OpenShell sandboxes for untrusted sessions
- **Tool Policies**: Fine-grained tool allow/deny lists
- **Gateway Auth**: Token/password auth, Tailscale support
- **Device Pairing**: Challenge-based device approval

---

## Development Setup

### Prerequisites

- **Node.js**: 24 (recommended) or 22.16+
- **pnpm**: Package manager (required)
- **Platform-specific**:
  - macOS: Xcode for iOS/macOS apps
  - Android: Android Studio for Android app
  - Linux: systemd for daemon

### Initial Setup

```bash
# Clone the repository
git clone https://github.com/openclaw/openclaw.git
cd openclaw

# Install dependencies
pnpm install

# Build the project
pnpm build

# Run in dev mode
pnpm dev

# Or use the built CLI
pnpm openclaw --help
```

### Key Commands

| Command | Description |
|---------|-------------|
| `pnpm install` | Install dependencies |
| `pnpm build` | Build all packages |
| `pnpm dev` | Run CLI in dev mode |
| `pnpm openclaw <cmd>` | Run CLI commands |
| `pnpm check` | Full type/lint check (no tests) |
| `pnpm check:changed` | Smart gate: only check changed code |
| `pnpm test` | Run all tests |
| `pnpm test:changed` | Run tests for changed code |
| `pnpm lint` | Lint code |
| `pnpm format` | Format code |
| `pnpm tsgo` | Fast core typecheck |
| `pnpm tsgo:prod` | Core + extensions typecheck |

### Development Modes

OpenClaw has a sophisticated build system optimized for fast iteration:

- **Dev mode**: `pnpm dev` (hot reload)
- **Production build**: `pnpm build` → `openclaw`
- **Smart testing**: `pnpm test:changed` (only test what changed)
- **Smart checks**: `pnpm check:changed` (pre-commit hook)

---

## Development Workflow

### Pre-Commit Hook

OpenClaw uses aggressive quality gates:
- Format check
- Lint
- `pnpm check:changed --staged` (smart typecheck + tests)
- Skipped for docs-only changes

**Override**: `FAST_COMMIT=1` to skip changed-scope checks

### Making Changes

1. **Pick an area**:
   - Core: `src/`
   - Plugin: `extensions/<name>/`
   - Docs: `docs/`
   - Apps: `apps/`

2. **Read the boundary rules**:
   - Root: `AGENTS.md` (symlink: `CLAUDE.md`)
   - Extensions: `extensions/AGENTS.md`
   - Plugin SDK: `src/plugin-sdk/AGENTS.md`
   - Gateway: `src/gateway/AGENTS.md`
   - Scoped guides in subdirectories

3. **Make your changes**:
   - Follow strict types (avoid `any`)
   - No `@ts-nocheck`, no lint suppressions without explanation
   - Prefer discriminated unions over freeform strings
   - Keep files ~700 LOC max

4. **Test**:
   ```bash
   pnpm test:changed          # Test changed code
   pnpm check:changed         # Full gate
   ```

5. **Commit**:
   ```bash
   scripts/committer "feat: add X" path/to/file.ts
   ```
   Or use normal git with pre-commit hook enforcement

### Contribution Flow

From `VISION.md`:
- **One PR = One Issue/Topic** (no bundling unrelated changes)
- **PRs over ~5,000 lines** reviewed only in exceptional circumstances
- **No large batches** of tiny PRs at once

---

## Testing

### Testing Philosophy

From `AGENTS.md`:
> Keep tests at seam depth: unit-test pure helpers/contracts; one integration smoke per boundary, not per branch.

### Test Organization

- **Colocated tests**: `*.test.ts` next to implementation
- **E2E tests**: `*.e2e.test.ts`
- **Test helpers**: `test/helpers/`, `src/test-helpers/`
- **Fixtures**: `test-fixtures/`

### Running Tests

```bash
# All tests
pnpm test

# Changed tests only
pnpm test:changed

# Specific file/pattern
pnpm test <path-or-filter>

# Extension tests
pnpm test:extensions              # All extensions
pnpm test extensions/<id>         # Specific extension

# Coverage
pnpm test:coverage

# Live tests (requires real APIs)
OPENCLAW_LIVE_TEST=1 pnpm test:live
```

### Test Best Practices

- Clean up: timers, env, globals, mocks, sockets, temp dirs
- Prefer `beforeAll` imports over per-test `resetModules()`
- Mock expensive seams: scanners, network, provider SDKs
- Share fixtures/builders
- Delete duplicate assertions
- Use `*.runtime.ts` seams for lazy boundaries

### Performance

- Keep imports light (measure with `pnpm test:perf:imports <file>`)
- Avoid broad `importOriginal()` in hot tests
- Use injected deps over module mocks
- Max 16 workers; for memory pressure: `OPENCLAW_VITEST_MAX_WORKERS=1`

---

## Plugin System

### Plugin Architecture

OpenClaw's plugin system is the foundation for extensibility.

#### Plugin Types

1. **Channel Plugins**: Messaging platform adapters
   - Examples: `extensions/telegram/`, `extensions/discord/`
   - Interface: `ChannelPlugin` from `plugin-sdk`

2. **Provider Plugins**: AI model providers
   - Examples: `extensions/anthropic/`, `extensions/openai/`
   - Interface: `ProviderPlugin` from `plugin-sdk`

3. **Tool Plugins**: Additional capabilities
   - Examples: `extensions/browser/`, `extensions/webhooks/`

#### Plugin Structure

Each plugin is a self-contained package:

```
extensions/<plugin-name>/
├── openclaw.plugin.json      # Plugin manifest
├── package.json               # Package metadata
├── src/
│   ├── index.ts              # Plugin entry point
│   ├── [implementation].ts   # Core logic
│   └── ...
├── api.ts                     # Public API (optional)
├── runtime-api.ts            # Runtime exports (optional)
└── [tests, docs, etc.]
```

#### Plugin Manifest (`openclaw.plugin.json`)

Declares plugin metadata:
- ID, name, version
- Type (channel, provider, tool)
- Capabilities
- Dependencies
- Settings schema

#### Boundary Rules (Critical!)

From `extensions/AGENTS.md`:
- Extension production code imports ONLY from:
  - `openclaw/plugin-sdk/*`
  - Local barrels: `./api.ts`, `./runtime-api.ts`
- **NO** imports from:
  - Core internals: `src/**`, `src/channels/**`
  - Other extensions: `extensions/*/src/**`
  - Relative paths outside package root
- Export public APIs through `api.ts`
- Keep provider-specific logic in the plugin

#### Plugin SDK Exports

The Plugin SDK (`src/plugin-sdk/`) provides typed interfaces:

```typescript
// Core plugin interfaces
import { Plugin } from 'openclaw/plugin-sdk/core'
import { ChannelPlugin } from 'openclaw/plugin-sdk/core'
import { ProviderPlugin } from 'openclaw/plugin-sdk/provider-entry'

// Runtime utilities
import { runtime } from 'openclaw/plugin-sdk/runtime'
import { logger } from 'openclaw/plugin-sdk/runtime-logger'

// Channel helpers
import { channelHelpers } from 'openclaw/plugin-sdk/channel-contract'

// Provider helpers
import { providerHelpers } from 'openclaw/plugin-sdk/provider-setup'
```

### Creating a Plugin

#### External Plugin (Recommended)

For third-party plugins, publish as npm package:

```bash
npm init openclaw-plugin
# Follow prompts
npm publish
```

Users install with:
```bash
openclaw plugins install <your-plugin>
```

#### Bundled Plugin (High Bar)

From `VISION.md`:
> The bar for adding optional plugins to core is intentionally high.

Only consider bundled plugins for:
- Essential channels/providers
- Security-critical functionality
- Deep integration requirements

Add to `extensions/<name>/` following boundary rules.

---

## Key Architectural Patterns

### 1. Manifest-First Control Plane

Prefer declarative metadata over runtime execution:
- Discovery from manifests
- Config validation from schemas
- Setup hints from descriptors
- Avoid runtime code on control-plane paths

### 2. Lazy Loading & Hot Paths

Optimize startup performance:
- Use `*.runtime.ts` for lazy boundaries
- Keep SDK facades cheap at module load
- Avoid broad eager imports
- After edits, run `pnpm build` and check for `[INEFFECTIVE_DYNAMIC_IMPORT]`

### 3. Dependency Injection

Prefer injected dependencies over module-level state:
- Pass contexts, loggers, stores explicitly
- Enables testing without module mocks
- Keeps seams explicit

### 4. Type Safety

TypeScript strict mode throughout:
- Avoid `any` (prefer `unknown` + narrow)
- Use discriminated unions for variants
- Schema validation at boundaries (TypeBox/Zod)
- No `@ts-nocheck`, no suppression without explanation

### 5. Prompt-Cache Determinism

Keep outputs deterministic for AI context caching:
- Deterministic ordering for maps/sets/registries
- Stable plugin lists
- Consistent file/network result ordering
- Preserve old transcript bytes when possible

---

## Contributing Guidelines

### From `CONTRIBUTING.md`

1. **Read `AGENTS.md`** first (root + scoped guides)
2. **One PR = One Topic** (no bundling)
3. **Keep PRs under ~5,000 lines** (rare exceptions)
4. **No large batches** of tiny PRs
5. **Update docs** when behavior changes
6. **Add tests** for new functionality
7. **Pass all gates**: `pnpm check`, `pnpm test`

### Before Landing

- [ ] Run `pnpm check:changed` (or full `pnpm check`)
- [ ] Run tests: `pnpm test` (at minimum `pnpm test:changed`)
- [ ] If touching build/packaging: `pnpm build` must pass
- [ ] Update changelog (user-facing changes only)
- [ ] Rebase on latest `origin/main`

### Commit Messages

- Conventional-ish format
- Concise, action-oriented
- Group related changes
- Example: `feat(channels): add Matrix thread support`

### Using `scripts/committer`

```bash
scripts/committer "feat: add X" path/to/file.ts path/to/another.ts
```

This helper ensures only intended files are staged.

### Code Style

From `AGENTS.md`:
- TypeScript ESM
- Strict types
- No `any` (prefer `unknown`)
- External boundaries: `zod` or schema helpers
- Runtime branching: discriminated unions
- Brief comments (non-obvious logic only)
- Split files around ~700 LOC
- American spelling

---

## Resources

### Essential Documentation

- **Root**: `README.md`, `AGENTS.md` (CLAUDE.md), `VISION.md`, `CONTRIBUTING.md`, `SECURITY.md`
- **Architecture**: `docs/concepts/architecture.md`
- **Gateway Protocol**: `docs/reference/rpc.md`, `src/gateway/protocol/AGENTS.md`
- **Plugin Development**: `docs/plugins/`
- **Channel Guides**: `docs/channels/`
- **Security**: `docs/gateway/security.md`, `docs/gateway/sandboxing.md`

### Boundary Guides

Critical reading for respective areas:
- `AGENTS.md` - Root rules
- `extensions/AGENTS.md` - Plugin boundary rules
- `src/plugin-sdk/AGENTS.md` - SDK contract rules
- `src/gateway/AGENTS.md` - Gateway hot paths
- `src/channels/AGENTS.md` - Channel core rules
- `src/plugins/AGENTS.md` - Plugin loader rules
- `test/helpers/AGENTS.md` - Test helper rules

### External Links

- **Website**: https://openclaw.ai
- **Docs**: https://docs.openclaw.ai
- **GitHub**: https://github.com/openclaw/openclaw
- **Discord**: https://discord.gg/clawd
- **ClawHub**: https://clawhub.com (skill registry)

### Getting Help

- GitHub Issues: https://github.com/openclaw/openclaw/issues
- Discord Community: https://discord.gg/clawd
- CLI: `openclaw --help`, `openclaw doctor`
- Docs search: https://docs.openclaw.ai

---

## Quick Start Checklist for New Contributors

- [ ] Clone repo: `git clone https://github.com/openclaw/openclaw.git`
- [ ] Install: `pnpm install`
- [ ] Build: `pnpm build`
- [ ] Read: `AGENTS.md`, `VISION.md`, `CONTRIBUTING.md`
- [ ] Read boundary guide for your area (e.g., `extensions/AGENTS.md`)
- [ ] Run tests: `pnpm test`
- [ ] Run gates: `pnpm check:changed`
- [ ] Pick an issue or area to explore
- [ ] Make small changes, iterate, learn the flow
- [ ] Ask questions in Discord or GitHub Issues

---

## Understanding the Codebase by Example

### Example 1: Message Flow (Inbound)

User sends WhatsApp message "What's the weather?" → Here's what happens:

1. **Channel Plugin** (`extensions/whatsapp/`):
   - Baileys SDK receives WhatsApp message
   - Plugin converts to unified `InboundMessage`
   - Checks DM policy, pairing

2. **Routing** (`src/routing/`):
   - Determines which agent handles this channel/peer
   - Routes to `main` agent by default

3. **Agent** (`src/agents/`):
   - Loads session (conversation history)
   - Builds prompt with message + context
   - Calls inference loop

4. **Provider Plugin** (`extensions/anthropic/`):
   - Sends prompt to Anthropic API
   - Streams response tokens

5. **Tool Execution** (`src/agents/tools/`):
   - If model requests tool (e.g., `web_search`)
   - Execute tool, return results
   - Continue inference loop

6. **Response**:
   - Agent streams response back
   - Channel plugin sends to WhatsApp
   - User receives answer

### Example 2: Adding a New Channel Plugin

Want to add support for a new messaging platform?

1. **Create plugin structure**:
   ```
   extensions/my-channel/
   ├── openclaw.plugin.json
   ├── package.json
   ├── src/
   │   └── index.ts
   └── api.ts
   ```

2. **Implement `ChannelPlugin`**:
   ```typescript
   import { ChannelPlugin } from 'openclaw/plugin-sdk/core'

   export const myChannelPlugin: ChannelPlugin = {
     id: 'my-channel',
     name: 'My Channel',
     version: '1.0.0',

     async activate(context) {
       // Connect to messaging platform
       // Register handlers for incoming messages
       // Return cleanup function
     },

     async send(message) {
       // Send message to platform
     },

     // ... other methods
   }
   ```

3. **Add manifest** (`openclaw.plugin.json`):
   ```json
   {
     "id": "my-channel",
     "name": "My Channel",
     "version": "1.0.0",
     "type": "channel",
     "main": "dist/index.js"
   }
   ```

4. **Test**: `pnpm test extensions/my-channel`
5. **Document**: Add `docs/channels/my-channel.md`
6. **PR**: Follow contribution guidelines

### Example 3: Understanding Tool Execution

Tools give the agent capabilities. Example: `bash` tool

1. **Tool Declaration** (`src/agents/tools/bash/`):
   - Schema defines parameters (command, timeout, etc.)
   - Handler executes command
   - Returns result

2. **Agent Request**:
   - Model decides to use tool
   - Sends tool request in structured format

3. **Execution** (`src/agents/tools/execution.ts`):
   - Validates request against schema
   - Checks security policy (sandbox mode, allowed tools)
   - Executes tool handler
   - Returns result to model

4. **Continuation**:
   - Model receives tool result
   - Decides next action (more tools, final answer)

---

## Next Steps

Now that you understand the structure, dive deeper:

1. **Explore a component**: Pick one area (channels, agents, plugins) and read the code
2. **Run the system**: `pnpm openclaw onboard`, connect a channel, send a message
3. **Make a change**: Start small (fix a typo, add a test, improve docs)
4. **Read actual code**: The best learning is reading real implementations
5. **Ask questions**: Discord, GitHub Issues

**Welcome to OpenClaw!** 🦞

---

*This guide was created to help developers understand and contribute to OpenClaw. For the latest information, always check the official repository and documentation.*

**Last Updated**: 2026-04-21
**OpenClaw Version**: 2026.4.20
**Repository**: https://github.com/openclaw/openclaw
