# Research Topics: OpenClaw as a Study Subject

> Suggested research directions derived from actual implementation patterns in OpenClaw.
> Each topic is grounded in specific code, not abstraction. Suitable candidates for
> arXiv (cs.AI, cs.SE, cs.NI, cs.CR depending on topic).

---

## Preamble: Why OpenClaw is Interesting for Research

OpenClaw is a **production open-source personal AI assistant gateway** that:
- Routes messages across 25+ heterogeneous messaging platforms to a unified AI agent
- Manages long-running conversation sessions with context window limits
- Handles multi-provider AI failover with sub-second reliability
- Enforces strict plugin isolation in a TypeScript monorepo with 100+ plugins
- Supports hierarchical sub-agent spawning with registry-backed lifecycle

Unlike research prototypes, it has to work in real deployments under real constraints:
character limits on WhatsApp, rate limits from Anthropic, memory bounds on Raspberry Pi,
and untrusted inbound DMs from strangers. This makes it a rich implementation to study.

---

## Topic 1: Structured System Prompt Partitioning for KV-Cache Efficiency

**Field**: cs.AI, cs.LG (LLM Systems)

### What's in the Codebase

OpenClaw inserts an explicit HTML comment — `<!-- OPENCLAW_CACHE_BOUNDARY -->` — to split
the system prompt into a **stable prefix** and a **dynamic suffix**:

```typescript
// src/agents/system-prompt-cache-boundary.ts
export const SYSTEM_PROMPT_CACHE_BOUNDARY = "\n<!-- OPENCLAW_CACHE_BOUNDARY -->\n";

export function splitSystemPromptCacheBoundary(text) {
  const idx = text.indexOf(SYSTEM_PROMPT_CACHE_BOUNDARY);
  return {
    stablePrefix: text.slice(0, idx).trimEnd(),
    dynamicSuffix: text.slice(idx + BOUNDARY_LEN).trimStart(),
  };
}
```

Per-run dynamic additions (skills, per-agent context, injected context) go **after** the boundary;
the static base prompt, capability description, and stable workspace rules go **before** it.
The invariant: the stable prefix bytes are identical across turns, so the LLM provider's KV
cache always hits.

The system also enforces deterministic ordering in registries, plugin lists, file results, and
map iteration specifically to preserve prompt bytes across turns (`AGENTS.md`: "Preserve old
transcript bytes when possible").

### Research Questions

1. How much does explicit stable-prefix/dynamic-suffix partitioning improve KV-cache hit rate
   versus naive system prompt construction, at different request rates and providers?
2. What is the optimal boundary placement heuristic? (Greedy maximize stable prefix?
   Information-theoretic stability scoring? Provider-specific token alignment?)
3. What is the tradeoff between prompt expressiveness and cache stability as agent context grows?
4. Can cache-stability scoring be automated as a linter/CI check (detecting when a commit
   moves stable content below the boundary)?

### Why Interesting for arXiv

Prompt caching is widely used (Anthropic, Google, OpenAI all support it) but the **engineering
discipline of maintaining stable prefix bytes across a long-lived assistant** is underexplored.
Most discussion is at the API level; OpenClaw provides a concrete production architecture to
study and benchmark.

---

## Topic 2: Adaptive Context Window Compaction with LLM-Generated Summaries

**Field**: cs.AI, cs.CL

### What's in the Codebase

When a session transcript approaches the context window limit, OpenClaw "compacts" it by
replacing older conversation turns with an LLM-generated summary (`src/agents/compaction.ts`):

```typescript
export const BASE_CHUNK_RATIO = 0.4;
export const MIN_CHUNK_RATIO = 0.15;
export const SAFETY_MARGIN = 1.2; // 20% buffer for estimateTokens() inaccuracy

const MERGE_SUMMARIES_INSTRUCTIONS = [
  "Merge these partial summaries into a single cohesive summary.",
  "MUST PRESERVE:",
  "- Active tasks and their current status (in-progress, blocked, pending)",
  "- Batch operation progress (e.g., '5/17 items completed')",
  "- The last thing the user requested and what was being done about it",
  ...
];

const IDENTIFIER_PRESERVATION_INSTRUCTIONS =
  "Preserve all opaque identifiers exactly as written (no shortening or reconstruction), " +
  "including UUIDs, hashes, IDs, hostnames, IPs, ports, URLs, and file names.";
```

The system supports:
- Chunked summarization (too large → split into chunks, summarize each, merge summaries)
- Identifier preservation policy (strict / custom / off) — preserving UUIDs, IPs in summaries
- Retry with fallback to "No prior history." on error
- Pre/post hooks (`before_compaction`, `after_compaction`) for plugins to observe or annotate
- Transcript repair (tool-use result pairing, orphaned tool call cleanup)

### Research Questions

1. How do different compaction triggers (token ratio vs. message count vs. time) affect
   downstream task performance on multi-turn agentic benchmarks?
2. What is the impact of **identifier preservation** in LLM-generated summaries on correctness
   of downstream actions? (E.g., does the model produce wrong git commit hashes or IPs when
   summaries reconstruct them?)
3. Adaptive compression ratios: can a model-in-the-loop evaluate its own summary quality and
   request more context before compacting?
4. Compaction strategies compared empirically: summary-based vs. sliding window vs. RAG-based
   episodic recall vs. hybrid — on a personal-assistant task suite.

### Why Interesting for arXiv

Context window management is a known problem but the **production constraints of a single-user
personal assistant** (the model may need to recall IDs from 2 hours ago; "No prior history"
is unacceptable) motivate different tradeoffs than what academic benchmarks typically measure.
The identifier preservation issue is a concrete failure mode not widely studied.

---

## Topic 3: Multi-Provider AI Failover at Single-User Scale: Cooldown Probing

**Field**: cs.DC, cs.AI (Systems Reliability)

### What's in the Codebase

OpenClaw has a multi-layer failover system across AI providers. When a provider call fails,
the system classifies the failure and applies different cooldown behaviors:

```typescript
// src/agents/failover-policy.ts
export function shouldAllowCooldownProbeForReason(reason) {
  return reason === "rate_limit" || reason === "overloaded" ||
    reason === "billing" || reason === "unknown" || reason === "timeout";
}

export function shouldPreserveTransientCooldownProbeSlot(reason) {
  return reason === "model_not_found" || reason === "format" ||
    reason === "auth" || reason === "auth_permanent" || reason === "session_expired";
}
```

Beyond failure classification, the system supports:
- Auth profile rotation (multiple API keys per provider, round-robin with last-used ordering)
- Cooldown auto-expiry tracking per profile
- Probe slots (transient vs. permanent cooldowns)
- Cross-provider fallback (OpenAI → Anthropic → Gemini)
- Auth health monitoring (`src/agents/auth-health.ts`)
- Doctor paths that detect and repair stale/failed auth configurations

### Research Questions

1. What failure mode taxonomy is appropriate for LLM API providers? (The codebase identifies:
   rate_limit, overload, billing, auth, auth_permanent, session_expired, format, timeout,
   model_not_found, unknown — is this sufficient/correct?)
2. How do different probe/cooldown strategies affect p95 latency and availability for a
   personal AI assistant under realistic workloads?
3. Is round-robin (ordered by last-used) the right key rotation algorithm, or do
   cost-aware/latency-aware strategies dominate?
4. How should single-user systems handle the "billing exhausted" failure mode differently
   from "rate limited" failures?

### Why Interesting for arXiv

Cloud reliability literature (circuit breakers, retry budgets, probe algorithms) is well-developed
for services, but **applying these patterns to LLM API orchestration at personal/single-user
scale** involves different constraints: no horizontal scaling, human-in-the-loop recovery,
and provider-specific semantic failures. This is an underexplored operational angle.

---

## Topic 4: Heterogeneous Message Channel Abstraction for Personal AI Gateways

**Field**: cs.NI, cs.SE, cs.AI

### What's in the Codebase

OpenClaw bridges 25+ messaging platforms (WhatsApp, Telegram, Discord, Slack, Signal,
iMessage, Matrix, IRC, LINE, WeChat, QQ, Nostr, Twitch, ...) through a unified plugin-based
channel abstraction (`src/channels/`, `extensions/*/`).

Each channel has:
- Different auth models (OAuth, QR scan, phone+code, bot token, webhook)
- Different message constraints (Discord: 2000 chars, WhatsApp: 65536, IRC: 512 bytes)
- Different threading/group semantics
- Different delivery guarantees (no delivery receipt vs. read receipts vs. none)
- Different media type support

The session routing system assigns a deterministic session key per (agent, channel, account,
peer, chat-type) tuple:

```typescript
// src/routing/session-key.ts
export function buildAgentPeerSessionKey(params: {
  agentId, channel, accountId, peerKind, peerId, identityLinks,
  dmScope: "main" | "per-peer" | "per-channel-peer" | "per-account-channel-peer"
}): string
```

Cross-channel identity linking (`identityLinks`) allows the same user's Discord DM and
WhatsApp DM to share a session.

### Research Questions

1. What is the minimal interface contract that captures the behavioral diversity of 25+
   messaging platforms without over-abstracting? (Plugin SDK surface in OpenClaw:
   `src/plugin-sdk/channel-contract.ts`)
2. What privacy and security invariants must hold across all session scope configurations?
   (Failure mode: per-peer DM isolation misconfigured → Bob sees Alice's context)
3. Can cross-channel identity linking be done without exposing cross-platform identity to
   third parties? What are the cryptographic and UX constraints?
4. How should a personal AI handle channel-specific rate limits and throttling at the
   delivery layer vs. at the AI inference layer?

### Why Interesting for arXiv

Multi-channel personal AI is distinct from multi-tenant or enterprise chatbot design.
The challenge of **preserving session context identity across heterogeneous protocols while
maintaining strict isolation guarantees** is a systems engineering problem with interesting
formal properties. OpenClaw provides a concrete worked example with 25 adapters.

---

## Topic 5: Streaming Text Delivery Across Constrained Messaging Platforms

**Field**: cs.NI, cs.HC (HCI / Messaging UX)

### What's in the Codebase

OpenClaw implements a sophisticated streaming chunking system (`EmbeddedBlockChunker`,
`src/agents/pi-embedded-block-chunker.ts`):

- **Low bound**: don't emit until buffer >= `minChars` (avoid single-word spam)
- **High bound**: prefer splits before `maxChars`; forced split at hard limit
- **Break preference**: `paragraph` → `newline` → `sentence` → `whitespace` → hard break
- **Markdown fence awareness**: never split inside code fences; when forced, close + reopen fence
- **Coalescing**: merge consecutive chunks during idle gaps (`idleMs`) before sending
- Per-channel constraints: Discord's `maxLinesPerMessage` (17), WhatsApp's `textChunkLimit`,
  platform-specific `chunkMode` (`length` or `newline`)

The docs explicitly state: "There is no true token-delta streaming to channel messages today.
Preview streaming is message-based (send + edits/appends)." — so Telegram/Discord/Slack
receive **edited messages** as a streaming simulation.

### Research Questions

1. What is the perceived quality difference between true token-delta streaming (like web UIs)
   vs. edit-based streaming vs. block-delivery streaming on messaging platforms?
2. What chunking strategies minimize perceived latency while respecting platform constraints?
   (Empirical study of `minChars`/`maxChars`/`idleMs` tradeoffs)
3. How do Markdown-aware chunking algorithms compare to naive length-based splits in
   practice? (Broken code fences, orphaned bullet points, etc.)
4. What are the delivery semantics that a messaging abstraction layer must preserve to make
   LLM output readable across 25+ platforms with different threading models?

### Why Interesting for arXiv

Streaming delivery to messaging platforms is a real engineering challenge with measurable
UX impact. The Markdown-fence-aware chunking algorithm is a specific novel contribution
that could be evaluated empirically against naive approaches.

---

## Topic 6: Hybrid Episodic Memory for Personal AI: BM25 + Vector Search with Temporal Decay

**Field**: cs.AI, cs.IR

### What's in the Codebase

The memory search system (`src/agents/memory-search.ts`, docs `memory-search.md`) runs
two retrieval paths **in parallel** and merges results:

```
Query
  ├─ Embedding → Vector Search (semantic: "gateway host" ≈ "machine running OpenClaw")
  └─ Tokenize  → BM25 Search  (lexical: exact IDs, error strings, config keys)
  → Weighted Merge → MMR Diversity Filter → Top-K
```

Additional features:
- **Temporal decay**: 30-day half-life on older notes; evergreen files (MEMORY.md) exempt
- **MMR (Maximal Marginal Relevance)**: prevents redundant results from daily notes
- **Multimodal** (with Gemini Embedding 2): index images and audio, search with text queries
- **Local embeddings**: GGUF-based local model as fallback (no API key needed)
- **Session transcript indexing**: opt-in recall of prior conversation history
- **Degraded mode**: lexical ranking over FTS when no embedding provider is available

### Research Questions

1. How does the hybrid BM25+vector approach compare to pure semantic retrieval for the
   specific recall patterns of a personal AI assistant? (Key hypothesis: personal AI memory
   has high rate of exact-identifier recall needs — UUIDs, error messages, config keys —
   that semantic-only approaches miss)
2. What temporal decay function is optimal for personal AI memory? (The current half-life=30d
   is a guess — what does empirical data say about how quickly personal context goes stale?)
3. How does MMR diversity interact with temporal decay in multi-day note archives?
4. What are the recall/precision characteristics of session transcript indexing vs. structured
   note-taking vs. no memory augmentation on realistic personal-assistant tasks?

### Why Interesting for arXiv

Personal AI memory is distinct from enterprise RAG: the corpus is small (one person's notes),
extremely heterogeneous, and has high-stakes exact-identifier recall requirements (getting
someone's IP wrong causes a real operational failure). The temporal decay + hybrid retrieval
architecture is a coherent system design worth formalizing and evaluating.

---

## Topic 7: Hierarchical Sub-Agent Orchestration with Registry-Backed Lifecycle Management

**Field**: cs.AI, cs.DC (Multi-Agent Systems)

### What's in the Codebase

OpenClaw implements a sub-agent spawning system (`src/agents/subagent-registry.ts` and ~30
related files) where the main agent can spawn child sessions:

```typescript
// Depth limiting
SUBAGENT_ENDED_REASON_COMPLETE / _ERROR / _KILLED

// Registry operations
countActiveDescendantRunsFromRuns()
countPendingDescendantRunsExcludingRunFromRuns()
isSubagentSessionRunActiveFromRuns()
reconcileOrphanedRun()           // handles process crash recovery
reconcileOrphanedRestoredRuns()  // handles gateway restart mid-run
```

Key properties:
- **Depth limits**: explicit limits on sub-agent nesting depth
- **Orphan recovery**: on gateway restart, detect and reconcile runs that were in-flight
- **Persistence**: registry survives process restart (`subagent-registry-state.ts`)
- **Announcement queue**: sub-agent completion notifications with retry + timeout
- **Scope inheritance**: sub-agents inherit sandbox policies, tool policies, workspace root
- **ACP child sessions**: ACP-spawned sub-agents inherit envelope constraints

### Research Questions

1. What failure modes are unique to hierarchical AI agent orchestration that do not appear
   in flat multi-agent or single-agent systems? (Orphan detection, depth explosion, announce
   queue saturation, cascading timeouts)
2. How should a personal AI gateway handle the "sub-agent completed while parent was
   compacting" race condition? What are the correct semantics?
3. What depth limits are empirically appropriate for multi-turn sub-agent tasks?
   (Tradeoff: too shallow → tasks can't be decomposed; too deep → resource exhaustion)
4. Is registry-backed persistence (write-ahead-log style) the right architecture for
   sub-agent state, or should it be event-sourced, in-memory, or something else?

### Why Interesting for arXiv

Sub-agent orchestration in production is getting real (Codex, Claude Code, etc.) but
the **reliability engineering layer** — orphan detection, crash recovery, announce queues,
depth limits — is rarely documented. OpenClaw provides a complete production implementation
to study and generalize.

---

## Topic 8: Device Pairing as a Trust Mechanism for Ambient Personal AI Gateways

**Field**: cs.CR (Security, Trust)

### What's in the Codebase

OpenClaw uses a **challenge-based device pairing protocol** (`src/pairing/`) for all WebSocket
clients (operators, companion apps, nodes):

- Every connect must sign a server-issued nonce challenge
- Signature payload `v3` binds: device ID + platform + deviceFamily (prevents impersonation
  by a different device class)
- Gateway issues **device tokens** for subsequent connects after first approval
- Repair pairing required if platform/deviceFamily changes (pins metadata at pair time)
- Network zone-sensitive trust:
  - Local loopback → auto-approve (same-host UX)
  - LAN / Tailscale → explicit approval required
  - Public internet → explicit approval required

The DM pairing system (separate from device pairing) handles unknown inbound senders:

```
Unknown WhatsApp sender → bot issues pairing code
User tells operator the code out-of-band
Operator: `openclaw pairing approve telegram <code>`
→ sender added to local allowlist
```

### Research Questions

1. What is the correct threat model for a **personal AI gateway** vs. a multi-tenant service?
   (Key difference: the operator and the primary user are the same person, so the gateway's
   threat model is mostly inbound attacker via channels, not malicious operator)
2. How does network-zone-sensitive auto-approval (loopback OK, LAN explicit) compare to
   alternatives (always require approval, PIN-based, biometric) on the UX/security Pareto
   frontier?
3. What is the formal security model for the `v3` challenge signature that binds platform +
   deviceFamily? What attacks does it prevent, and what attacks remain?
4. The DM pairing ("stranger sends a code") is an invitation system. What social engineering
   attacks does it enable, and how does rate-limiting the pairing flow defend against them?

### Why Interesting for arXiv

Ambient personal AI assistants that listen on real messaging channels have a fundamentally
different trust model from cloud services: they are **single-tenant but multi-ingress**,
with inbound attacks coming through third-party platforms (WhatsApp, Telegram) rather than
through the gateway itself. This threat model is under-formalized.

---

## Topic 9: Protocol Bridge Architecture for IDE-Integrated Personal AI

**Field**: cs.SE, cs.AI (AI Systems Integration)

### What's in the Codebase

The `openclaw acp` command (`src/acp/translator.ts`) bridges three protocol layers:

```
IDE / Editor
  ↕ ACP (Agent Client Protocol — stdio, @agentclientprotocol/sdk)
ACP Translator
  ↕ OpenClaw Gateway WebSocket (custom RPC — src/gateway/protocol/)
Gateway
  ↕ MCP (Model Context Protocol — via mcporter)
External MCP Servers
```

The translator handles:
- Session ID mapping (ACP session UUID ↔ Gateway session key like `agent:main:main`)
- Event mapping (Gateway agent stream events → ACP tool_call / output_text / usage_update)
- Stop reason translation (Gateway stop codes → ACP StopReason enum)
- Rate limiting at the ACP layer (DoS protection: max 2MB prompt, fixed-window rate limiter)
- Best-effort session replay (`loadSession` — user/assistant text, not tool history)

From `src/acp/translator.ts`:
```typescript
const MAX_PROMPT_BYTES = 2 * 1024 * 1024; // CWE-400 / GHSA-cxpw-2g23-2vgw
const ACP_LOAD_SESSION_REPLAY_LIMIT = 1_000_000;
```

### Research Questions

1. What is the minimal faithful translation between ACP's session model and a
   Gateway-backed WebSocket session model? Where do the semantics genuinely diverge?
2. How should a bridge handle **partial protocol coverage**? (OpenClaw explicitly documents
   which ACP features are unsupported, partial, or implemented — this is rare transparency)
3. What are the failure modes of **event stream impedance mismatch** between protocols?
   (Gateway streams tool events; ACP expects tool_call with structured locations)
4. Is stdio-based ACP the right transport for IDE integration, or does the stateless
   request-response model of stdio clash with the stateful streaming model of LLM inference?

### Why Interesting for arXiv

Protocol proliferation in AI tooling (ACP, MCP, OpenAI Responses API, custom WebSocket
formats) creates a real interoperability problem. OpenClaw's translator is a concrete
production bridge with documented limitations and workarounds — a case study in AI
protocol impedance.

---

## Topic 10: Architectural Boundary Enforcement in Large TypeScript AI Systems

**Field**: cs.SE (Software Engineering)

### What's in the Codebase

OpenClaw has ~100 bundled plugins in `extensions/`, a core in `src/`, and a public SDK
in `src/plugin-sdk/`. The boundary rules are enforced through multiple layers:

1. **TypeScript path mapping** (`extensions/tsconfig.package-boundary.paths.json`) — redirects
   core imports to SDK-only paths when compiling extension code
2. **CI architecture gate** (`pnpm check:architecture` / `check-additional` in CI) — runs
   `madge` cycle detection and a custom architecture check
3. **Module boundary linting** — oxlint rules on import patterns
4. **Pre-commit hooks** — `pnpm check:changed --staged` validates staged changes
5. **Human-readable AGENTS.md rules** — explicitly prohibit certain import patterns

The boundary is: extensions import from `openclaw/plugin-sdk/*` and their own local barrels
(`api.ts`, `runtime-api.ts`). Core must not deep-import extension internals.

Violations in the real world tend to be "plugin needs a core helper that isn't in the SDK
yet" — OpenClaw's answer is to expand the SDK rather than breach the boundary.

### Research Questions

1. How effective are different boundary enforcement mechanisms (type-level, linter, CI gate,
   human rules) individually and in combination? What enforcement layer catches what class
   of violation?
2. What is the maintenance cost of strict plugin boundaries in a monorepo with 100+ plugins
   over time? (Does the SDK surface grow to absorb boundary pressure? Does it ever shrink?)
3. Can boundary violations be automatically detected and fixed using LLM-assisted refactoring?
   (The fix is usually "extract to SDK + update both sides" — a well-defined mechanical task)
4. What is the relationship between plugin boundary strictness and the ease of third-party
   plugin development? (Tighter boundaries → more stable third-party surface, but also more
   friction for bundled plugins)

### Why Interesting for arXiv

Plugin architecture boundary enforcement is discussed in software engineering literature
(OSGi, Eclipse, microkernel design) but rarely studied empirically in modern TypeScript AI
systems where boundaries must coexist with fast iteration. OpenClaw provides a measurable
real-world subject with explicit boundary rules, CI enforcement, and a growing plugin corpus.

---

## Meta: Which Topics Have the Best Arxiv Potential?

| Topic | Novelty | Empirical Tractability | Relevance to Broader Field |
|-------|---------|----------------------|---------------------------|
| Prompt prefix caching (T1) | High | High (A/B on real system) | Very high — affects all LLM apps |
| Context compaction (T2) | Medium-High | Medium | High — context length is unsolved |
| Provider failover (T3) | Medium | High (synthetic workloads) | Medium |
| Multi-channel abstraction (T4) | Medium | Low (breadth study) | Medium |
| Streaming delivery (T5) | Low-Medium | High (user study) | Medium |
| Hybrid memory (T6) | Medium | High (benchmark suite) | High — RAG for personal AI |
| Sub-agent orchestration (T7) | High | Medium (fault injection) | Very high — agent reliability |
| Device pairing trust model (T8) | High | Low (formal analysis) | Medium |
| Protocol bridge (T9) | Medium | Low (case study) | Medium |
| Boundary enforcement (T10) | Medium | Medium (empirical) | Medium |

### Recommended Starting Points

**Quickest to paper**: T1 (prompt caching) or T5 (streaming delivery) — concrete engineering
decisions with measurable outcomes, grounded in a real system.

**Highest impact if well-executed**: T2 (compaction) or T6 (hybrid memory) — address unsolved
problems in LLM systems, and OpenClaw provides a production baseline to compare against.

**Most novel conceptually**: T7 (sub-agent orchestration reliability) or T8 (personal AI trust
model) — areas where almost no literature exists and OpenClaw provides a concrete architecture
to study.

---

## How to Engage with the Codebase for Each Topic

| Topic | Primary Entry Point | Key Files |
|-------|-------------------|-----------|
| T1: Prompt caching | `src/agents/` | `system-prompt-cache-boundary.ts`, `prompt-cache-stability.ts`, `cache-trace.ts` |
| T2: Compaction | `src/agents/` | `compaction.ts`, `pi-embedded-subscribe.handlers.compaction.ts` |
| T3: Failover | `src/agents/` | `failover-policy.ts`, `auth-profiles.ts`, `auth-health.ts`, `model-fallback.ts` |
| T4: Channels | `extensions/`, `src/channels/` | Channel plugin manifests, `src/routing/` |
| T5: Streaming | `src/agents/` | `pi-embedded-block-chunker.ts`, `pi-embedded-subscribe.ts`, docs/concepts/streaming.md |
| T6: Memory | `src/agents/memory-search.ts` | `docs/concepts/memory-search.md`, `extensions/active-memory/` |
| T7: Sub-agents | `src/agents/` | `subagent-registry.ts`, `subagent-registry-lifecycle.ts`, `subagent-orphan-recovery.ts` |
| T8: Trust/Pairing | `src/pairing/` | `pairing-store.ts`, `pairing-challenge.ts`, `docs/gateway/pairing.md` |
| T9: ACP Bridge | `src/acp/` | `translator.ts`, `event-mapper.ts`, `session-mapper.ts` |
| T10: Boundaries | `extensions/`, `src/plugin-sdk/` | `tsconfig.package-boundary.paths.json`, `scripts/check-architecture.ts` |

---

*Document prepared: 2026-04-21. Based on code exploration of OpenClaw v2026.4.20.*
*All file references are relative to the repo root.*
