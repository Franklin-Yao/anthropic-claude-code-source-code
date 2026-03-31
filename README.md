# Claude Code — Source Code

The code is from https://x.com/Fried_rice/status/2038894956459290963. 
# History
At 7am on March 31, 2026, I woke up and opened my phone. I saw this news that claude code leaked their code. This is so exciting as every developer want to know how it works because it is a masterpiece of agent harness engineering.

Let's Study it and create a better version out of it. Please join our discord: https://discord.gg/BwcjABYwEe

# Codebase Explanation

Claude Code is Anthropic's official CLI for Claude. It is an interactive AI coding assistant that runs in your terminal, capable of reading and editing files, running shell commands, searching the web, managing tasks, spawning sub-agents, and integrating with external tools through the Model Context Protocol (MCP).

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Technology Stack](#technology-stack)
- [Directory Structure](#directory-structure)
- [Entry Points & Startup Sequence](#entry-points--startup-sequence)
- [Core Systems](#core-systems)
  - [Query Engine](#query-engine)
  - [Tool System](#tool-system)
  - [Agent Orchestration](#agent-orchestration)
  - [Context System](#context-system)
  - [Permission System](#permission-system)
  - [Terminal UI (Ink)](#terminal-ui-ink)
  - [State Management](#state-management)
  - [MCP Integration](#mcp-integration)
  - [Skills & Plugins](#skills--plugins)
  - [Hooks](#hooks)
- [CLI Commands](#cli-commands)
- [Tools Reference](#tools-reference)
- [Configuration](#configuration)
- [Authentication & Providers](#authentication--providers)
- [Environment Variables](#environment-variables)
- [Feature Flags (Build-time)](#feature-flags-build-time)
- [Session & History Management](#session--history-management)
- [Remote & Bridge Modes](#remote--bridge-modes)

---

## Overview

Claude Code provides a full agentic loop in the terminal:

1. User enters a prompt or command at the REPL.
2. Context is gathered (git status, memory files, file attachments, project info).
3. A query is built and sent to the Claude API via `QueryEngine`.
4. Claude responds with text and/or tool calls.
5. Tools execute (with permission checks) and their results feed back into the loop.
6. Output is rendered in the terminal via a custom React/Ink renderer.
7. The loop repeats until Claude produces a final response.

The application supports single-turn queries, interactive multi-turn sessions, multi-agent orchestration, background task execution, remote sessions, plan/approval mode, and extensibility via MCP servers, plugins, and skills.

---

## Architecture

```
┌──────────────────────────────────────────────────────────┐
│  CLI Entry Points (entrypoints/cli.tsx, entrypoints/init)│
├──────────────────────────────────────────────────────────┤
│  REPL / replLauncher.tsx  ←→  commands.ts                │
├──────────────────────────────────────────────────────────┤
│  Query Layer                                             │
│    query.ts          – top-level query function          │
│    QueryEngine.ts    – message loop, token budgeting     │
│    context.ts        – context assembly                  │
├──────────────────────────────────────────────────────────┤
│  Tool Execution                                          │
│    Tool.ts           – base tool interface & runner      │
│    tools.ts          – registry & permission filtering   │
│    tools/*           – 45+ individual tool modules       │
├──────────────────────────────────────────────────────────┤
│  Services                                                │
│    services/api/     – Anthropic API client              │
│    services/mcp/     – MCP server management             │
│    services/oauth/   – OAuth flows                       │
│    services/compact/ – Context compaction                │
│    services/lsp/     – Language Server Protocol          │
│    services/analytics/ – Telemetry & metrics             │
│    + 32 more service modules                             │
├──────────────────────────────────────────────────────────┤
│  State                                                   │
│    bootstrap/        – global session state              │
│    state/            – React AppState & AppStateStore    │
├──────────────────────────────────────────────────────────┤
│  UI Layer                                                │
│    ink/              – custom React → terminal renderer  │
│    components/       – 146 React terminal components     │
└──────────────────────────────────────────────────────────┘
```

---

## Technology Stack

| Layer | Technology |
|-------|-----------|
| Runtime | Bun.js |
| Language | TypeScript / TSX |
| UI Framework | React (custom terminal renderer via Ink) |
| CLI Parsing | Commander.js (`@commander-js/extra-typings`) |
| Schema Validation | Zod v4 |
| LLM SDK | `@anthropic-ai/sdk` |
| Tool Protocol | `@modelcontextprotocol/sdk` |
| Observability | OpenTelemetry |
| Terminal Colors | Chalk |
| Utilities | Lodash-es |
| Cloud Auth | `google-auth-library`, `@azure/identity` |

---

## Directory Structure

```
.
├── entrypoints/          # CLI bootstrap (cli.tsx, init.ts)
├── main.tsx              # Argument parsing, initialization, launchRepl()
├── setup.ts              # Node version check, git detection, worktree setup
├── replLauncher.tsx      # Interactive REPL loop
├── commands.ts           # Command routing definitions
├── commands/             # 103 command subdirectories
│
├── query.ts              # Top-level query function
├── QueryEngine.ts        # Message loop, token budgeting, compaction
├── context.ts            # Context gathering (git, memory, attachments)
├── query/                # Query config and transition helpers
│
├── Tool.ts               # Base tool interface and execution runner
├── tools.ts              # Tool registry, permission filtering
├── tools/                # 45+ individual tool implementations
│   ├── AgentTool/        # Multi-agent spawning & orchestration
│   ├── BashTool/         # Shell command execution
│   ├── FileReadTool/     # File reading
│   ├── FileWriteTool/    # File writing
│   ├── FileEditTool/     # File editing with diff display
│   ├── GlobTool/         # File pattern matching
│   ├── GrepTool/         # Content search
│   ├── WebFetchTool/     # HTTP fetching
│   ├── WebSearchTool/    # Web search
│   ├── MCPTool/          # MCP server tool proxy
│   ├── SkillTool/        # Skill execution
│   ├── TaskCreateTool/   # Task creation
│   ├── TaskListTool/     # Task listing
│   ├── TaskGetTool/      # Task retrieval
│   ├── TaskUpdateTool/   # Task updates
│   ├── TaskOutputTool/   # Task output streaming
│   ├── TaskStopTool/     # Task cancellation
│   ├── NotebookEditTool/ # Jupyter notebook editing
│   ├── PowerShellTool/   # PowerShell execution (Windows)
│   ├── EnterWorktreeTool/# Git worktree isolation
│   ├── ExitWorktreeTool/ # Worktree exit
│   ├── EnterPlanMode/    # Plan mode activation
│   ├── ExitPlanMode/     # Plan mode deactivation
│   └── ...               # Additional tools
│
├── services/             # 38+ service modules
│   ├── api/              # Anthropic API client (claude.ts, client.ts)
│   ├── mcp/              # MCP server lifecycle management
│   ├── oauth/            # OAuth authentication flows
│   ├── compact/          # Context compaction strategies
│   ├── lsp/              # Language Server Protocol integration
│   ├── analytics/        # Telemetry and growth tracking
│   ├── plugins/          # Plugin system
│   ├── SessionMemory/    # Session persistence
│   └── remoteManagedSettings/ # Remote configuration sync
│
├── state/                # React AppState & AppStateStore
├── bootstrap/            # Global session initialization
├── hooks/                # 87 custom React hooks
├── components/           # 146 React terminal UI components
├── ink/                  # Custom React → ANSI terminal renderer
│
├── constants/            # System prompts, OAuth config, product info
├── schemas/              # Zod schemas
├── types/                # TypeScript type definitions
├── utils/                # 549+ utility files
├── migrations/           # Settings migration helpers
├── keybindings/          # Keyboard binding configuration
│
├── coordinator/          # Multi-agent coordinator
├── bridge/               # Remote session bridge (replBridge, remoteBridgeCore)
├── remote/               # Remote session management
├── tasks/                # Local and remote agent task management
├── Task.ts               # Task model
├── tasks.ts              # Task utilities
│
├── memdir/               # claude.md memory file management
├── context/              # Project context gathering
├── history.ts            # Session history helpers
├── assistant/            # KAIROS assistant mode
├── buddy/                # Internal buddy assistant
├── skills/               # Built-in skill definitions
├── plugins/              # Plugin runtime
├── outputStyles/         # Output formatting (json, text, xml, etc.)
├── vim/                  # Vim-mode keybinding support
├── voice/                # Voice interaction (build-gated)
├── screens/              # Full-screen UI screens
├── server/               # Internal server utilities
└── upstreamproxy/        # Upstream proxy support
```

---

## Entry Points & Startup Sequence

### 1. `entrypoints/cli.tsx`
The outermost bootstrap. Handles:
- `--version` flag
- Special modes: `--daemon-worker`, `--dump-system-prompt`, `--claude-in-chrome-mcp`
- Startup performance profiling
- Routes to `main.tsx`

### 2. `entrypoints/init.ts`
Service initialization layer. Runs before the REPL:
- Enables configuration system
- Applies safe environment variables
- Registers graceful shutdown handlers
- Initializes telemetry, OAuth, and policy limits
- Loads remote managed settings
- Sets up policy refresh on auth change events

### 3. `main.tsx`
Core initialization (the largest file, ~4,700 lines):
- Parses all CLI arguments via Commander.js
- Parallelizes: startup profiler, MDM reading, keychain prefetch
- Handles trust dialog acceptance and first-run onboarding
- Loads user and project configuration
- Initializes APIs, services, and the hook system
- Calls `launchRepl()` to start the interactive session

### 4. `setup.ts`
Project-level setup:
- Requires Node.js 18+
- Detects git repository and project root
- Sets up working directories
- Handles worktree creation if enabled
- Manages terminal state restoration on exit

### 5. `replLauncher.tsx`
The interactive REPL. Main loop:
- Reads user input
- Processes special `/commands`
- Invokes `query()` → `QueryEngine` for AI turns
- Renders output via the Ink UI layer

---

## Core Systems

### Query Engine

**Files:** `query.ts`, `QueryEngine.ts`

`query.ts` is the top-level function that assembles:
- System prompt (from `constants/prompts.ts`)
- Context (git status, memory files, attachments, project info)
- Message history
- Available tools (filtered by permissions)

`QueryEngine.ts` runs the actual message loop:
- Sends requests to the Anthropic API
- Handles streaming responses
- Dispatches tool calls to the tool runner
- Manages token budgets and context window limits
- Triggers compaction strategies when context grows too large:
  - **Microcompact** — lightweight summarization
  - **Reactive compaction** — summarize on overflow
  - **Snipping** — drop old messages
- Handles stop reasons: `end_turn`, `tool_use`, `max_tokens`
- Stores large tool results with disk spilling to avoid memory bloat

### Tool System

**Files:** `Tool.ts`, `tools.ts`, `tools/`

`Tool.ts` defines the base tool interface. Every tool exposes:
- `name` — tool name used in API calls
- `description` — shown to Claude
- `inputSchema` — Zod schema for input validation
- `call(input, context)` — execution function

`tools.ts` builds the active tool registry at runtime by:
- Loading all tool modules
- Applying permission allow/deny rules from settings
- Filtering by current mode (plan mode, sandbox, remote)
- Injecting MCP server tools

Tools execute with the following guarantees:
- Input validated against schema before execution
- Permissions checked against user settings and bypass mode
- Results formatted for both terminal display and API feedback

### Agent Orchestration

**Files:** `tools/AgentTool/`, `coordinator/`, `tasks/`, `Task.ts`

The `Agent` tool allows Claude to spawn sub-agents that independently handle subtasks:

```
Agent({
  description: "short task description",
  prompt: "detailed instructions",
  subagent_type?: "general-purpose" | "Explore" | ...,
  model?: "haiku" | "sonnet" | "opus",
  run_in_background?: boolean,
  isolation?: "worktree"
})
```

Agent capabilities:
- Parallel execution via background mode
- Worktree isolation for safe parallel file edits (each agent gets its own git worktree)
- Remote execution in CCR (Claude Code Remote) environments
- Progress tracking via the Task system
- Token budget propagation from parent to child agents
- Visual color assignment per agent for terminal output distinction

The coordinator (`coordinator/`) manages agent swarms with team names and lifecycle events.

### Context System

**Files:** `context.ts`, `context/`, `memdir/`

Before each query, context is assembled from:
- **Git info** — current branch, staged/unstaged files, recent commits (cached)
- **Memory files** — `claude.md` files found in project directories and `~/.claude/`
- **File attachments** — images, archives, or arbitrary files passed by the user
- **Project metadata** — repo name, directory structure, toolchain detection
- **External includes** — `@path/to/file` references in messages

Context assembly is token-aware and deduplicates attachments across turns.

### Permission System

**Files:** `tools.ts`, `Tool.ts`, hooks in `hooks/`

Permission evaluation:
- **Deny rules** — block specific tools, commands, or file paths
- **Allow rules** — explicitly permit specific operations
- **Plan mode** — Claude must present a plan for user approval before executing tools
- **Bypass modes:**
  - `dangerous` — skip confirmation for destructive tools
  - `all` — auto-approve all tool calls
  - `explicit` — only approve listed tools
- **Sandbox mode** — restricts file system and network access
- **MCP server permissions** — separate gates per MCP server

### Terminal UI (Ink)

**Files:** `ink/`, `components/`

A custom React-to-terminal renderer built on top of Ink concepts:
- Parses React component trees into ANSI escape sequences
- Manages layout (flexbox-like), borders, colors, and text wrapping
- Handles live re-renders (streaming output updates without flickering)
- Supports 146 specialized components including:
  - Diff display for file edits
  - Tool call input/output panels
  - Progress indicators and spinners
  - Dialogs and prompts
  - Syntax-highlighted code blocks
  - Cost and token usage display

### State Management

**Files:** `state/`, `bootstrap/`

Two-tier state:
- **`bootstrap/`** — Global mutable state initialized at startup (config, auth, session ID, flags). Accessed synchronously throughout the app.
- **`state/`** — React-based `AppState` and `AppStateStore`. Drives re-renders of the Ink UI. Contains the conversation messages, tool states, current mode, and UI interaction state.

### MCP Integration

**Files:** `services/mcp/`, `tools/MCPTool/`, `tools/ListMcpResourcesTool/`

Model Context Protocol support:
- Starts and manages MCP server processes (stdio, SSE, and WebSocket transports)
- Proxies MCP server tools into Claude's tool list
- Exposes MCP resources as context attachments
- Applies per-server permission gates from settings

Configure MCP servers in `settings.json`:
```json
{
  "mcp": {
    "servers": {
      "my-server": {
        "command": "npx",
        "args": ["my-mcp-server"]
      }
    }
  }
}
```

### Skills & Plugins

**Files:** `skills/`, `tools/SkillTool/`, `plugins/`, `services/plugins/`

**Skills** are reusable prompt templates invoked with `/skill-name`. They expand into full instructions before being sent to Claude. Built-in skills live in `skills/`; user-defined skills can be placed in `~/.claude/skills/` or `.claude/skills/`.

**Plugins** extend Claude Code with custom commands and MCP servers. Plugin entries are loaded from `settings.json` and from local `.claude/plugins/` directories.

### Hooks

**Files:** `hooks/` (87 custom React hooks), settings `hooks` key

Two categories:

1. **React UI hooks** (`hooks/`) — custom hooks used inside Ink components (e.g., `usePermissions`, `useKeybindings`, `useSuggestions`, `useCostTracker`).

2. **Shell hooks** (configured in `settings.json`) — shell commands that run automatically in response to Claude Code events:
   - `PreToolUse` — run before a tool executes
   - `PostToolUse` — run after a tool completes
   - `UserPromptSubmit` — run when the user submits input
   - `Notification` — run on notifications

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Bash",
        "hooks": [{ "type": "command", "command": "echo 'bash ran'" }]
      }
    ]
  }
}
```

---

## CLI Commands

Commands are defined in `commands/` (103 subdirectories) and routed through `commands.ts`. Select commands:

| Category | Commands |
|----------|----------|
| **Session** | `session`, `resume`, `history`, `teleport` |
| **Auth** | `login`, `logout` |
| **Configuration** | `config`, `keybindings`, `color`, `theme` |
| **Code** | `commit`, `review`, `diff`, `pr_comments` |
| **Planning** | `plan`, `brief` |
| **Extensions** | `skills`, `plugin`, `mcp` |
| **Memory** | `memory` |
| **Diagnostics** | `doctor`, `status`, `help` |
| **Context** | `compact`, `context`, `effort` |
| **Export** | `export`, `share`, `copy` |
| **Onboarding** | `onboarding`, `release-notes`, `feedback` |

---

## Tools Reference

| Tool | Description |
|------|-------------|
| `Agent` | Spawn a sub-agent to handle a subtask |
| `Bash` | Execute shell commands |
| `PowerShell` | Execute PowerShell commands (Windows) |
| `Read` | Read file contents |
| `Write` | Write file contents |
| `Edit` | Edit files with exact string replacement |
| `Glob` | Find files by glob pattern |
| `Grep` | Search file contents with regex |
| `WebFetch` | Fetch a URL |
| `WebSearch` | Search the web |
| `NotebookEdit` | Edit Jupyter notebook cells |
| `TaskCreate` | Create a tracked task |
| `TaskList` | List active tasks |
| `TaskGet` | Get task details |
| `TaskUpdate` | Update task status |
| `TaskOutput` | Stream task output |
| `TaskStop` | Stop a running task |
| `EnterPlanMode` | Activate plan/approval mode |
| `ExitPlanMode` | Deactivate plan mode |
| `EnterWorktree` | Enter a git worktree isolation |
| `ExitWorktree` | Exit git worktree |
| `MCPTool` | Proxy tools from an MCP server |
| `ListMcpResources` | List available MCP resources |
| `SkillTool` | Execute a skill prompt |

---

## Configuration

Configuration is loaded from multiple sources in priority order (highest first):

1. CLI flags
2. Environment variables
3. `.claude/settings.json` (project-level)
4. `~/.claude/settings.json` (user-level)
5. Remote managed settings (enterprise)

### Key `settings.json` Fields

```json
{
  "model": "claude-sonnet-4-6",
  "permissions": {
    "allow": ["Bash(git *)"],
    "deny": ["Bash(rm -rf *)"]
  },
  "effort": "normal",
  "fastMode": false,
  "autoMode": false,
  "autoUpdates": true,
  "outputStyle": "text",
  "telemetry": true,
  "mcp": { "servers": {} },
  "plugins": [],
  "hooks": {},
  "sandbox": false
}
```

### Memory Files

`claude.md` files are automatically injected as context. Claude Code searches for them in:
- `~/.claude/CLAUDE.md` — global user instructions
- `<project-root>/CLAUDE.md` — project instructions
- Any `claude.md` file in the directory tree

---

## Authentication & Providers

Claude Code supports four API providers, selected automatically based on available credentials:

| Provider | Credential |
|----------|-----------|
| **Anthropic API** | `ANTHROPIC_API_KEY` or OAuth login |
| **AWS Bedrock** | AWS credentials + `AWS_REGION` |
| **Azure AI Foundry** | `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, etc. |
| **GCP Vertex AI** | Application Default Credentials |

OAuth login (`claude login`) performs a browser-based OAuth flow and stores tokens in the system keychain.

---

## Environment Variables

| Variable | Description |
|----------|-------------|
| `ANTHROPIC_API_KEY` | Direct API key |
| `ANTHROPIC_BASE_URL` | Override API endpoint |
| `CLAUDE_CODE_REMOTE` | Enable remote/CCR mode |
| `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS` | Disable background task execution |
| `DISABLE_INTERLEAVED_THINKING` | Disable extended thinking |
| `CLAUDE_CODE_MAX_OUTPUT_TOKENS` | Override max output tokens |
| `AWS_REGION` | AWS Bedrock region |
| `BASH_DEFAULT_TIMEOUT_MS` | Default timeout for Bash tool |
| `BASH_MAX_TIMEOUT_MS` | Maximum allowed Bash timeout |

---

## Feature Flags (Build-time)

Feature flags are processed by Bun's bundler for dead code elimination. Active flags:

| Flag | Feature |
|------|---------|
| `COORDINATOR_MODE` | Multi-agent coordinator |
| `KAIROS` | KAIROS assistant mode |
| `PROACTIVE` | Proactive suggestions |
| `DAEMON` | Daemon worker mode |
| `BRIDGE_MODE` | Remote bridge support |
| `VOICE_MODE` | Voice interaction |
| `ABLATION_BASELINE` | A/B testing baseline |

---

## Session & History Management

Sessions are identified by a UUID generated at startup. Session data is persisted to `~/.claude/sessions/` and includes:
- Full conversation message history
- Tool call inputs and outputs
- Token usage and cost
- Timestamps and metadata

Commands:
- `session` — list and switch sessions
- `resume` — resume the most recent session
- `history` — search conversation history
- `export` — export a session to markdown or JSON

---

## Remote & Bridge Modes

**Bridge mode** (`bridge/`) enables IDE integrations (VS Code, JetBrains) by running Claude Code as a subprocess and communicating over a structured protocol. `replBridge.ts` implements the host side; `remoteBridgeCore.ts` implements the subprocess side.

**Remote mode** (`remote/`, `CLAUDE_CODE_REMOTE=1`) runs Claude Code inside CCR (Claude Code Remote) infrastructure. Remote-safe command filtering ensures only approved operations run in multi-tenant environments.

**Worktree isolation** (`tools/EnterWorktreeTool/`) creates a temporary git worktree so agent changes are isolated from the main branch until explicitly merged. The worktree is automatically cleaned up if no changes are made.
