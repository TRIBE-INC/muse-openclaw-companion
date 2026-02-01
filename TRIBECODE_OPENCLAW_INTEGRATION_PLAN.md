# TribeCode x OpenClaw Integration Plan

## Plugin: `@tribecode/openclaw-skill` for ClawdBot via ClawhHub

**Version**: 1.1.0-draft
**Date**: 2026-01-31
**Status**: Planning (Review-corrected)

---

## 1. Executive Summary

This document outlines the integration plan for publishing a **TribeCode skill** to [ClawhHub](https://www.clawhub.ai/) that enables OpenClaw (ClawdBot) users to install and use the full TRIBE CLI directly from their OpenClaw agent. The skill will:

1. Install the TRIBE CLI binary (`@_xtribe/cli`) on the target device
2. Expose all 60+ TRIBE CLI functions as OpenClaw agent tools
3. Register as a SKILL.md-based skill for conversational access
4. Publish to ClawhHub as a discoverable community skill
5. Provide a companion OpenClaw extension (plugin) for deeper integration

---

## 2. Platform Analysis

### 2.1 OpenClaw Architecture

**OpenClaw** is a local-first personal AI assistant (135k+ GitHub stars) that runs on user devices and connects to 13+ messaging platforms (WhatsApp, Telegram, Slack, Discord, Signal, iMessage, Teams, etc.).

**Core Architecture**:
```
Messaging Channels --> Gateway (ws://127.0.0.1:18789) --> Pi agent (RPC)
                                    |
                    CLI | WebChat UI | macOS app | iOS/Android nodes
```

**Key Integration Points**:
- **Gateway**: WebSocket control plane managing sessions, channels, tools, events
- **Skills**: Markdown-based at `~/.openclaw/workspace/skills/<name>/SKILL.md`
- **Plugins (Extensions)**: npm packages installed to `~/.openclaw/extensions/`
- **Tools**: Agent-callable functions registered via plugin registry
- **ClawHub**: Public registry at clawhub.ai for sharing skills and plugins
- **Runtime**: Node.js >= 22, pnpm monorepo

### 2.2 OpenClaw Plugin System (Technical Details)

**Two integration layers exist**:

| Layer | Format | Location | Registration |
|-------|--------|----------|--------------|
| **Skills** | `SKILL.md` markdown | `~/.openclaw/workspace/skills/<name>/` | Auto-discovered, conversational |
| **Extensions** | npm package with `openclaw.extensions` | `~/.openclaw/extensions/` | Plugin registry, tools/hooks/routes |

**Extension Manifest** (`package.json`):
```json
{
  "name": "@tribecode/openclaw",
  "version": "1.0.0",
  "description": "TribeCode AI analytics and agent orchestration for OpenClaw",
  "openclaw": {
    "extensions": ["./index.ts"]
  }
}
```

**Plugin Manifest** (`openclaw.plugin.json`):
```json
{
  "id": "tribecode",
  "configSchema": {
    "type": "object",
    "properties": {
      "apiKey": { "type": "string", "sensitive": true },
      "tutorServer": { "type": "string", "default": "https://tutor.tribecode.ai" },
      "autoSync": { "type": "boolean", "default": true }
    }
  }
}
```

**Extension Entry Point Convention**:

Extensions must export an object with `id`, `name`, and a `register(api)` method (NOT a bare `activate()` function):
```typescript
import type { OpenClawPluginApi } from "openclaw/plugin-sdk";

export default {
  id: "tribecode",
  name: "TribeCode",
  register(api: OpenClawPluginApi) {
    // Register tools, hooks, services, etc.
  }
};
```

**Plugin Registry** supports registering:
- **Tools**: Agent-callable functions (allowlist-gated) via `api.registerTool()`
- **Hooks**: Event handlers for lifecycle events via `api.registerHook()`
- **HTTP Routes**: Webhook endpoints via `api.registerHttpRoute()`
- **Services**: Background services via `api.registerService()`
- **CLI Commands**: Gateway CLI extensions via `api.registerCli()` / `api.registerCommand()`
- **Providers**: AI model providers via `api.registerProvider()`
- **Channels**: Messaging channels via `api.registerChannel()`
- **Gateway Methods**: RPC methods via `api.registerGatewayMethod()`

**registerTool API** (critical -- differs from naive expectation):
```typescript
// Signature: api.registerTool(tool: AnyAgentTool | OpenClawPluginToolFactory, opts?)
// Factory pattern (preferred for multiple tools):
api.registerTool((ctx) => [
  { name: "tool_a", description: "...", parameters: {...}, execute: async (args) => {...} },
  { name: "tool_b", description: "...", parameters: {...}, execute: async (args) => {...} },
], { names: ["tool_a", "tool_b"] });

// Single tool pattern:
api.registerTool({
  name: "my_tool",
  description: "...",
  parameters: { type: "object", properties: {...} },
  execute: async (args, ctx) => { return "result"; }
});
```

**Key API fields**:
- `api.logger` (NOT `api.log`) - logging interface
- `api.pluginConfig` (NOT `api.config`) - plugin configuration access

**Installation Mechanism**:
- From npm: `npm pack` + extract to `~/.openclaw/extensions/`
- Validates `package.json` has `openclaw.extensions` array
- Runs `npm install --omit=dev` for dependencies
- Backup/rollback on update failure
- Default timeout: 120s (300s for deps)

**Tool Access Control**:
- Optional tools require allowlist approval
- Three approval paths: direct tool name, plugin ID, or `group:plugins`
- Conflict detection prevents duplicate tool names

### 2.3 TribeCode CLI

**TRIBE CLI** is a Go binary distributed via npm (`@_xtribe/cli`), providing:

**Core Categories** (60+ commands across 15+ groups):

| Category | Key Commands |
|----------|-------------|
| **Telemetry** | `enable`, `disable`, `status`, `logs`, `config`, `realtime` |
| **Authentication** | `login`, `logout`, `auth-status`, `account-info`, `reset-session` |
| **Knowledge Base** | `kb save/search/list/get/tag/delete/sync/stats/export/import/daemon/extract` (12 subcommands) |
| **Sessions** | `sessions list/read/search/events/context` (5 subcommands) |
| **MUSE (Interactive)** | `muse start/status/spawn/prompt/output/attach/clean/cleanup/agents/circuit/review/monitor/watchdog/kill/celebrate/autoscale/negotiate/negotiate-accept/negotiate-refine` (18+ subcommands) |
| **CIRCUIT (Autonomous)** | `circuit list/spawn/kill/attach/restart/status/metrics/sync/next/dashboard/watchdog/auto/logs/issue/clean/heartbeat/monitor` (17 subcommands) |
| **Cluster Management** | `cluster-on`, `cluster-start`, `cluster-stop`, `cluster-delete`, `cluster-status` |
| **Task Management** | `submit-prompt`, `list-tasks`, `show-task`, `list-projects`, `reclaim-tasks` |
| **Workers** | `worker-status`, `scale-workers`, `check-autoscaling` |
| **Server** | `server start/stop/status/config/logs` (config has sub: `set/database/port/reset`) |
| **Agents** | `agents list/register/status/heartbeat/deregister` (5 subcommands) |
| **Memory** | `memory get/set/list/delete/clear` (5 subcommands) |
| **Traces** | `traces list/get/latest` (3 subcommands) |
| **Config** | `config show/set/reset/reset-sync/purge/localhost/production/path` (8 subcommands) |
| **Data Management** | `sync start/stop/stop-all/list`, `sync-insights`, `import` |
| **Disk** | `disk status/expand/provider/recommendations` (4 subcommands) |
| **System** | `fix`, `diagnose`, `cleanup`, `cleanup-status`, `cleanup-schedule`, `version`, `uninstall`, `reset` |
| **MCP** | `mcp serve` |
| **Leaderboard** | `leaderboard show/record/history/clear` (4 subcommands) |

**Installation**: `npx @_xtribe/cli@latest` --> downloads platform binary to `~/.tribe/bin/tribe`

**Build Details**:
- Language: Go 1.24.5
- UI: Bubble Tea (charmbracelet) TUI framework
- Binary: Statically compiled for macOS (Intel/ARM64) and Linux (x86_64)
- Config: `~/.tribe/`

**TUI vs JSON Mode** (important for subprocess integration):

Several commands use Bubble Tea interactive TUI and will NOT work in headless subprocess mode. These must be run with `--format json` or avoided:

| Command | Mode | Subprocess Strategy |
|---------|------|-------------------|
| `tribe logs` | TUI (live stream) | Use `--format json` or `--tail N` |
| `tribe circuit dashboard` | TUI | Use `circuit metrics --format json` instead |
| `tribe muse monitor` | TUI | Use `muse status --format json` instead |
| `tribe realtime` | TUI (live stream) | Use periodic `status --format json` polling |
| `tribe login` | TUI (browser OAuth) | See OAuth section below |
| `tribe muse attach` | Interactive stdin | Not supported in subprocess; use `muse prompt` |
| `tribe circuit attach` | Interactive stdin | Not supported in subprocess; use `circuit status` |

**OAuth / Login in Headless Environments**:

`tribe login` opens a browser for OAuth flow, which won't work in headless subprocess. Strategies:
1. **Pre-authenticate**: Guide user to run `tribe login` in their terminal first
2. **Token check**: Use `tribe auth-status --format json` to verify before attempting authenticated commands
3. **Fail gracefully**: If not authenticated, return a clear message asking user to run login manually

---

## 3. Integration Architecture

### 3.1 Two-Layer Approach

We will deliver the integration as **two complementary packages**:

```
Layer 1: SKILL.md (Conversational - Quick Start)
  - Natural language access to TRIBE commands
  - Zero-code setup via ClawHub install
  - Auto-installs TRIBE CLI binary on first use

Layer 2: OpenClaw Extension (Programmatic - Full Power)
  - Registered agent tools for each TRIBE command
  - Background telemetry sync service
  - Webhook integration for real-time events
  - Deep gateway integration
```

### 3.2 System Architecture

```
                    OpenClaw Gateway
                         |
            +------------+------------+
            |                         |
    SKILL.md Layer              Extension Layer
    (conversational)            (programmatic)
            |                         |
            v                         v
     tribe-skill/               @tribecode/openclaw
     SKILL.md                   extension package
            |                         |
            +--------+--------+-------+
                     |
              TRIBE CLI Binary
              (~/.tribe/bin/tribe)
                     |
            +--------+--------+
            |        |        |
       tribecode  Self-Host  MUSE/CIRCUIT
       Cloud API  Server     Agents
```

### 3.3 Directory Structure

```
openclaw-tribecode/
├── skill/
│   └── SKILL.md                    # ClawHub skill (conversational layer)
├── extension/
│   ├── package.json                # npm package (@tribecode/openclaw)
│   ├── openclaw.plugin.json        # Extension manifest
│   ├── index.ts                    # Extension entry point
│   ├── tools/
│   │   ├── telemetry.ts            # enable, disable, status, logs
│   │   ├── knowledge.ts            # search, recall, extract, query, kb
│   │   ├── sessions.ts             # sessions list, read, search, events
│   │   ├── muse.ts                 # MUSE agent orchestration tools
│   │   ├── circuit.ts              # CIRCUIT autonomous agent tools
│   │   ├── cluster.ts              # Cluster management tools
│   │   ├── tasks.ts                # Task management tools
│   │   ├── server.ts               # Self-hosted server tools
│   │   ├── agents.ts               # Agent management tools
│   │   ├── memory.ts               # Agent memory tools
│   │   ├── traces.ts               # Reasoning traces tools
│   │   ├── config.ts               # Configuration management tools
│   │   ├── disk.ts                 # Disk management tools
│   │   └── system.ts               # Diagnostics, cleanup, version
│   ├── services/
│   │   ├── installer.ts            # CLI binary installer/updater
│   │   ├── sync.ts                 # Background telemetry sync
│   │   └── health.ts               # Health monitoring service
│   ├── hooks/
│   │   └── lifecycle.ts            # Session start/end hooks
│   └── lib/
│       ├── cli-runner.ts           # Subprocess wrapper for tribe binary
│       └── config.ts               # Configuration management
├── README.md
├── LICENSE
└── clawhub.yaml                    # ClawHub registry metadata
```

---

## 4. Implementation Plan

### Phase 1: SKILL.md (ClawHub Quick Start)

**Goal**: Get a working TRIBE skill on ClawHub that users can install with one command.

#### 4.1.1 Skill Definition

Create `skill/SKILL.md`:

```markdown
---
name: tribecode
description: AI analytics, tribal knowledge search, and agent orchestration via TribeCode
metadata:
  openclaw:
    requires:
      bins:
        - tribe
    install: "npx @_xtribe/cli@latest"
---

# TribeCode Skill

You are a TribeCode assistant that helps users track AI usage analytics,
search their coding history, and orchestrate AI agents using the TRIBE CLI.

## Setup

Before using any commands, verify TRIBE CLI is installed:
```bash
if ! command -v tribe &> /dev/null && [ ! -f ~/.tribe/bin/tribe ]; then
  npx @_xtribe/cli@latest
fi
```

If the user hasn't logged in yet:
```bash
~/.tribe/bin/tribe auth-status
```
If not authenticated, guide them to run `~/.tribe/bin/tribe login` in their terminal (requires browser for OAuth).

## Available Commands

### Analytics & Telemetry
- **Enable tracking**: `~/.tribe/bin/tribe enable`
- **Disable tracking**: `~/.tribe/bin/tribe disable`
- **Check status**: `~/.tribe/bin/tribe status`
- **View logs**: `~/.tribe/bin/tribe logs --format json`
- **Configuration**: `~/.tribe/bin/tribe config`
- **Realtime monitoring**: `~/.tribe/bin/tribe status` (use status in JSON mode, realtime is TUI-only)

### Knowledge Base (Tribal Knowledge)
- **Save document**: `~/.tribe/bin/tribe kb save "<title>" --content "<content>"`
- **Search knowledge**: `~/.tribe/bin/tribe kb search "<query>" --format json`
- **List knowledge**: `~/.tribe/bin/tribe kb list --format json`
- **Get document**: `~/.tribe/bin/tribe kb get <id> --format json`
- **Tag document**: `~/.tribe/bin/tribe kb tag <id> --tags "<tag1>,<tag2>"`
- **Delete document**: `~/.tribe/bin/tribe kb delete <id>`
- **Sync knowledge**: `~/.tribe/bin/tribe kb sync`
- **KB statistics**: `~/.tribe/bin/tribe kb stats --format json`
- **Export all**: `~/.tribe/bin/tribe kb export --format json`
- **Import from file**: `~/.tribe/bin/tribe kb import <file>`
- **Extract patterns**: `~/.tribe/bin/tribe kb extract`

### Session Management
- **List sessions**: `~/.tribe/bin/tribe sessions list --format json`
- **Read session**: `~/.tribe/bin/tribe sessions read <id> --format json`
- **Search sessions**: `~/.tribe/bin/tribe sessions search "<query>" --format json`
- **Session events**: `~/.tribe/bin/tribe sessions events <id> --format json`

### Search & Recall
- **Search sessions**: `~/.tribe/bin/tribe search "<query>" --limit 10 --format json`
- **Recall session**: `~/.tribe/bin/tribe recall <session-id> --format json`
- **Extract code/commands**: `~/.tribe/bin/tribe extract <session-id> --type code`
- **Query insights**: `~/.tribe/bin/tribe query insights --time-range 7d --format json`

### MUSE (Interactive Agent Orchestration)
- **Start leader**: `~/.tribe/bin/tribe muse start`
- **Spawn worker**: `~/.tribe/bin/tribe muse spawn "<task>" [name]`
- **Check agents**: `~/.tribe/bin/tribe muse agents --format json`
- **Get output**: `~/.tribe/bin/tribe muse output <session>`
- **Send prompt**: `~/.tribe/bin/tribe muse prompt <session> "<message>"`
- **Check status**: `~/.tribe/bin/tribe muse status --format json`
- **Kill instance**: `~/.tribe/bin/tribe muse kill <session>`
- **Clean resources**: `~/.tribe/bin/tribe muse clean`
- **Auto-scale on**: `~/.tribe/bin/tribe muse autoscale on`
- **Auto-scale off**: `~/.tribe/bin/tribe muse autoscale off`
- **Auto-scale status**: `~/.tribe/bin/tribe muse autoscale status --format json`
- **Start negotiation**: `~/.tribe/bin/tribe muse negotiate`
- **Accept negotiation**: `~/.tribe/bin/tribe muse negotiate-accept`
- **Refine negotiation**: `~/.tribe/bin/tribe muse negotiate-refine`
- **Review**: `~/.tribe/bin/tribe muse review`
- **Watchdog**: `~/.tribe/bin/tribe muse watchdog`
- **MUSE circuit spawn**: `~/.tribe/bin/tribe muse circuit spawn`
- **MUSE circuit status**: `~/.tribe/bin/tribe muse circuit status --format json`
- **Stop leader**: `~/.tribe/bin/tribe muse stop` (if available, or use `muse kill`)

### CIRCUIT (Autonomous Agents)
- **List agents**: `~/.tribe/bin/tribe circuit list --format json`
- **Spawn for issue**: `~/.tribe/bin/tribe circuit spawn <issue>`
- **Kill agent**: `~/.tribe/bin/tribe circuit kill <id>`
- **Restart agent**: `~/.tribe/bin/tribe circuit restart <id>`
- **Agent status**: `~/.tribe/bin/tribe circuit status --format json`
- **Metrics**: `~/.tribe/bin/tribe circuit metrics --format json`
- **Sync data**: `~/.tribe/bin/tribe circuit sync`
- **Next task**: `~/.tribe/bin/tribe circuit next --format json`
- **Auto-assign**: `~/.tribe/bin/tribe circuit auto`
- **Logs**: `~/.tribe/bin/tribe circuit logs --format json`
- **Issues**: `~/.tribe/bin/tribe circuit issue`
- **Clean**: `~/.tribe/bin/tribe circuit clean`
- **Heartbeat**: `~/.tribe/bin/tribe circuit heartbeat`
- **Watchdog**: `~/.tribe/bin/tribe circuit watchdog`
- **Monitor**: `~/.tribe/bin/tribe circuit monitor --format json`

### Agent Management
- **List agents**: `~/.tribe/bin/tribe agents list --format json`
- **Register agent**: `~/.tribe/bin/tribe agents register`
- **Agent status**: `~/.tribe/bin/tribe agents status --format json`
- **Heartbeat**: `~/.tribe/bin/tribe agents heartbeat`
- **Deregister**: `~/.tribe/bin/tribe agents deregister <id>`

### Agent Memory
- **Get memory**: `~/.tribe/bin/tribe memory get <key>`
- **Set memory**: `~/.tribe/bin/tribe memory set <key> "<value>"`
- **List memory**: `~/.tribe/bin/tribe memory list --format json`
- **Delete entry**: `~/.tribe/bin/tribe memory delete <key>`
- **Clear all**: `~/.tribe/bin/tribe memory clear`

### Reasoning Traces
- **List traces**: `~/.tribe/bin/tribe traces list --format json`
- **Get trace**: `~/.tribe/bin/tribe traces get <id> --format json`
- **Latest trace**: `~/.tribe/bin/tribe traces latest --format json`

### Task Management
- **Submit work**: `~/.tribe/bin/tribe submit-prompt --prompt "<description>"`
- **List tasks**: `~/.tribe/bin/tribe list-tasks --format json`
- **Show task**: `~/.tribe/bin/tribe show-task <id> --format json`
- **List projects**: `~/.tribe/bin/tribe list-projects --format json`
- **Reclaim tasks**: `~/.tribe/bin/tribe reclaim-tasks`

### Cluster Management
- **Start cluster**: `~/.tribe/bin/tribe cluster-on`
- **Stop cluster**: `~/.tribe/bin/tribe cluster-stop`
- **Cluster status**: `~/.tribe/bin/tribe cluster-status --format json`
- **Delete cluster**: `~/.tribe/bin/tribe cluster-delete`

### Worker Management
- **Worker status**: `~/.tribe/bin/tribe worker-status --format json`
- **Scale workers**: `~/.tribe/bin/tribe scale-workers <count>`
- **Check autoscaling**: `~/.tribe/bin/tribe check-autoscaling --format json`

### Configuration
- **Show config**: `~/.tribe/bin/tribe config show`
- **Set config value**: `~/.tribe/bin/tribe config set <key> <value>`
- **Reset config**: `~/.tribe/bin/tribe config reset`
- **Show config path**: `~/.tribe/bin/tribe config path`

### Server (Self-Hosted)
- **Start server**: `~/.tribe/bin/tribe server start`
- **Stop server**: `~/.tribe/bin/tribe server stop`
- **Server status**: `~/.tribe/bin/tribe server status --format json`
- **Server config**: `~/.tribe/bin/tribe server config set <key> <value>`
- **Server logs**: `~/.tribe/bin/tribe server logs --format json`

### Data & Sync
- **Sync data**: `~/.tribe/bin/tribe sync`
- **Sync insights**: `~/.tribe/bin/tribe sync-insights`
- **Import data**: `~/.tribe/bin/tribe import <file>`

### Disk Management
- **Disk status**: `~/.tribe/bin/tribe disk status --format json`
- **Expand disk**: `~/.tribe/bin/tribe disk expand`
- **Disk provider**: `~/.tribe/bin/tribe disk provider`
- **Recommendations**: `~/.tribe/bin/tribe disk recommendations --format json`

### System
- **Diagnostics**: `~/.tribe/bin/tribe diagnose`
- **Fix issues**: `~/.tribe/bin/tribe fix`
- **Cleanup**: `~/.tribe/bin/tribe cleanup`
- **Version**: `~/.tribe/bin/tribe version`

## Important Notes

- Commands using `--format json` produce machine-readable output. Always parse JSON and present in human-readable format.
- Some commands (`logs`, `realtime`, `circuit dashboard`, `muse monitor`, `muse attach`, `circuit attach`) are TUI/interactive and cannot run in subprocess mode. Use the JSON alternatives listed above.
- `tribe login` requires a browser. If user is not authenticated, ask them to run it manually.
- For long-running operations (MUSE, CIRCUIT), use appropriate timeouts (60-120s).

## Response Format

When presenting search results or session data, format them clearly with:
- Session IDs as clickable references
- Code blocks for extracted code
- Summary statistics for analytics queries
- Status indicators for agent health
```

#### 4.1.2 ClawHub Publication

**Registry Metadata** (`clawhub.yaml`):
```yaml
name: tribecode
version: 1.0.0
description: AI analytics, tribal knowledge, and agent orchestration
author: TRIBE-INC
homepage: https://tribecode.ai
repository: https://github.com/TRIBE-INC/openclaw-tribecode
license: MIT
tags:
  - analytics
  - ai-tools
  - agent-orchestration
  - knowledge-base
  - telemetry
  - productivity
category: developer-tools
platforms:
  - macos
  - linux
```

**Installation command** (for users):
```bash
# Via ClawHub CLI
clawhub install tribecode

# Or manual
mkdir -p ~/.openclaw/workspace/skills/tribecode
cp SKILL.md ~/.openclaw/workspace/skills/tribecode/SKILL.md
```

### Phase 2: OpenClaw Extension (Deep Integration)

**Goal**: Registered tools, background services, and lifecycle hooks.

#### 4.2.1 Extension Entry Point

`extension/index.ts`:
```typescript
import type { OpenClawPluginApi } from "openclaw/plugin-sdk";
import { registerTelemetryTools } from "./tools/telemetry";
import { registerKnowledgeTools } from "./tools/knowledge";
import { registerSessionTools } from "./tools/sessions";
import { registerMuseTools } from "./tools/muse";
import { registerCircuitTools } from "./tools/circuit";
import { registerClusterTools } from "./tools/cluster";
import { registerTaskTools } from "./tools/tasks";
import { registerServerTools } from "./tools/server";
import { registerAgentTools } from "./tools/agents";
import { registerMemoryTools } from "./tools/memory";
import { registerTracesTools } from "./tools/traces";
import { registerConfigTools } from "./tools/config";
import { registerDiskTools } from "./tools/disk";
import { registerSystemTools } from "./tools/system";
import { startSyncService } from "./services/sync";
import { ensureCliInstalled } from "./services/installer";
import { registerLifecycleHooks } from "./hooks/lifecycle";

export default {
  id: "tribecode",
  name: "TribeCode",

  register(api: OpenClawPluginApi) {
    const logger = api.logger;

    // Ensure CLI binary is available
    ensureCliInstalled(logger);

    // Register all tool categories
    registerTelemetryTools(api);
    registerKnowledgeTools(api);
    registerSessionTools(api);
    registerMuseTools(api);
    registerCircuitTools(api);
    registerClusterTools(api);
    registerTaskTools(api);
    registerServerTools(api);
    registerAgentTools(api);
    registerMemoryTools(api);
    registerTracesTools(api);
    registerConfigTools(api);
    registerDiskTools(api);
    registerSystemTools(api);

    // Start background services
    if (api.pluginConfig?.autoSync) {
      startSyncService(api);
    }

    // Register lifecycle hooks
    registerLifecycleHooks(api);

    logger.info("TribeCode extension activated");
  }
};
```

#### 4.2.2 CLI Runner (Core Abstraction)

`extension/lib/cli-runner.ts`:
```typescript
import { execFile } from "node:child_process";
import { existsSync } from "node:fs";
import { homedir } from "node:os";
import { join } from "node:path";

const TRIBE_BIN = join(homedir(), ".tribe", "bin", "tribe");

export interface CliResult {
  stdout: string;
  stderr: string;
  exitCode: number;
}

// Timeouts by command category
const TIMEOUT_FAST = 15_000;     // status checks, version
const TIMEOUT_DEFAULT = 30_000;  // standard commands
const TIMEOUT_SLOW = 60_000;     // sync, export, import
const TIMEOUT_LONG = 120_000;    // MUSE/CIRCUIT spawn, cluster operations

export function getTimeout(command: string): number {
  if (/^(version|auth-status|status|config)/.test(command)) return TIMEOUT_FAST;
  if (/^(muse|circuit|cluster|server start)/.test(command)) return TIMEOUT_LONG;
  if (/^(sync|export|import|kb (sync|export|import))/.test(command)) return TIMEOUT_SLOW;
  return TIMEOUT_DEFAULT;
}

export async function runTribe(
  args: string[],
  options?: { timeout?: number; json?: boolean }
): Promise<CliResult> {
  const bin = existsSync(TRIBE_BIN) ? TRIBE_BIN : "tribe";
  const finalArgs = options?.json ? [...args, "--format", "json"] : args;
  const timeout = options?.timeout ?? getTimeout(args.join(" "));

  return new Promise((resolve, reject) => {
    execFile(bin, finalArgs, { timeout }, (error, stdout, stderr) => {
      resolve({
        stdout: stdout.toString(),
        stderr: stderr.toString(),
        exitCode: error?.code ?? 0,
      });
    });
  });
}

export function parseJsonOutput<T>(result: CliResult): T | null {
  try {
    return JSON.parse(result.stdout) as T;
  } catch {
    return null;
  }
}
```

#### 4.2.3 Tool Registration (Example: Knowledge Tools)

`extension/tools/knowledge.ts`:
```typescript
import type { OpenClawPluginApi } from "openclaw/plugin-sdk";
import { runTribe, parseJsonOutput } from "../lib/cli-runner";

export function registerKnowledgeTools(api: OpenClawPluginApi) {
  // Use factory pattern for registering multiple tools
  api.registerTool((ctx) => [
    {
      name: "tribe_search",
      description: "Search past coding sessions for patterns, solutions, and context",
      parameters: {
        type: "object",
        properties: {
          query: { type: "string", description: "Search query" },
          limit: { type: "number", description: "Max results (default: 10)" },
          project: { type: "string", description: "Filter by project name" },
          timeRange: { type: "string", description: "Time window: 7d, 30d, 90d" },
        },
        required: ["query"],
      },
      execute: async ({ query, limit, project, timeRange }) => {
        const args = ["search", query];
        if (limit) args.push("--limit", String(limit));
        if (project) args.push("--project", project);
        if (timeRange) args.push("--time-range", timeRange);
        const result = await runTribe(args, { json: true });
        return parseJsonOutput(result) ?? result.stdout;
      },
    },
    {
      name: "tribe_recall",
      description: "Retrieve a detailed summary of a specific coding session",
      parameters: {
        type: "object",
        properties: {
          sessionId: { type: "string", description: "Session ID to recall" },
        },
        required: ["sessionId"],
      },
      execute: async ({ sessionId }) => {
        const result = await runTribe(["recall", sessionId], { json: true });
        return parseJsonOutput(result) ?? result.stdout;
      },
    },
    {
      name: "tribe_extract",
      description: "Extract specific content types (code, commands, files, edits) from a session",
      parameters: {
        type: "object",
        properties: {
          sessionId: { type: "string", description: "Session ID" },
          type: { type: "string", enum: ["code", "commands", "files", "edits"] },
          limit: { type: "number", description: "Max items to extract" },
        },
        required: ["sessionId", "type"],
      },
      execute: async ({ sessionId, type, limit }) => {
        const args = ["extract", sessionId, "--type", type];
        if (limit) args.push("--limit", String(limit));
        const result = await runTribe(args, { json: true });
        return parseJsonOutput(result) ?? result.stdout;
      },
    },
    {
      name: "tribe_query",
      description: "Query TribeCode for sessions, insights, or events",
      parameters: {
        type: "object",
        properties: {
          type: { type: "string", enum: ["sessions", "insights", "events"] },
          timeRange: { type: "string", description: "Time range filter" },
          limit: { type: "number", description: "Max results" },
        },
        required: ["type"],
      },
      execute: async ({ type, timeRange, limit }) => {
        const args = ["query", type];
        if (timeRange) args.push("--time-range", timeRange);
        if (limit) args.push("--limit", String(limit));
        const result = await runTribe(args, { json: true });
        return parseJsonOutput(result) ?? result.stdout;
      },
    },
    {
      name: "tribe_kb_save",
      description: "Save a document to the TribeCode knowledge base",
      parameters: {
        type: "object",
        properties: {
          title: { type: "string", description: "Document title" },
          content: { type: "string", description: "Document content" },
          tags: { type: "string", description: "Comma-separated tags" },
        },
        required: ["title", "content"],
      },
      execute: async ({ title, content, tags }) => {
        const args = ["kb", "save", title, "--content", content];
        if (tags) args.push("--tags", tags);
        const result = await runTribe(args);
        return result.stdout;
      },
    },
    {
      name: "tribe_kb_search",
      description: "Search the TribeCode knowledge base",
      parameters: {
        type: "object",
        properties: {
          query: { type: "string", description: "Search query" },
        },
        required: ["query"],
      },
      execute: async ({ query }) => {
        const result = await runTribe(["kb", "search", query], { json: true });
        return parseJsonOutput(result) ?? result.stdout;
      },
    },
    {
      name: "tribe_kb_list",
      description: "List all documents in the TribeCode knowledge base",
      parameters: { type: "object", properties: {} },
      execute: async () => {
        const result = await runTribe(["kb", "list"], { json: true });
        return parseJsonOutput(result) ?? result.stdout;
      },
    },
    {
      name: "tribe_kb_get",
      description: "Get a specific document from the knowledge base by ID",
      parameters: {
        type: "object",
        properties: {
          id: { type: "string", description: "Document ID" },
        },
        required: ["id"],
      },
      execute: async ({ id }) => {
        const result = await runTribe(["kb", "get", id], { json: true });
        return parseJsonOutput(result) ?? result.stdout;
      },
    },
    {
      name: "tribe_kb_stats",
      description: "Show knowledge base statistics",
      parameters: { type: "object", properties: {} },
      execute: async () => {
        const result = await runTribe(["kb", "stats"], { json: true });
        return parseJsonOutput(result) ?? result.stdout;
      },
    },
    {
      name: "tribe_kb_delete",
      description: "Delete a document from the knowledge base",
      parameters: {
        type: "object",
        properties: {
          id: { type: "string", description: "Document ID to delete" },
        },
        required: ["id"],
      },
      execute: async ({ id }) => {
        const result = await runTribe(["kb", "delete", id]);
        return result.stdout;
      },
    },
    {
      name: "tribe_kb_tag",
      description: "Add tags to a knowledge base document",
      parameters: {
        type: "object",
        properties: {
          id: { type: "string", description: "Document ID" },
          tags: { type: "string", description: "Comma-separated tags to add" },
        },
        required: ["id", "tags"],
      },
      execute: async ({ id, tags }) => {
        const result = await runTribe(["kb", "tag", id, "--tags", tags]);
        return result.stdout;
      },
    },
    {
      name: "tribe_kb_sync",
      description: "Sync knowledge base with remote server",
      parameters: { type: "object", properties: {} },
      execute: async () => {
        const result = await runTribe(["kb", "sync"], { timeout: 60_000 });
        return result.stdout;
      },
    },
    {
      name: "tribe_kb_export",
      description: "Export all knowledge base documents as JSON",
      parameters: { type: "object", properties: {} },
      execute: async () => {
        const result = await runTribe(["kb", "export"], { json: true, timeout: 60_000 });
        return parseJsonOutput(result) ?? result.stdout;
      },
    },
    {
      name: "tribe_kb_import",
      description: "Import knowledge base documents from a JSON file",
      parameters: {
        type: "object",
        properties: {
          file: { type: "string", description: "Path to JSON file to import" },
        },
        required: ["file"],
      },
      execute: async ({ file }) => {
        const result = await runTribe(["kb", "import", file], { timeout: 60_000 });
        return result.stdout;
      },
    },
  ], {
    names: [
      "tribe_search", "tribe_recall", "tribe_extract", "tribe_query",
      "tribe_kb_save", "tribe_kb_search", "tribe_kb_list", "tribe_kb_get",
      "tribe_kb_stats", "tribe_kb_delete", "tribe_kb_tag", "tribe_kb_sync",
      "tribe_kb_export", "tribe_kb_import",
    ],
  });
}
```

#### 4.2.4 CLI Installer Service

`extension/services/installer.ts`:
```typescript
import { execSync } from "node:child_process";
import { existsSync, mkdirSync } from "node:fs";
import { homedir } from "node:os";
import { join } from "node:path";

const TRIBE_DIR = join(homedir(), ".tribe");
const TRIBE_BIN = join(TRIBE_DIR, "bin", "tribe");

export async function ensureCliInstalled(logger: any): Promise<boolean> {
  if (existsSync(TRIBE_BIN)) {
    logger.info("TRIBE CLI found at " + TRIBE_BIN);
    return true;
  }

  logger.info("TRIBE CLI not found, installing via npm...");

  try {
    mkdirSync(join(TRIBE_DIR, "bin"), { recursive: true });
    execSync("npx @_xtribe/cli@latest --version", {
      timeout: 120_000,
      stdio: "pipe",
    });

    if (existsSync(TRIBE_BIN)) {
      logger.info("TRIBE CLI installed successfully");
      return true;
    }
  } catch (err) {
    logger.warn("Failed to install TRIBE CLI: " + err);
  }

  return false;
}

export async function updateCli(logger: any): Promise<string | null> {
  try {
    execSync("npx @_xtribe/cli@latest --version", {
      timeout: 120_000,
      stdio: "pipe",
    });
    const version = execSync(TRIBE_BIN + " version", { encoding: "utf8" }).trim();
    logger.info("TRIBE CLI updated to " + version);
    return version;
  } catch {
    return null;
  }
}
```

#### 4.2.5 Background Sync Service

`extension/services/sync.ts`:
```typescript
import type { OpenClawPluginApi } from "openclaw/plugin-sdk";
import { runTribe } from "../lib/cli-runner";

let syncInterval: NodeJS.Timer | null = null;

export function startSyncService(api: OpenClawPluginApi) {
  const intervalMs = 5 * 60 * 1000; // 5 minutes

  syncInterval = setInterval(async () => {
    try {
      await runTribe(["sync"], { timeout: 60_000 });
    } catch (err) {
      api.logger.warn("Telemetry sync failed: " + err);
    }
  }, intervalMs);

  api.logger.info("TribeCode sync service started (interval: 5m)");
}

export function stopSyncService() {
  if (syncInterval) {
    clearInterval(syncInterval);
    syncInterval = null;
  }
}
```

### Phase 3: Complete Tool Coverage

All TRIBE CLI commands mapped to OpenClaw tools (60+ commands across 15 categories):

#### Telemetry (6 tools)
| Tool Name | CLI Command | Notes |
|-----------|-------------|-------|
| `tribe_enable` | `tribe enable` | |
| `tribe_disable` | `tribe disable` | |
| `tribe_status` | `tribe status` | Use `--format json` |
| `tribe_logs` | `tribe logs --format json` | TUI mode not supported; use `--format json` or `--tail N` |
| `tribe_config_show` | `tribe config show` | |
| `tribe_realtime` | `tribe status --format json` | `realtime` is TUI-only; poll `status` instead |

#### Authentication (5 tools)
| Tool Name | CLI Command | Notes |
|-----------|-------------|-------|
| `tribe_login` | `tribe login` | Requires browser -- guide user to run manually |
| `tribe_logout` | `tribe logout` | |
| `tribe_auth_status` | `tribe auth-status` | Use `--format json` |
| `tribe_account_info` | `tribe account-info` | Use `--format json` |
| `tribe_reset_session` | `tribe reset-session` | |

#### Knowledge Base (14 tools)
| Tool Name | CLI Command | Notes |
|-----------|-------------|-------|
| `tribe_search` | `tribe search` | Top-level search |
| `tribe_recall` | `tribe recall` | Session recall |
| `tribe_extract` | `tribe extract` | Extract code/commands |
| `tribe_query` | `tribe query` | Query insights/events/sessions |
| `tribe_kb_save` | `tribe kb save` | |
| `tribe_kb_search` | `tribe kb search` | |
| `tribe_kb_list` | `tribe kb list` | |
| `tribe_kb_get` | `tribe kb get` | |
| `tribe_kb_tag` | `tribe kb tag` | |
| `tribe_kb_delete` | `tribe kb delete` | |
| `tribe_kb_sync` | `tribe kb sync` | Timeout: 60s |
| `tribe_kb_stats` | `tribe kb stats` | |
| `tribe_kb_export` | `tribe kb export` | Timeout: 60s |
| `tribe_kb_import` | `tribe kb import` | Timeout: 60s |

#### Sessions (5 tools)
| Tool Name | CLI Command | Notes |
|-----------|-------------|-------|
| `tribe_sessions_list` | `tribe sessions list` | Use `--format json` |
| `tribe_sessions_read` | `tribe sessions read` | Use `--format json` |
| `tribe_sessions_search` | `tribe sessions search` | Use `--format json` |
| `tribe_sessions_events` | `tribe sessions events` | Use `--format json` |
| `tribe_sessions_context` | `tribe sessions context` | |

#### MUSE (18 tools)
| Tool Name | CLI Command | Notes |
|-----------|-------------|-------|
| `tribe_muse_start` | `tribe muse start` | Timeout: 120s |
| `tribe_muse_status` | `tribe muse status` | Use `--format json` |
| `tribe_muse_spawn` | `tribe muse spawn` | Timeout: 120s |
| `tribe_muse_prompt` | `tribe muse prompt` | |
| `tribe_muse_output` | `tribe muse output` | |
| `tribe_muse_agents` | `tribe muse agents` | Use `--format json` |
| `tribe_muse_kill` | `tribe muse kill` | |
| `tribe_muse_clean` | `tribe muse clean` | |
| `tribe_muse_cleanup` | `tribe muse cleanup` | |
| `tribe_muse_review` | `tribe muse review` | |
| `tribe_muse_monitor` | `tribe muse status --format json` | `monitor` is TUI; use `status` |
| `tribe_muse_watchdog` | `tribe muse watchdog` | |
| `tribe_muse_celebrate` | `tribe muse celebrate` | |
| `tribe_muse_autoscale_on` | `tribe muse autoscale on` | |
| `tribe_muse_autoscale_off` | `tribe muse autoscale off` | |
| `tribe_muse_autoscale_status` | `tribe muse autoscale status` | Use `--format json` |
| `tribe_muse_negotiate` | `tribe muse negotiate` | |
| `tribe_muse_negotiate_accept` | `tribe muse negotiate-accept` | |

Note: `tribe muse attach` is interactive/TUI and cannot be used via subprocess. Use `tribe muse prompt` to send messages instead.

#### CIRCUIT (17 tools)
| Tool Name | CLI Command | Notes |
|-----------|-------------|-------|
| `tribe_circuit_list` | `tribe circuit list` | Use `--format json` |
| `tribe_circuit_spawn` | `tribe circuit spawn` | Timeout: 120s |
| `tribe_circuit_kill` | `tribe circuit kill` | |
| `tribe_circuit_restart` | `tribe circuit restart` | |
| `tribe_circuit_status` | `tribe circuit status` | Use `--format json` |
| `tribe_circuit_metrics` | `tribe circuit metrics` | Use `--format json` |
| `tribe_circuit_sync` | `tribe circuit sync` | Timeout: 60s |
| `tribe_circuit_next` | `tribe circuit next` | Use `--format json` |
| `tribe_circuit_dashboard` | `tribe circuit metrics --format json` | `dashboard` is TUI; use `metrics` |
| `tribe_circuit_watchdog` | `tribe circuit watchdog` | |
| `tribe_circuit_auto` | `tribe circuit auto` | |
| `tribe_circuit_logs` | `tribe circuit logs` | Use `--format json` |
| `tribe_circuit_issue` | `tribe circuit issue` | |
| `tribe_circuit_clean` | `tribe circuit clean` | |
| `tribe_circuit_heartbeat` | `tribe circuit heartbeat` | |
| `tribe_circuit_monitor` | `tribe circuit status --format json` | `monitor` is TUI; use `status` |
| `tribe_circuit_negotiate_refine` | `tribe muse negotiate-refine` | Via MUSE |

Note: `tribe circuit attach` is interactive/TUI and cannot be used via subprocess.

#### Agents (5 tools)
| Tool Name | CLI Command | Notes |
|-----------|-------------|-------|
| `tribe_agents_list` | `tribe agents list` | Use `--format json` |
| `tribe_agents_register` | `tribe agents register` | |
| `tribe_agents_status` | `tribe agents status` | Use `--format json` |
| `tribe_agents_heartbeat` | `tribe agents heartbeat` | |
| `tribe_agents_deregister` | `tribe agents deregister` | |

#### Memory (5 tools)
| Tool Name | CLI Command | Notes |
|-----------|-------------|-------|
| `tribe_memory_get` | `tribe memory get` | |
| `tribe_memory_set` | `tribe memory set` | |
| `tribe_memory_list` | `tribe memory list` | Use `--format json` |
| `tribe_memory_delete` | `tribe memory delete` | |
| `tribe_memory_clear` | `tribe memory clear` | |

#### Traces (3 tools)
| Tool Name | CLI Command | Notes |
|-----------|-------------|-------|
| `tribe_traces_list` | `tribe traces list` | Use `--format json` |
| `tribe_traces_get` | `tribe traces get` | Use `--format json` |
| `tribe_traces_latest` | `tribe traces latest` | Use `--format json` |

#### Task Management (5 tools)
| Tool Name | CLI Command | Notes |
|-----------|-------------|-------|
| `tribe_submit_prompt` | `tribe submit-prompt` | |
| `tribe_list_tasks` | `tribe list-tasks` | Use `--format json` |
| `tribe_show_task` | `tribe show-task` | Use `--format json` |
| `tribe_list_projects` | `tribe list-projects` | Use `--format json` |
| `tribe_reclaim_tasks` | `tribe reclaim-tasks` | |

#### Cluster (5 tools)
| Tool Name | CLI Command | Notes |
|-----------|-------------|-------|
| `tribe_cluster_on` | `tribe cluster-on` | Timeout: 120s |
| `tribe_cluster_start` | `tribe cluster-start` | Timeout: 120s |
| `tribe_cluster_stop` | `tribe cluster-stop` | |
| `tribe_cluster_delete` | `tribe cluster-delete` | |
| `tribe_cluster_status` | `tribe cluster-status` | Use `--format json` |

#### Workers (3 tools)
| Tool Name | CLI Command | Notes |
|-----------|-------------|-------|
| `tribe_worker_status` | `tribe worker-status` | Use `--format json` |
| `tribe_scale_workers` | `tribe scale-workers` | |
| `tribe_check_autoscaling` | `tribe check-autoscaling` | Use `--format json` |

#### Server (5 tools)
| Tool Name | CLI Command | Notes |
|-----------|-------------|-------|
| `tribe_server_start` | `tribe server start` | Timeout: 120s |
| `tribe_server_stop` | `tribe server stop` | |
| `tribe_server_status` | `tribe server status` | Use `--format json` |
| `tribe_server_config` | `tribe server config` | |
| `tribe_server_logs` | `tribe server logs` | Use `--format json` |

#### Configuration (8 tools)
| Tool Name | CLI Command | Notes |
|-----------|-------------|-------|
| `tribe_config_show` | `tribe config show` | |
| `tribe_config_set` | `tribe config set` | |
| `tribe_config_reset` | `tribe config reset` | |
| `tribe_config_reset_sync` | `tribe config reset-sync` | |
| `tribe_config_purge` | `tribe config purge` | |
| `tribe_config_localhost` | `tribe config localhost` | |
| `tribe_config_production` | `tribe config production` | |
| `tribe_config_path` | `tribe config path` | |

#### Data & Sync (5 tools)
| Tool Name | CLI Command | Notes |
|-----------|-------------|-------|
| `tribe_sync` | `tribe sync` | Timeout: 60s |
| `tribe_sync_insights` | `tribe sync-insights` | Timeout: 60s |
| `tribe_import` | `tribe import` | Timeout: 60s |
| `tribe_sync_start` | `tribe sync start` | |
| `tribe_sync_stop` | `tribe sync stop` | |

#### Disk Management (4 tools)
| Tool Name | CLI Command | Notes |
|-----------|-------------|-------|
| `tribe_disk_status` | `tribe disk status` | Use `--format json` |
| `tribe_disk_expand` | `tribe disk expand` | |
| `tribe_disk_provider` | `tribe disk provider` | |
| `tribe_disk_recommendations` | `tribe disk recommendations` | Use `--format json` |

#### System & Maintenance (8 tools)
| Tool Name | CLI Command | Notes |
|-----------|-------------|-------|
| `tribe_fix` | `tribe fix` | |
| `tribe_diagnose` | `tribe diagnose` | |
| `tribe_cleanup` | `tribe cleanup` | |
| `tribe_cleanup_status` | `tribe cleanup-status` | Use `--format json` |
| `tribe_cleanup_schedule` | `tribe cleanup-schedule` | |
| `tribe_version` | `tribe version` | |
| `tribe_uninstall` | `tribe uninstall` | |
| `tribe_reset` | `tribe reset` | |

#### Leaderboard (4 tools)
| Tool Name | CLI Command | Notes |
|-----------|-------------|-------|
| `tribe_leaderboard_show` | `tribe leaderboard show` | Use `--format json` |
| `tribe_leaderboard_record` | `tribe leaderboard record` | |
| `tribe_leaderboard_history` | `tribe leaderboard history` | Use `--format json` |
| `tribe_leaderboard_clear` | `tribe leaderboard clear` | |

#### MCP (1 tool)
| Tool Name | CLI Command | Notes |
|-----------|-------------|-------|
| `tribe_mcp_serve` | `tribe mcp serve` | Long-running; may not suit tool pattern |

**Total: ~120 tools across 16 categories** covering all 60+ CLI commands and subcommands.

---

## 5. ClawHub Publication Process

### 5.1 Skill Publication (Layer 1)

ClawHub skills are published as Git repositories. The process:

1. Create public repo: `github.com/TRIBE-INC/openclaw-tribecode`
2. Add `skill/SKILL.md` with `metadata.openclaw` frontmatter
3. Register on ClawHub via the submission process
4. Users install via: `clawhub install tribecode`

This copies `SKILL.md` to `~/.openclaw/workspace/skills/tribecode/SKILL.md`.

### 5.2 Extension Publication (Layer 2)

Extensions are published as npm packages:

1. Publish `@tribecode/openclaw` to npm registry
2. Users install via: `openclaw plugin install @tribecode/openclaw`
3. Package extracted to `~/.openclaw/extensions/@tribecode__openclaw/`
4. `openclaw.extensions` entry points loaded by Gateway

### 5.3 Combined Installation

Recommended user experience:

```bash
# From ClawHub (skill layer)
clawhub install tribecode

# From npm (extension layer)
openclaw plugin install @tribecode/openclaw

# Or from any OpenClaw chat channel (if /install command available)
/install tribecode
```

---

## 6. Security Considerations

### 6.1 Tool Access Control

OpenClaw uses allowlist-gated tool access. The TribeCode extension:

- All tools registered as "optional" requiring explicit user approval
- Approval via: tool name allowlist, plugin ID (`tribecode`), or `group:plugins`
- Sensitive config fields (`apiKey`) are marked in the schema
- No network access beyond what the TRIBE CLI itself does

### 6.2 Data Privacy

Aligning with TribeCode's privacy-first stance:

- **Local-first**: All telemetry data stored locally in `~/.tribe/`
- **PII scrubbing**: TRIBE CLI automatically redacts secrets before upload
- **Opt-in sync**: Cloud sync only when user explicitly enables
- **No data exfiltration**: Extension never reads OpenClaw conversation data directly
- **Transparent CLI calls**: All operations delegate to the user's own TRIBE binary

### 6.3 Sandbox Compatibility

OpenClaw supports Docker sandboxes for non-main sessions:
- The TRIBE CLI binary lives outside sandbox at `~/.tribe/bin/`
- Sandbox sessions may need individual tribe tools allowlisted
- Config recommends restricting TRIBE tools to main (DM) sessions

### 6.4 OAuth / Authentication Limitations

- `tribe login` requires a browser-based OAuth flow
- In headless/subprocess mode, the browser cannot be opened
- **Mitigation**: Extension checks auth status on activation; if not authenticated, surfaces a user-visible message directing them to run `tribe login` in their terminal
- Once authenticated, tokens are stored at `~/.tribe/` and all subsequent commands work without browser interaction

---

## 7. Platform Compatibility

| Platform | TRIBE CLI | OpenClaw | Integration |
|----------|-----------|----------|-------------|
| macOS (Intel) | Binary available | Full support | Full support |
| macOS (ARM64) | Binary available | Full support | Full support |
| Linux (x86_64) | Binary available | Full support | Full support |
| Windows | Not yet available | WSL2 required | WSL2 only |
| iOS | N/A | Node (limited) | Skill only (no CLI) |
| Android | N/A | Node (limited) | Skill only (no CLI) |

The installer detects platform and downloads the correct binary. On unsupported platforms, the skill gracefully degrades to documentation-only mode.

---

## 8. TUI Command Handling Strategy

Several TRIBE CLI commands use Bubble Tea interactive TUI, which cannot operate in headless subprocess mode. The extension handles these through substitution:

| TUI Command | Subprocess Alternative | Strategy |
|-------------|----------------------|----------|
| `tribe logs` | `tribe logs --format json --tail N` | JSON mode with line limit |
| `tribe realtime` | `tribe status --format json` (polled) | Periodic status polling |
| `tribe circuit dashboard` | `tribe circuit metrics --format json` | Metrics as data |
| `tribe muse monitor` | `tribe muse status --format json` | Status as data |
| `tribe muse attach` | `tribe muse prompt <session> "<msg>"` | Non-interactive prompting |
| `tribe circuit attach` | `tribe circuit status --format json` | Status only |
| `tribe login` | Pre-auth check + manual guidance | See Section 6.4 |

The `cli-runner.ts` abstraction automatically appends `--format json` where appropriate, and the tool descriptions communicate the JSON-mode limitation to the agent.

---

## 9. Testing Plan

### 9.1 Unit Tests

- CLI runner subprocess execution
- JSON output parsing for all command categories
- Config validation and defaults
- Installer detection and download logic
- Timeout category detection (`getTimeout`)
- Factory tool registration patterns

### 9.2 Integration Tests

- SKILL.md loads correctly in OpenClaw Gateway
- Extension `register()` method called with correct `OpenClawPluginApi`
- All 120 tools register without name conflicts
- Tool allowlist gating works correctly
- Background sync service starts/stops cleanly
- `api.logger` and `api.pluginConfig` used correctly

### 9.3 End-to-End Tests

- Fresh install from ClawHub on clean system
- `tribe_search` returns formatted results via OpenClaw chat
- MUSE agent spawning from WhatsApp/Telegram message
- CIRCUIT metrics accessed from WebChat
- Multi-channel: same skill works across all OpenClaw channels
- TUI command fallbacks work correctly (JSON alternatives)
- Auth flow: unauthenticated user gets clear guidance

---

## 10. Rollout Plan

### Milestone 1: Skill MVP
- [ ] Create `SKILL.md` with `metadata.openclaw` frontmatter and all command mappings
- [ ] Test in local OpenClaw instance
- [ ] Submit to ClawHub via `clawhub install` workflow
- [ ] Write installation documentation

### Milestone 2: Extension Beta
- [ ] Implement `@tribecode/openclaw` npm package with object-export entry point
- [ ] Register all 120 tools via factory pattern `registerTool` API
- [ ] CLI installer with platform detection
- [ ] Background sync service
- [ ] TUI fallback handling for all interactive commands
- [ ] Auth status check on activation
- [ ] Publish to npm as beta

### Milestone 3: Production
- [ ] Full test suite passing
- [ ] ClawHub listing live
- [ ] Documentation on tribecode.ai/docs
- [ ] Cross-channel testing (WhatsApp, Telegram, Slack, Discord)
- [ ] Error handling and graceful degradation

### Milestone 4: Advanced Features
- [ ] Webhook integration for real-time TRIBE events
- [ ] OpenClaw session data --> TRIBE knowledge base sync
- [ ] MUSE agent output streaming to OpenClaw channels
- [ ] CIRCUIT issue sync from OpenClaw conversations
- [ ] Analytics dashboard widget for OpenClaw WebChat

---

## 11. Dependencies and Prerequisites

**For Development**:
- Node.js >= 22
- pnpm (OpenClaw monorepo toolchain)
- Go 1.24.5 (TRIBE CLI development)
- OpenClaw dev instance (`pnpm gateway:watch`)

**For Users**:
- OpenClaw installed (`npm install -g openclaw@latest`)
- Internet access for initial TRIBE CLI download
- TribeCode account (free tier sufficient)

**npm Packages**:
- `openclaw/plugin-sdk` (for `OpenClawPluginApi` type)
- No external runtime dependencies (CLI is a standalone binary)

---

## 12. Open Questions

1. **ClawHub Submission Process**: Is there a formal review process or is it open submission? Need to verify at clawhub.ai.
2. **Plugin Signing**: Does OpenClaw verify npm package integrity beyond standard npm checksums?
3. **Windows Support**: Should we target WSL2 explicitly or wait for native TRIBE CLI Windows builds?
4. **Rate Limits**: OpenClaw Gateway tool call limits per session -- need to verify no throttling for CLI-heavy workflows.
5. **Monetization**: Should the extension be free (driving TRIBE signups) or gated behind TRIBE Pro subscription?
6. **MCP Server Integration**: `tribe mcp serve` is a long-running server process. Should this be exposed as a tool, a background service, or omitted?

---

## Appendix A: Reference Links

| Resource | URL |
|----------|-----|
| TribeCode Docs | https://tribecode.ai/docs |
| TribeCode CLI Commands | https://tribecode.ai/docs/cli-commands |
| TribeCode API Reference | https://tribecode.ai/docs/api-reference |
| TribeCode MUSE Docs | https://tribecode.ai/docs/muse |
| TribeCode CIRCUIT Docs | https://tribecode.ai/docs/circuit |
| OpenClaw GitHub | https://github.com/openclaw/openclaw |
| OpenClaw Docs | https://docs.openclaw.ai |
| ClawHub Registry | https://www.clawhub.ai |
| OpenClaw Skills Dir | https://github.com/openclaw/openclaw/tree/main/skills |
| OpenClaw Plugins Source | https://github.com/openclaw/openclaw/tree/main/src/plugins |
| TRIBE CLI npm Package | https://www.npmjs.com/package/@_xtribe/cli |
| TRIBE CLI Source | /Users/almorris/TRIBE/0zen/cli/sdk/ |

## Appendix B: Existing OpenClaw Skills for Reference

52 skills exist in the OpenClaw repo, including patterns relevant to our integration:
- `github` - External service integration via CLI (uses `metadata.openclaw.requires.bins`)
- `coding-agent` - Agent orchestration pattern
- `clawhub` - Registry interaction
- `skill-creator` - Meta-skill for building skills
- `tmux` - Session/process management (similar to MUSE)
- `session-logs` - Log retrieval (similar to TRIBE recall)

These provide architectural patterns and conventions to follow.

## Appendix C: Review Corrections Applied (v1.1.0)

This version incorporates corrections from two independent review agents:

| Issue | Was | Corrected To |
|-------|-----|-------------|
| Type import | `import type { PluginApi } from "openclaw"` | `import type { OpenClawPluginApi } from "openclaw/plugin-sdk"` |
| Entry point export | `export default function activate(api)` | `export default { id, name, register(api) {} }` |
| registerTool signature | `api.registerTool("name", { description, parameters, handler })` | `api.registerTool(factory, { names: [...] })` or single-tool object |
| Logger access | `api.log` | `api.logger` |
| Config access | `api.config.autoSync` | `api.pluginConfig?.autoSync` |
| SKILL.md frontmatter | `tools: - system.run` | `metadata.openclaw.requires.bins` + `metadata.openclaw.install` |
| ClawHub install cmd | `openclaw skill install tribecode` | `clawhub install tribecode` |
| CLI command coverage | ~39 commands (POOR) | 60+ commands / ~120 tools (COMPLETE) |
| Missing categories | No kb, sessions, agents, memory, traces, config, disk | All categories included |
| TUI handling | Not addressed | Full TUI fallback strategy documented |
| OAuth limitations | Not addressed | Documented with mitigation strategy |
| Timeouts | Fixed 30s for all | Category-based: 15s/30s/60s/120s |
