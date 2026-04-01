# Claude Code (Leaked Source Archive)

English | [中文](./README_ZH.md) | [繁體中文](./README_ZH_TW.md) | [조선어](./README_KO.md) | [日本語](./README_JA.md) | [Español](./README_ES.md) | [Русский](./README_RU.md)

<p align="center">
  <img src="https://img.shields.io/badge/status-archived%20%2B%20rebuilding-blue" />
  <img src="https://img.shields.io/badge/runtime-Bun%20%2F%20Node-black" />
  <img src="https://img.shields.io/badge/language-TypeScript%20→%20Rust-orange" />
  <img src="https://img.shields.io/badge/focus-Code%20Agent%20Systems-green" />
</p>

---

## 🚨 UPDATE → Rebuilding in Rust

> **Build better harness tooling—not just hoarding leaked Claude Code.  
Don’t collect code. Create outcomes.**

We are actively **rebuilding Claude Code in Rust**, aiming to create a **more powerful, reliable, and production-grade code agent system**.

👉 If you're interested in the Rust implementation and latest progress:  
https://github.com/claw-cli/claw-code-cli

---

## 🚨 UPDATE → Now can `npm start build`

> from : https://github.com/Ta0ing/claude-code_evil

## 📦 What is this repo?

This repository is an **archived snapshot of Claude Code v2.1.88 source code**, which became publicly accessible via a sourcemap bundled in the npm package.

It serves as:

- 📚 A **research artifact** for studying real-world AI agent systems  
- 🔍 A **reference implementation** of large-scale LLM tooling architecture  
- 🧪 A **foundation for rebuilding better systems**

---

On March 31, 2026, Chaofan Shou spotted something unusual: the entire source code of Claude Code had quietly been published to npm—hidden inside a sourcemap file included in the package.

[![The tweet announcing the leak](https://raw.githubusercontent.com/kuberwastaken/claude-code/main/public/leak-tweet.png)](https://raw.githubusercontent.com/kuberwastaken/claude-code/main/public/leak-tweet.png)

This repo captures that moment. It serves as both an archive of the exposed code and a breakdown of how the leak occurred, along with what it reveals about the system behind the scenes.

Let's get into it.

## How did this occur?

This is the part that really makes you pause for a second.

When publishing a JavaScript or TypeScript package to npm, modern build pipelines typically generate **source map files** (`.map`). These files act as a bridge between the bundled/minified production output and the original source code. Their purpose is straightforward: when something breaks in production, stack traces can map back to the exact line in the original source, instead of pointing to an unreadable position inside a compressed bundle.

The important detail, though, is that **source maps often embed the original source code itself**. Not references—actual raw code—stored as strings inside a JSON structure.

A typical `.map` file looks like this:

```json
{
  "version": 3,
  "sources": ["../src/main.tsx", "../src/tools/BashTool.ts", "..."],
  "sourcesContent": ["// full original source code per file", "..."],
  "mappings": "AAAA,SAAS,OAAO..."
}
````

That `sourcesContent` field is the key. It can contain the complete contents of every original file: code, comments, internal constants, prompts—everything. If this file is published, the entire codebase effectively ships with it.

And npm will happily distribute it. Anyone running `npm pack`, inspecting the tarball, or browsing the package contents can access it directly.

This isn’t a new class of issue. It has happened multiple times before, and it will likely happen again. The root cause is usually simple: either `*.map` files aren’t excluded via `.npmignore`, or the bundler isn’t configured to disable source map generation for production builds.

In this case, the project was built using Bun, which generates source maps by default unless explicitly disabled—making this kind of exposure easy to overlook.

![Claude Code source files exposed in npm package](https://raw.githubusercontent.com/kuberwastaken/claude-code/main/public/claude-files.png)](https://raw.githubusercontent.com/kuberwastaken/claude-code/main/public/claude-files.png)

What makes this particularly ironic is the existence of an internal system called “Undercover Mode,” designed to prevent sensitive information from leaking through generated outputs like commits or PR descriptions.

A considerable amount of effort went into ensuring the AI wouldn’t accidentally expose internal details in text—yet the entire codebase ended up being published through a build artifact instead.

---

## What's Claude Under The Hood?

If you've been living under a rock, Claude Code is Anthropic's official CLI tool for coding with Claude and the most popular AI coding agent.

From the outside, it looks like a polished but relatively simple CLI.

From the inside, It's a **785KB [`main.tsx`](https://github.com/kuberwastaken/claude-code/blob/main/main.tsx)** entry point, a custom React terminal renderer, 40+ tools, a multi-agent orchestration system, a background memory consolidation engine called "dream," and much more

Enough yapping, here's some parts about the source code that are genuinely cool that I found after an afternoon deep dive:

---

## BUDDY - A Tamagotchi Inside Your Terminal

I am not making this up.

Claude Code has a full **Tamagotchi-style companion pet system** called "Buddy." A **deterministic gacha system** with species rarity, shiny variants, procedurally generated stats, and a soul description written by Claude on first hatch like OpenClaw.

The entire thing lives in [`buddy/`](https://github.com/kuberwastaken/claude-code/tree/main/buddy) and is gated behind the `BUDDY` compile-time feature flag.

### The Gacha System

Your buddy's species is determined by a **Mulberry32 PRNG**, a fast 32-bit pseudo-random number generator seeded from your `userId` hash with the salt `'friend-2026-401'`:

```typescript
// Mulberry32 PRNG - deterministic, reproducible per-user
function mulberry32(seed: number): () => number {
  return function() {
    seed |= 0; seed = seed + 0x6D2B79F5 | 0;
    var t = Math.imul(seed ^ seed >>> 15, 1 | seed);
    t = t + Math.imul(t ^ t >>> 7, 61 | t) ^ t;
    return ((t ^ t >>> 14) >>> 0) / 4294967296;
  }
}
```

Same user always gets the same buddy.

### 18 Species (Obfuscated in Code)

The species names are hidden via `String.fromCharCode()` arrays - Anthropic clearly didn't want these showing up in string searches. Decoded, the full species list is:

| Rarity | Species |
|--------|---------|
| **Common** (60%) | Pebblecrab, Dustbunny, Mossfrog, Twigling, Dewdrop, Puddlefish |
| **Uncommon** (25%) | Cloudferret, Gustowl, Bramblebear, Thornfox |
| **Rare** (10%) | Crystaldrake, Deepstag, Lavapup |
| **Epic** (4%) | Stormwyrm, Voidcat, Aetherling |
| **Legendary** (1%) | Cosmoshale, Nebulynx |

On top of that, there's a **1% shiny chance** completely independent of rarity. So a Shiny Legendary Nebulynx has a **0.01%** chance of being rolled. Dang.

### Stats, Eyes, Hats, and Soul

Each buddy gets procedurally generated:
- **5 stats**: `DEBUGGING`, `PATIENCE`, `CHAOS`, `WISDOM`, `SNARK` (0-100 each)
- **6 possible eye styles** and **8 hat options** (some gated by rarity)
- **A "soul"** as mentioned, the personality generated by Claude on first hatch, written in character

The sprites are rendered as **5-line-tall, 12-character-wide ASCII art** with multiple animation frames. There are idle animations, reaction animations, and they sit next to your input prompt.

### The Lore

The code references April 1-7, 2026 as a **teaser window** (so probably for easter?), with a full launch gated for May 2026. The companion has a system prompt that tells Claude:

```
A small {species} named {name} sits beside the user's input box and 
occasionally comments in a speech bubble. You're not {name} - it's a 
separate watcher.
```

So it's not just cosmetic - the buddy has its own personality and can respond when addressed by name. I really do hope they ship it.

---

## KAIROS - "Always-On Claude"

Inside [`assistant/`](https://github.com/kuberwastaken/claude-code/tree/main/assistant), there's an entire mode called **KAIROS** i.e. a persistent, always-running Claude assistant that doesn't wait for you to type. It watches, logs, and **proactively** acts on things it notices.

This is gated behind the `PROACTIVE` / `KAIROS` compile-time feature flags and is completely absent from external builds.

### How It Works

KAIROS maintains **append-only daily log files** - it writes observations, decisions, and actions throughout the day. On a regular interval, it receives `<tick>` prompts that let it decide whether to act proactively or stay quiet.

The system has a **15-second blocking budget**, any proactive action that would block the user's workflow for more than 15 seconds gets deferred. This is Claude trying to be helpful without being annoying.

### Brief Mode

When KAIROS is active, there's a special output mode called **Brief**, extremely concise responses designed for a persistent assistant that shouldn't flood your terminal. Think of it as the difference between a chatty friend and a professional assistant who only speaks when they have something valuable to say.

### Exclusive Tools

KAIROS gets tools that regular Claude Code doesn't have:

| Tool | What It Does |
|------|-------------|
| **SendUserFile** | Push files directly to the user (notifications, summaries) |
| **PushNotification** | Send push notifications to the user's device |
| **SubscribePR** | Subscribe to and monitor pull request activity |

 ---

## ULTRAPLAN - 30-Minute Remote Planning Sessions

Here's one that's wild from an infrastructure perspective.

**ULTRAPLAN** is a mode where Claude Code offloads a complex planning task to a **remote Cloud Container Runtime (CCR) session** running **Opus 4.6**, gives it up to **30 minutes** to think, and lets you approve the result from your browser.

The basic flow:

1. Claude Code identifies a task that needs deep planning
2. It spins up a remote CCR session via the `tengu_ultraplan_model` config
3. Your terminal shows a polling state - checking every **3 seconds** for the result
4. Meanwhile, a browser-based UI lets you watch the planning happen and approve/reject it
5. When approved, there's a special sentinel value `__ULTRAPLAN_TELEPORT_LOCAL__` that "teleports" the result back to your local terminal

---

## The "Dream" System - Claude Literally Dreams

Okay this is genuinely one of the coolest things in here.

Claude Code has a system called **autoDream** ([`services/autoDream/`](https://github.com/kuberwastaken/claude-code/tree/main/services/autoDream)) - a background memory consolidation engine that runs as a **forked subagent**. The naming is very intentional. It's Claude... dreaming.

This is extremely funny because [I had the same idea for LITMUS last week - OpenClaw subagents creatively having leisure time to find fun new papers](https://github.com/Kuberwastaken/litmus)

### The Three-Gate Trigger

The dream doesn't just run whenever it feels like it. It has a **three-gate trigger system**:

1. **Time gate**: 24 hours since last dream
2. **Session gate**: At least 5 sessions since last dream  
3. **Lock gate**: Acquires a consolidation lock (prevents concurrent dreams)

All three must pass. This prevents both over-dreaming and under-dreaming.

### The Four Phases

When it runs, the dream follows four strict phases from the prompt in [`consolidationPrompt.ts`](https://github.com/kuberwastaken/claude-code/blob/main/services/autoDream/consolidationPrompt.ts):

**Phase 1 - Orient**: `ls` the memory directory, read `MEMORY.md`, skim existing topic files to improve.

**Phase 2 - Gather Recent Signal**: Find new information worth persisting. Sources in priority: daily logs → drifted memories → transcript search.

**Phase 3 - Consolidate**: Write or update memory files. Convert relative dates to absolute. Delete contradicted facts.

**Phase 4 - Prune and Index**: Keep `MEMORY.md` under 200 lines AND ~25KB. Remove stale pointers. Resolve contradictions.

The prompt literally says:

> *"You are performing a dream - a reflective pass over your memory files. Synthesize what you've learned recently into durable, well-organized memories so that future sessions can orient quickly."*

The dream subagent gets **read-only bash** - it can look at your project but not modify anything. It's purely a memory consolidation pass.

---

## The Full Tool Registry - 40+ Tools

Claude Code's tool system lives in [`tools/`](https://github.com/kuberwastaken/claude-code/tree/main/tools).Here's the complete list:

| Tool | What It Does |
|------|-------------|
| **AgentTool** | Spawn child agents/subagents |
| **BashTool** / **PowerShellTool** | Shell execution (with optional sandboxing) |
| **FileReadTool** / **FileEditTool** / **FileWriteTool** | File operations |
| **GlobTool** / **GrepTool** | File search (uses native `bfs`/`ugrep` when available) |
| **WebFetchTool** / **WebSearchTool** / **WebBrowserTool** | Web access |
| **NotebookEditTool** | Jupyter notebook editing |
| **SkillTool** | Invoke user-defined skills |
| **REPLTool** | Interactive VM shell (bare mode) |
| **LSPTool** | Language Server Protocol communication |
| **AskUserQuestionTool** | Prompt user for input |
| **EnterPlanModeTool** / **ExitPlanModeV2Tool** | Plan mode control |
| **BriefTool** | Upload/summarize files to claude.ai |
| **SendMessageTool** / **TeamCreateTool** / **TeamDeleteTool** | Agent swarm management |
| **TaskCreateTool** / **TaskGetTool** / **TaskListTool** / **TaskUpdateTool** / **TaskOutputTool** / **TaskStopTool** | Background task management |
| **TodoWriteTool** | Write todos (legacy) |
| **ListMcpResourcesTool** / **ReadMcpResourceTool** | MCP resource access |
| **SleepTool** | Async delays |
| **SnipTool** | History snippet extraction |
| **ToolSearchTool** | Tool discovery |
| **ListPeersTool** | List peer agents (UDS inbox) |
| **MonitorTool** | Monitor MCP servers |
| **EnterWorktreeTool** / **ExitWorktreeTool** | Git worktree management |
| **ScheduleCronTool** | Schedule cron jobs |
| **RemoteTriggerTool** | Trigger remote agents |
| **WorkflowTool** | Execute workflow scripts |
| **ConfigTool** | Modify settings (**internal only**) |
| **TungstenTool** | Advanced features (**internal only**) |
| **SendUserFile** / **PushNotification** / **SubscribePR** | KAIROS-exclusive tools |

Tools are registered via `getAllBaseTools()` and filtered by feature gates, user type, environment flags, and permission deny rules. There's a **tool schema cache** ([`toolSchemaCache.ts`](https://github.com/kuberwastaken/claude-code/blob/main/tools/toolSchemaCache.ts)) that caches JSON schemas for prompt efficiency.

---

## The Permission and Security System

Claude Code's permission system in [`tools/permissions/`](https://github.com/kuberwastaken/claude-code/tree/main/tools/permissions) is far more sophisticated than "allow/deny":

**Permission Modes**: `default` (interactive prompts), `auto` (ML-based auto-approval via transcript classifier), `bypass` (skip checks), `yolo` (deny all - ironically named)

**Risk Classification**: Every tool action is classified as **LOW**, **MEDIUM**, or **HIGH** risk. There's a **YOLO classifier** - a fast ML-based permission decision system that decides automatically.

**Protected Files**: `.gitconfig`, `.bashrc`, `.zshrc`, `.mcp.json`, `.claude.json` and others are guarded from automatic editing.

**Path Traversal Prevention**: URL-encoded traversals, Unicode normalization attacks, backslash injection, case-insensitive path manipulation - all handled.

**Permission Explainer**: A separate LLM call explains tool risks to the user before they approve. When Claude says "this command will modify your git config" - that explanation is itself generated by Claude.

---

## Hidden Beta Headers and Unreleased API Features

The [`constants/betas.ts`](https://github.com/kuberwastaken/claude-code/blob/main/constants/betas.ts) file reveals every beta feature Claude Code negotiates with the API:

```typescript
'interleaved-thinking-2025-05-14'      // Extended thinking
'context-1m-2025-08-07'                // 1M token context window
'structured-outputs-2025-12-15'        // Structured output format
'web-search-2025-03-05'                // Web search
'advanced-tool-use-2025-11-20'         // Advanced tool use
'effort-2025-11-24'                    // Effort level control
'task-budgets-2026-03-13'              // Task budget management
'prompt-caching-scope-2026-01-05'      // Prompt cache scoping
'fast-mode-2026-02-01'                 // Fast mode (Penguin)
'redact-thinking-2026-02-12'           // Redacted thinking
'token-efficient-tools-2026-03-28'     // Token-efficient tool schemas
'afk-mode-2026-01-31'                  // AFK mode
'cli-internal-2026-02-09'             // Internal-only (ant)
'advisor-tool-2026-03-01'              // Advisor tool
'summarize-connector-text-2026-03-13'  // Connector text summarization
```

`redact-thinking`, `afk-mode`, and `advisor-tool` are also not released.

---

## Feature Gating - Internal vs. External Builds

This is one of the most architecturally interesting parts of the codebase.

Claude Code uses **compile-time feature flags** via Bun's `feature()` function from `bun:bundle`. The bundler **constant-folds** these and **dead-code-eliminates** the gated branches from external builds. The complete list of known flags:

| Flag | What It Gates |
|------|--------------|
| `PROACTIVE` / `KAIROS` | Always-on assistant mode |
| `KAIROS_BRIEF` | Brief command |
| `BRIDGE_MODE` | Remote control via claude.ai |
| `DAEMON` | Background daemon mode |
| `VOICE_MODE` | Voice input |
| `WORKFLOW_SCRIPTS` | Workflow automation |
| `COORDINATOR_MODE` | Multi-agent orchestration |
| `TRANSCRIPT_CLASSIFIER` | AFK mode (ML auto-approval) |
| `BUDDY` | Companion pet system |
| `NATIVE_CLIENT_ATTESTATION` | Client attestation |
| `HISTORY_SNIP` | History snipping |
| `EXPERIMENTAL_SKILL_SEARCH` | Skill discovery |

Additionally, `USER_TYPE === 'ant'` gates Anthropic-internal features: staging API access (`claude-ai.staging.ant.dev`), internal beta headers, Undercover mode, the `/security-review` command, `ConfigTool`, `TungstenTool`, and debug prompt dumping to `~/.config/claude/dump-prompts/`.

**GrowthBook** handles runtime feature gating with aggressively cached values. Feature flags prefixed with `tengu_` control everything from fast mode to memory consolidation. Many checks use `getFeatureValue_CACHED_MAY_BE_STALE()` to avoid blocking the main loop - stale data is considered acceptable for feature gates.

---

## Other Notable Findings

### The Upstream Proxy
The [`upstreamproxy/`](https://github.com/kuberwastaken/claude-code/tree/main/upstreamproxy) directory contains a container-aware proxy relay that uses **`prctl(PR_SET_DUMPABLE, 0)`** to prevent same-UID ptrace of heap memory. It reads session tokens from `/run/ccr/session_token` in CCR containers, downloads CA certificates, and starts a local CONNECT→WebSocket relay. Anthropic API, GitHub, npmjs.org, and pypi.org are explicitly excluded from proxying.

### Bridge Mode
A JWT-authenticated bridge system in [`bridge/`](https://github.com/kuberwastaken/claude-code/tree/main/bridge) for integrating with claude.ai. Supports work modes: `'single-session'` | `'worktree'` | `'same-dir'`. Includes trusted device tokens for elevated security tiers.

### Model Codenames in Migrations
The [`migrations/`](https://github.com/kuberwastaken/claude-code/tree/main/migrations) directory reveals the internal codename history:
- `migrateFennecToOpus` - **"Fennec"** (the fox) was an Opus codename
- `migrateSonnet1mToSonnet45` - Sonnet with 1M context became Sonnet 4.5
- `migrateSonnet45ToSonnet46` - Sonnet 4.5 → Sonnet 4.6
- `resetProToOpusDefault` - Pro users were reset to Opus at some point

### Attribution Header
Every API request includes:
```
x-anthropic-billing-header: cc_version={VERSION}.{FINGERPRINT}; 
  cc_entrypoint={ENTRYPOINT}; cch={ATTESTATION_PLACEHOLDER}; cc_workload={WORKLOAD};
```
The `NATIVE_CLIENT_ATTESTATION` feature lets Bun's HTTP stack overwrite the `cch=00000` placeholder with a computed hash - essentially a client authenticity check so Anthropic can verify the request came from a real Claude Code install.

### Computer Use - "Chicago"
Claude Code includes a full Computer Use implementation, internally codenamed **"Chicago"**, built on `@ant/computer-use-mcp`. It provides screenshot capture, click/keyboard input, and coordinate transformation. Gated to Max/Pro subscriptions (with an ant bypass for internal users).

### Pricing
For anyone wondering - all pricing in [`utils/modelCost.ts`](https://github.com/kuberwastaken/claude-code/blob/main/utils/modelCost.ts) matches [Anthropic's public pricing](https://docs.anthropic.com/en/docs/about-claude/models) exactly. Nothing newsworthy there.

---

## ⚡ TL;DR

- Claude Code is far more than a CLI — it's a **full agent platform**
- Includes:
  - Multi-agent orchestration
  - Background memory ("Dream system")
  - Tool ecosystem (40+ tools)
  - Proactive assistant (KAIROS)
- Heavy use of:
  - Feature flags
  - Runtime + compile-time gating
  - Prompt engineering as system design

---

## 🧩 Key Architectural Insights

### 1. Multi-Agent Orchestration

Claude Code can operate in a **Coordinator Mode**, spawning multiple workers:

- Parallel research
- Centralized planning
- Distributed execution
- Verification loops

> Parallelism is treated as a first-class primitive.

---

### 2. Tooling System (40+ Tools)

A rich tool ecosystem including:

- File system ops
- Shell execution
- Web browsing
- Task scheduling
- Agent communication

All tools are:
- Schema-driven
- Permission-gated
- Dynamically enabled

---

### 3. Memory System ("Dream")

A background process that:

- Runs periodically
- Consolidates knowledge
- Prunes outdated info
- Maintains long-term memory

> This is essentially **LLM memory compression + reflection**.

---

### 4. Proactive Agent (KAIROS)

An always-on assistant that:

- Observes activity
- Logs behavior
- Acts without explicit prompts

This is **agent → system evolution**.

---

### 5. Feature Gating & Build Strategy

- Compile-time flags (via Bun)
- Dead-code elimination
- Internal vs external builds

This enables:
- Hidden features
- Gradual rollout
- Internal experimentation

---

## 🧠 What We Learned

- AI coding tools are becoming **operating systems**, not just assistants
- Prompt engineering = **system architecture**
- Memory + tools + orchestration = **real agents**
- Production systems rely heavily on:
  - Guardrails
  - Permissions
  - Observability

---

## 🛠 Why Rebuild in Rust?

This codebase is powerful—but:

- ❌ Complex & hard to maintain
- ❌ JS runtime limitations
- ❌ Weak performance guarantees
- ❌ Hard to reason about concurrency

We believe the next-gen agent system should be:

- ⚡ Faster (native performance)
- 🔒 Safer (memory + execution)
- 🧵 Concurrent by design
- 📦 Better for distribution (CLI + infra)

---

## 🚀 Our Direction

We are building:

> A **next-generation code agent runtime** — not just a CLI wrapper.

Focus areas:

- Deterministic agent execution
- Better tool sandboxing
- First-class multi-agent orchestration
- Real memory systems (not prompt hacks)
- Bun / npm-friendly distribution pipeline

---

## 📦 Build & Distribution

Currently working on:

- ✅ Bun-based build pipeline
- 📦 npm distribution
- ⚡ Bun-native execution

Goal:

> Seamless install, instant execution, zero friction.

---

## ⚠️ Disclaimer

This repository is for:

- Research
- Education
- Reverse engineering insights

We do **not claim ownership** of original Claude Code.

---

## ⭐ Final Note

This repo started as an archive.

It is now a **launchpad**.

> Don't collect code.  
> Build systems that ship.
