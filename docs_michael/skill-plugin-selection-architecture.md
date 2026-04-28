# OpenClaw Skill and Plugin Selection Architecture

**Author**: Technical Analysis
**Date**: 2026-04-28
**Scope**: How OpenClaw selects skills and plugins when processing user prompts

---

## Quick Reference: Key Limits

**IMPORTANT**: Skills sent to the model are limited by hard constraints:

| Limit                        | Default | Configurable | Description                             |
| ---------------------------- | ------- | ------------ | --------------------------------------- |
| **maxSkillsInPrompt**        | 150     | Yes          | Max number of skills in prompt          |
| **maxSkillsPromptChars**     | 18,000  | Yes          | Max total characters for skills section |
| **maxSkillsLoadedPerSource** | 200     | Yes          | Max skills loaded per directory source  |
| **maxCandidatesPerRoot**     | 300     | Yes          | Max skill directories scanned per root  |
| **maxSkillFileBytes**        | 256,000 | Yes          | Max size of individual SKILL.md file    |

**Critical insight**: If you have 2000 relevant skills, **at most 150** will be sent to the model (and possibly fewer if character budget is exceeded). Skills beyond this limit are **completely invisible** to the model.

---

## Table of Contents

1. [Overview](#overview)
2. [Skills Architecture](#skills-architecture)
3. [Plugins Architecture](#plugins-architecture)
4. [Prompt Processing Pipeline](#prompt-processing-pipeline)
5. [Skill Selection Mechanism](#skill-selection-mechanism)
6. [Plugin Selection Mechanism](#plugin-selection-mechanism)
7. [Tool Assembly & Filtering](#tool-assembly--filtering)
8. [Scoring & Ranking Systems](#scoring--ranking-systems)
9. [System Prompt Construction](#system-prompt-construction)
10. [Implementation Details](#implementation-details)
11. [What Happens to Skills That Don't Fit](#what-happens-to-skills-that-dont-fit)
12. [Summary](#summary)

---

## Overview

OpenClaw uses a **declarative, rule-based selection system** rather than ML-based relevance matching. The architecture has two primary components:

- **Skills**: Declarative instruction sets (SKILL.md files) that the model reads when relevant
- **Plugins**: Extensions that register capabilities (tools, providers, harnesses, channels) into a central registry

**Key Principle**: The system doesn't predict which skills/plugins are relevant. Instead:

- Skills are presented to the model as a catalog; the model decides which to read
- Plugins self-declare their capabilities; selection is policy-driven (allow/deny lists, availability checks)

---

## Skills Architecture

### Definition

Skills are specialized instruction sets stored as markdown files (`SKILL.md`).

**Location**: `src/agents/skills/skill-contract.ts`

```typescript
type Skill = CanonicalSkill & {
  name: string; // Skill identifier
  description: string; // What the skill does
  filePath: string; // Absolute path to SKILL.md
  baseDir: string; // Skill directory
  disableModelInvocation?: boolean; // Don't show to model
};
```

### Skill Metadata (Frontmatter)

Skills contain YAML frontmatter with:

```yaml
always: boolean # Always include in prompt
primaryEnv: string # Primary env var needed (e.g., ANTHROPIC_API_KEY)
requires:
  bins: string[] # Required binaries (e.g., ["git", "npm"])
  env: string[] # Required env vars
os: string[] # Supported OSes (darwin, linux, win32)
exposure:
  includeInRuntimeRegistry: boolean
  includeInAvailableSkillsPrompt: boolean
  userInvocable: boolean
```

### Skill Sources (Precedence Order)

Skills are loaded from multiple directories with **override precedence** (higher number = higher priority):

1. **Extra dirs** (priority: 1) - `config.skills.load.extraDirs`
2. **Bundled** (priority: 2) - Bundled with OpenClaw
3. **Managed** (priority: 3) - `~/.openclaw/skills`
4. **Personal .agents** (priority: 4) - `~/.agents/skills`
5. **Project .agents** (priority: 5) - `<workspace>/.agents/skills`
6. **Workspace** (priority: 6) - `<workspace>/skills`

**Deduplication**: Skills with the same name from lower-priority sources are discarded.

**Implementation**: `src/agents/skills/workspace.ts:loadSkillsFromDirs()`

---

## Plugins Architecture

### Definition

Plugins are external extensions loaded from `extensions/` directory or npm packages.

**Location**: `src/plugins/registry.ts`

### Plugin Registration

Plugins are initialized with an API object that provides registration methods:

```typescript
// Plugin initialization receives:
createApi(record, {
  config: OpenClawConfig,
  pluginConfig: unknown,
  registrationMode: "full" | "discovery" | "setup-only",
});
```

### Plugin Capabilities

Plugins can register multiple capability types:

```typescript
interface PluginApi {
  registerTool(tool: ToolFactory, opts?: RegistrationOpts): void;
  registerProvider(provider: ProviderPlugin): void;
  registerChannel(registration: ChannelPluginRegistration): void;
  registerAgentHarness(harness: AgentHarnessPlugin): void;
  registerHook(events: string[], handler: HookHandler, opts?: HookOpts): void;
  registerCommand(command: CommandRegistration): void;
  registerService(service: ServiceRegistration): void;
}
```

### Plugin Registry Structure

```typescript
class PluginRegistry {
  tools: Map<string, ToolRegistration>; // Tool factories by ID
  providers: Map<string, ProviderRegistration>; // Model providers
  channels: Map<string, ChannelRegistration>; // Channel plugins
  agentHarnesses: Map<string, HarnessRegistration>; // Execution harnesses
  hooks: HookRegistration[]; // Event hooks
  commands: Map<string, CommandRegistration>; // Custom commands
  services: Map<string, ServiceRegistration>; // Background services
}
```

Each registration entry includes:

- `pluginId`: Source plugin identifier
- `source`: Registration source type
- `rootDir`: Plugin root directory

---

## Prompt Processing Pipeline

### Flow Overview

```
User Prompt
  ↓
1. Message Reception (auto-reply system)
  ↓
2. Agent Harness Selection (execution runtime)
  ↓
3. Tool Assembly (filter & build available tools)
  ↓
4. Skill Selection (filter applicable skills)
  ↓
5. System Prompt Construction
  ↓
6. Execution (through selected harness)
```

### Key Entry Points

1. **Auto-reply system**: `src/auto-reply/reply/agent-runner-utils.ts`
2. **Harness selection**: `src/agents/harness/selection.ts:selectAgentHarness()`
3. **Tool assembly**: `src/agents/pi-tools.ts:createOpenClawCodingTools()`
4. **Skill filtering**: `src/agents/skills/workspace.ts:buildWorkspaceSkillsPrompt()`
5. **System prompt**: `src/agents/system-prompt.ts`

---

## Skill Selection Mechanism

### Primary Function

**Location**: `src/agents/skills/workspace.ts`

```typescript
function filterSkillEntries(
  entries: SkillEntry[],
  config?: OpenClawConfig,
  skillFilter?: string[],
  eligibility?: SkillEligibilityContext,
): SkillEntry[];
```

### Selection Criteria (Applied in Order)

#### 1. Config-based Filtering

```typescript
function shouldIncludeSkill(
  entry: SkillEntry,
  config?: OpenClawConfig,
  eligibility?: SkillEligibilityContext,
): boolean;
```

Checks:

- **Allow/Deny lists**: `config.skills.allow` / `config.skills.deny`
- **Bundled allowlist**: Only allowed bundled skills
- **Eligibility context**:
  - `remotePlatform`: Remote OS compatibility check
  - `hasBinary`: Required binary availability
  - `hasEnv`: Required environment variable availability
  - `os`: Current OS compatibility

**Logic**:

1. If in deny list → exclude
2. If allow list exists and skill not in it → exclude
3. If bundled and not in bundled allowlist → exclude
4. If remote platform doesn't match skill's OS requirements → exclude
5. If required binary missing → exclude
6. If required env var missing → exclude
7. Otherwise → include

#### 2. Explicit Skill Filter

If `skillFilter` array is provided:

```typescript
if (skillFilter && !skillFilter.includes(entry.name.toLowerCase())) {
  return false;
}
```

#### 3. Visibility Filtering

```typescript
function isSkillVisibleInAvailableSkillsPrompt(entry: SkillEntry): boolean {
  return entry.exposure.includeInAvailableSkillsPrompt && !entry.disableModelInvocation;
}
```

#### 4. Budget Limits (CRITICAL CONSTRAINT)

Skills are subject to **hard prompt budget limits**:

```typescript
const maxSkillsInPrompt = config?.skills?.maxSkillsInPrompt ?? 150;
const maxSkillsPromptChars = config?.skills?.maxSkillsPromptChars ?? 18_000;
```

**Budget enforcement algorithm** (from `src/agents/skills/workspace.ts:676-717`):

1. **Count truncation**: Take first N skills (alphabetically) where N ≤ `maxSkillsInPrompt` (default: 150)
   - **If you have 2000 skills, skills 151-2000 are immediately dropped**

2. **Character budget check**: Format these N skills in full format
   - If total characters ≤ `maxSkillsPromptChars` (default: 18,000) → **done**

3. **Compact fallback**: If full format exceeds character budget:
   - Switch to **compact format** (name + location only, no descriptions)
   - Saves ~70% space
   - If compact format fits → **done** (all N skills kept, just without descriptions)

4. **Binary search truncation**: If compact format still exceeds budget:
   - Perform binary search to find largest prefix of skills that fits
   - Example: if 150 skills in compact don't fit, might drop to 90, 80, 50... whatever fits
   - **More skills are dropped until budget satisfied**

**Real-world example**:

- You have **2000 relevant skills**
- System takes first **150** (alphabetically)
- 150 skills in full format = 25,000 chars → **exceeds 18,000**
- Switch to compact format: 150 skills = 8,000 chars → **fits!**
- Result: **150 skills sent** (in compact format)

**Worst-case example**:

- You have **2000 relevant skills** with very long names
- System takes first **150** (alphabetically)
- 150 skills in compact format = 22,000 chars → **still exceeds 18,000**
- Binary search finds that only **110 skills** fit in compact
- Result: **110 skills sent** (skills 111-2000 dropped)

### Skill Ordering

After filtering, skills are sorted alphabetically:

```typescript
.sort((a, b) => a.name.localeCompare(b.name, "en"))
```

---

## Plugin Selection Mechanism

### Agent Harness Selection

**Location**: `src/agents/harness/selection.ts`

```typescript
function selectAgentHarness(params: {
  provider: string; // Model provider (e.g., "anthropic")
  modelId?: string; // Model ID (e.g., "claude-sonnet-4.6")
  config?: OpenClawConfig;
  agentId?: string;
  sessionKey?: string;
  agentHarnessId?: string; // Explicit harness override
}): AgentHarness;
```

### Selection Algorithm

1. **Pinned harness** (highest priority):

   ```typescript
   if (agentHarnessId) {
     return registry.agentHarnesses.get(agentHarnessId);
   }
   ```

2. **Runtime policy** (from config):

   ```typescript
   const runtime = config.agents[agentId]?.runtime ?? config.agents.defaults.id ?? "auto";
   ```

3. **Auto mode** (if runtime === "auto"):

   ```typescript
   const candidates = registry.agentHarnesses
     .values()
     .map((harness) => ({
       harness,
       support: harness.supports({
         provider,
         modelId,
         requestedRuntime: runtime,
       }),
     }))
     .filter((c) => c.support.supported)
     .sort((a, b) => (b.support.priority ?? 0) - (a.support.priority ?? 0));

   return candidates[0]?.harness;
   ```

4. **Fallback**: Built-in "pi" harness (if configured)

### Harness Support Declaration

Each harness plugin implements:

```typescript
interface AgentHarnessPlugin {
  supports(ctx: {
    provider: string;
    modelId?: string;
    requestedRuntime: string;
  }): { supported: true; priority?: number } | { supported: false };
}
```

**Priority semantics**: Higher number = higher priority (no upper bound)

---

## Tool Assembly & Filtering

### Overview

**Location**: `src/agents/tools-effective-inventory.ts`

```typescript
function resolveEffectiveToolInventory(
  params: ResolveEffectiveToolInventoryParams,
): EffectiveToolInventoryResult;
```

### Tool Sources

Tools come from three sources:

1. **Core tools**: Built-in OpenClaw tools (Read, Write, Bash, etc.)
2. **Channel tools**: Channel-specific tools (e.g., Telegram attachment upload)
3. **Plugin tools**: Registered via `pluginApi.registerTool()`

### Tool Filtering Pipeline

Tools go through a multi-stage filter:

#### 1. Profile Policy

```typescript
const profile = config?.tools?.profile ?? "full";
```

- `"minimal"`: Only essential tools
- `"full"`: All applicable tools

#### 2. Allow/Deny Lists

```typescript
const globalAllow = config?.tools?.allow;
const globalDeny = config?.tools?.deny;
const agentAllow = config?.agents?.[agentId]?.tools?.allow;
const agentDeny = config?.agents?.[agentId]?.tools?.deny;
```

**Logic**:

- If in deny list → exclude
- If allow list exists and tool not in it → exclude
- Agent-specific lists override global lists

#### 3. Also-Allow

```typescript
const alsoAllow = config?.tools?.alsoAllow ?? [];
```

Explicitly adds tools even if not in allow list.

#### 4. Owner Policy

Some tools are restricted to message sender:

```typescript
if (tool.ownerOnly && !isOwner(messageFrom)) {
  return false;
}
```

#### 5. Model Compatibility

```typescript
if (tool.providers) {
  if (!tool.providers.includes(currentProvider)) {
    return false;
  }
}
```

#### 6. Channel-Specific Filtering

Channels can filter tools based on message context:

```typescript
if (channel.filterTools) {
  tools = channel.filterTools(tools, messageContext);
}
```

### Tool Ordering

Tools are grouped by source:

1. **Core tools** (in registration order)
2. **Channel tools** (in registration order)
3. **Plugin tools** (in registration order)

No explicit scoring/ranking within groups.

---

## Scoring & Ranking Systems

### Agent Harnesses: Priority-Based

```typescript
{ supported: true, priority: 100 }
```

- Higher priority = preferred
- No upper bound
- Sort: `(b.priority ?? 0) - (a.priority ?? 0)`

### Skills: Alphabetical

```typescript
.sort((a, b) => a.name.localeCompare(b.name, "en"))
```

No priority or relevance scoring.

### Tools: Registration Order

No explicit scoring. Order determined by:

1. Source type (core → channel → plugin)
2. Registration sequence within source

### No ML-Based Relevance

The system does **not** use:

- Semantic similarity matching
- Embedding-based relevance
- Usage frequency scoring
- Learned ranking models

All selection is **declarative and rule-based**.

---

## System Prompt Construction

### Skills in System Prompt

**Location**: `src/agents/skills/skill-contract.ts`

```typescript
function formatSkillsForPrompt(skills: Skill[]): string;
```

**Full format** (default):

```xml
<available_skills>
  <skill>
    <name>commit</name>
    <description>Create git commits following repository conventions</description>
    <location>~/.agents/skills/commit/SKILL.md</location>
  </skill>
  <!-- ... more skills ... -->
</available_skills>
```

**Compact format** (when over budget):

```xml
<available_skills>
  <skill>
    <name>commit</name>
    <location>~/.agents/skills/commit/SKILL.md</location>
  </skill>
  <!-- ... more skills ... -->
</available_skills>
```

### Model Instructions

The agent system prompt includes:

> "The following skills provide specialized instructions for specific tasks. Use the read tool to load a skill's file when the task matches its description."

**Key implication**: The **model decides** when to invoke a skill by:

1. Seeing skill name + description in prompt
2. Determining if task matches skill description
3. Using `Read` tool to load `SKILL.md` file
4. Following instructions in skill file

### Tools in System Prompt

Tools are presented as function definitions following the model provider's schema (Anthropic, OpenAI, etc.).

**No explicit relevance hint** - all allowed tools are presented equally.

---

## Implementation Details

### Critical Files

| Component         | File Path                                    |
| ----------------- | -------------------------------------------- |
| Skill loading     | `src/agents/skills/workspace.ts`             |
| Skill contract    | `src/agents/skills/skill-contract.ts`        |
| Plugin registry   | `src/plugins/registry.ts`                    |
| Plugin loader     | `src/plugins/loader.ts`                      |
| Harness selection | `src/agents/harness/selection.ts`            |
| Tool inventory    | `src/agents/tools-effective-inventory.ts`    |
| Tool creation     | `src/agents/pi-tools.ts`                     |
| System prompt     | `src/agents/system-prompt.ts`                |
| Auto-reply runner | `src/auto-reply/reply/agent-runner-utils.ts` |

### Configuration Schema

**Skills config**:

```typescript
{
  skills: {
    allow?: string[];          // Skill allowlist
    deny?: string[];           // Skill denylist
    maxSkillsInPrompt?: number;      // Max skills (default: 150)
    maxSkillsPromptChars?: number;   // Max chars (default: 18,000)
    load?: {
      extraDirs?: string[];    // Additional skill directories
    }
  }
}
```

**Tools config**:

```typescript
{
  tools: {
    profile?: "minimal" | "full";  // Tool profile
    allow?: string[];              // Tool allowlist
    deny?: string[];               // Tool denylist
    alsoAllow?: string[];          // Always include these
  },
  agents: {
    [agentId]: {
      tools?: {
        allow?: string[];      // Agent-specific allowlist
        deny?: string[];       // Agent-specific denylist
      }
    }
  }
}
```

**Agent harness config**:

```typescript
{
  agents: {
    defaults: {
      id?: string;            // Default harness ID ("auto" or specific)
    },
    [agentId]: {
      runtime?: string;       // Agent-specific harness
    }
  }
}
```

### Eligibility Context

Skills are filtered based on runtime environment:

```typescript
interface SkillEligibilityContext {
  remotePlatform?: NodeJS.Platform; // Remote OS
  hasBinary?: (bin: string) => boolean; // Binary availability
  hasEnv?: (env: string) => boolean; // Env var availability
}
```

Used for:

- Cross-platform skill filtering (macOS skills on Linux remote)
- Required binary detection (git, docker, etc.)
- Environment variable checks (API keys, credentials)

### Plugin Discovery

Plugins are discovered from:

1. **Core extensions**: `extensions/*/manifest.json`
2. **NPM packages**: `node_modules/@openclaw-plugin-*/manifest.json`
3. **Local paths**: From config `plugins.load.extraDirs`

**Manifest structure**:

```json
{
  "id": "plugin-id",
  "name": "Plugin Name",
  "version": "1.0.0",
  "main": "./dist/index.js",
  "capabilities": ["tool", "provider", "channel"]
}
```

---

## What Happens to Skills That Don't Fit?

**They are completely invisible to the model.**

- Skills beyond the count limit are **never sent** in the prompt
- The model has **no awareness** these skills exist
- The model **cannot invoke** them (can't read what it doesn't know about)

**Critical implication**: If you have 500 relevant skills and only 150 fit, the model will only know about 150. The other 350 are completely unavailable for that conversation.

**Mitigation strategies**:

1. **Explicit skill filter**: `config.agents[agentId].skillFilter = ["skill1", "skill2"]` - only include specific skills
2. **Raise count limit**: `config.skills.limits.maxSkillsInPrompt = 300` - allow more skills
3. **Raise character budget**: `config.skills.limits.maxSkillsPromptChars = 40_000` - allow longer prompts
4. **Use skill allow list**: `config.skills.allow = ["skill-a", "skill-b", ...]` - pre-filter to relevant skills
5. **Keep skill names/descriptions short** - helps with character budget
6. **Use compact format**: Happens automatically when over budget (drops descriptions)

---

## Summary

### How Skills Are Selected

1. **Loaded** from multiple directories (workspace → project .agents → personal .agents → managed → bundled → extra)
2. **Filtered** by:
   - Allow/deny lists in config
   - OS/binary/env requirements
   - Visibility settings
3. **Sorted** alphabetically by name
4. **Truncated** by budget constraints:
   - **Hard count limit**: Max 150 skills (default)
   - **Hard character limit**: Max 18,000 chars (default)
   - Binary search to find largest subset that fits
   - Skills beyond limit are **completely invisible** to model
5. **Presented** to model as catalog in system prompt (only skills that fit!)
6. **Invoked** by model decision (model reads SKILL.md when task matches)

### How Plugins Are Selected

1. **Registered** during plugin initialization
2. **Harnesses selected** via:
   - Explicit override (`agentHarnessId`)
   - Runtime policy (config)
   - Auto mode (priority-based matching)
3. **Tools filtered** via:
   - Profile policy (minimal/full)
   - Allow/deny lists
   - Model compatibility
   - Owner restrictions
   - Channel-specific filtering
4. **No ranking** - all allowed tools presented equally

### Key Architectural Principles

- **Declarative over learned**: No ML-based relevance matching
- **Policy-driven**: Selection controlled by explicit configuration
- **Self-declaring**: Plugins/harnesses declare their capabilities
- **Model-driven skill usage**: Model decides which skills to read
- **Rule-based filtering**: All selection uses deterministic rules

### Performance Considerations

- **Budget limits** prevent prompt bloat (skill count + character limits)
- **Binary search** for optimal skill subset when over budget
- **Compact format** fallback reduces skill prompt size by ~70%
- **Lazy loading** - skills only read when model invokes them
- **Tool filtering** before construction reduces overhead

---

## Developer Notes

### Adding a New Skill

1. Create `skills/<skill-name>/SKILL.md`
2. Add YAML frontmatter with metadata
3. Skills auto-discovered on next load
4. Use `config.skills.allow` to control visibility

### Adding a New Plugin Tool

1. In plugin initialization:
   ```typescript
   api.registerTool({
     name: "my_tool",
     description: "What the tool does",
     parameters: zodSchema,
     execute: async (params) => {
       /* ... */
     },
   });
   ```
2. Tool auto-available in next agent run
3. Use `config.tools.deny` to exclude if needed

### Adding a New Agent Harness

1. Implement `AgentHarnessPlugin` interface
2. Register with `api.registerAgentHarness(harness)`
3. Implement `supports()` with priority logic
4. Harness auto-selected in "auto" mode based on priority

### Debugging Selection

**Skills**:

- Set `config.skills.allow` to narrow to specific skills
- Check skill frontmatter for `disableModelInvocation`
- Verify OS/binary/env requirements match

**Tools**:

- Set `config.tools.profile = "minimal"` to reduce noise
- Use `config.tools.deny` to exclude specific tools
- Check agent-specific tool config overrides

**Harnesses**:

- Set explicit `agentHarnessId` to force specific harness
- Check `config.agents.defaults.id` for default
- Verify provider/model in harness `supports()` logic

---

## Related Research

For a comprehensive research proposal on improving skill selection using classical machine learning, see:

**[Research: Learning to Route - Efficient Skill Selection for Autonomous Agents using Classical Machine Learning](./research-skill-routing-ml.md)**

This research proposal addresses the current limitations of alphabetical skill ranking and proposes a Learning-to-Rank approach using gradient boosted trees (LightGBM/XGBoost) trained on trajectory logs. Key highlights:

- **Problem**: Current alphabetical ranking wastes budget on irrelevant skills and drops relevant ones
- **Approach**: ML-based ranking using features from skill descriptions, historical usage, and task context
- **Data Source**: Trajectory logs (`~/.openclaw/trajectories/*.jsonl`) with skill invocation tracking
- **Expected Impact**: +2-5% task success rate improvement, -10-20% reduction in skills invoked
- **Implementation**: Production-ready integration with <1ms inference latency

---

**End of Document**
