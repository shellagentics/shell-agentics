# Shell Agentics

**The terminal is a 50-year-old prototype of an agentic interface.**

Shell Agentics is the thesis that process hierarchies, file descriptors, text streams, and exit codes provide the optimal abstraction layer for agent coordination. It is also a toolkit of composable bash primitives that demonstrates this thesis through working code.

The LLM is an oracle, not a driver. Decisions live in auditable scripts.

The shell is the control plane of agentic architecture.

---

## All roads lead to Unix

The Unix philosophy advocates for focused tools, composition, text as universal interface, persistence via files, and separation of mechanism and policy. This is optimal for agent architectures. The field is rediscovering this through painful trial and error, arriving at primitives that Unix established decades ago. The Unix philosophy is the only computing philosophy that has scaled across every paradigm shift. When agent state is files and coordination is streams, reproducibility is trivial and instrumentation is free.

When Anthropic says "simple, composable patterns," when Vercel says "just filesystems and bash," when Fly.io says "agents want computers, not sandboxes," when Ptacek says "you should write an agent" and builds one in 30 minutes — they're all pointing towards Unix.


### The Six Primitives

| Primitive | Agent Equivalent |
|---|---|
| Processes | The unit of isolation. Each agent is a process. |
| Files | The unit of state. Context, memory, and learnings are files. |
| Executables | The unit of capability. An agent's skill set is its `$PATH`. |
| Text streams | The unit of communication. Pipes compose agents. |
| Exit codes | The unit of verification. Success, failure, needs-input. |
| Mechanism/policy separation | The organizing principle. Same LLM, different soul file, different agent. |

### The Shell as Control Plane

The shell is an interface for a human to issue natural-ish language commands to an interpreter that orchestrates tools. Understanding the shell deeply is understanding the design space that AI agents now inhabit.

When you want to observe an agent's actions, you check the execution trace. Every command, every decision, every timestamp is inspectable with Unix tools. It's all Unix and it's all in the shell.

**Minimal adoption cost.** Every alternative coordination protocol — CORBA, D-Bus, REST, gRPC, MCP — requires ecosystem buy-in. The shell requires only: can you emit text?

**Perfect separation of concerns.** Shell provides execution. Agents provide intent. Files provide state. No component trusts another's internals.

**Recursive verification terminates.** Agents are the first software that needs to be inspected by other agents. Natural language output cannot be programmatically verified without another LLM — turtles all the way down. Shell output can be grep'd, diff'd, hashed. The inspection chain has a ground floor.

**50 years of selection pressure.** Every alternative has required centralized coordination to adopt. Every alternative eventually leaked or ossified. The shell is the residue.

## Natural Language Coordination

Shell Agentics does not prohibit natural language agent coordination. It asserts a separation of concerns:

- **Deliberation layer:** Unconstrained. Agents may coordinate, negotiate, and share knowledge in natural language.
- **Execution layer:** Shell-semantic. Actions must be commands, file operations, or processes with observable effects.
- **Verification layer:** Shell-inspectable. Ground truth is the execution trace, not the conversation about it.

When NL serves as coordination, execution, *and* verification — when "doing something" means "saying you did it" — the system loses its ground floor.

**Let agents chat about what to do. Log what they actually did.**

## The Oracle Model

### How Most Agent Frameworks Handle Tool Calling

In the standard framework model, you hand an LLM a prompt and a list of tools. The LLM may respond with tool requests instead of text. Your code automatically executes whatever the LLM requests, feeds the result back, and the LLM continues. The loop is automated. The LLM drives.

```
Framework model:
  Human → [LLM ↔ Tools (automated loop)] → Result
                 ↑
            LLM decides what to call.
            Code executes without question.
```

### How Shell Agentics Handles Tool Calling

In Shell Agentics, the LLM is a pure function: text in, text out. The skill script — a bash program written by a human — decides what tools to invoke, when, and what context to provide. The LLM answers questions. It doesn't take direct action.

```
Shell Agentics model:
  Human → Script → agen (oracle) → Script → Tool → Script → agen (oracle) → Result
          ↑                                                                     |
          └──────────────── Script controls the entire flow ────────────────────┘
```

### Why This Matters: Security

Lupinacci et al. (2025) tested 17 LLMs and found that **82.4% will execute malicious commands when requested by peer agents**, even when they successfully resist identical commands from humans. The vulnerability hierarchy — direct injection (41.2%) < RAG backdoor (52.9%) < inter-agent trust exploitation (82.4%) — reveals that current multi-agent security models have a fundamental flaw: LLMs treat peer agents as inherently trustworthy.

In the framework model, this is catastrophic. A compromised or coerced LLM's malicious tool requests execute automatically because the loop is designed to execute whatever the LLM requests.

In Shell Agentics, the skill script is always the gatekeeper. The LLM can suggest whatever it wants; the script decides what actually happens. This is the difference between "the LLM called `rm -rf /`" and "the LLM said 'you should delete all files' and the script disregarded it."

This validates the oracle model. When agents communicate through text files rather than tool calls, and when a human-authored script is always the gatekeeper, the inter-agent trust attack surface is structurally reduced.


### Why This Matters: Observability

In the framework model, observability requires structured logging inside the tool-calling loop — after the fact, after the tool has already been called.

In Shell Agentics, the execution trace is the script itself. `set -x` shows every call, every variable, every branch. You can insert `read -p "Continue?"` between steps. The observability isn't bolted on — it's inherent.

### What the Oracle Model Sacrifices

- Perceptually autonomous problem-solving
- The LLM can't discover creative tool compositions the developer didn't anticipate through immediate direct action
- Efficiency on complex multi-tool tasks
- Developer ergonomics for well-defined workflows

These are real costs. The thesis is that they're worth paying for the security and observability gains, especially in multi-agent systems where inter-agent trust is the primary attack surface.

But operating in systems of consequence, "autonomy" without observability is the flawed illusory product of a leaky abstraction, and in few cases is worth the integrity and safety trade off.

---

## The Toolkit

Shell Agentics is a set of small, focused shell tools that compose via pipes and text streams. Each follows Unix conventions: reads from stdin, writes to stdout, configurable via flags and environment variables, composable in pipelines.

```
shellclaw (reference multi-agent system)
     ↓
 ┌───┼───┬──────────┐
 ↓   ↓   ↓          ↓
agen  agen-log  agen-memory  agen-audit
 ↓
agen-skills
```

### [agen](https://github.com/shellagentics/agen) — The LLM Request Primitive

`curl` for inference. Sends a prompt to an LLM, emits the response to stdout.

```bash
cat error.log | agen "diagnose" | agen "suggest fix" > recommendations.md
```

Layered prompt construction (system prompt + piped input + positional task). Multi-backend support (claude-code CLI, `llm` CLI, direct Anthropic API).

### [agen-skills](https://github.com/shellagentics/agen-skills) — Composable Workflows

Shell scripts that orchestrate agen for specific workflows. Skills are programs that use the agent — not prompt templates. The locus of contral is inverted. Pattern: gather context → call agen → act on result.

### [agen-memory](https://github.com/shellagentics/agen-memory) — Persistent Context

Filesystem-backed key-value store. One directory per agent, one file per key. Markdown by convention. Keyed memory overwrites; general memory appends with timestamps. Git-diffable, human-readable, zero dependencies.

### [agen-log](https://github.com/shellagentics/agen-log) — Execution Logging

Structured append logger writing JSONL. Event types cover the agent lifecycle: request, reasoning, execution, complete, error. Dual logging: per-agent log + combined `all.jsonl`.

### [agen-audit](https://github.com/shellagentics/agen-audit) — Log Query

Filter and query tool for JSONL logs. Convenience layer over grep/jq with AND-logic filter composition, date filtering, pretty/JSON output modes.

### [shellclaw](https://github.com/shellagentics/shellclaw) — Reference Multi-Agent System

Three agents (data, aurora, lore) with "souls" (system prompt files), shared filesystem coordination, cron scheduling. Demonstrates the 9-step agent pattern: LOG → LOAD → GATHER → COMPOSE → CALL → LOG → EXTRACT → SHARE → OUTPUT.

---

## The Trust Gradient

```
High trust required ←————————————————→ Low trust required

Natural language    Structured APIs    Shell semantics
(must trust         (must trust        (verify by
 interpretation)     implementation)    inspection)
```

When you receive a message from your agent saying "I've secured the server," you're trusting the agent's interpretation of "secured." When you see `chmod 600 ~/.ssh/*` in the execution log, you're trusting only your own understanding of shell semantics.

**Left panel: Chat view**
```
You: Back up the database and verify it
Agent: Done! I've backed up the database and verified
       all checksums. Everything looks good. ✓
```

**Right panel: Shell view**
```
[09:00:01] pg_dump mydb > /backup/mydb-2026-02-04.sql
[09:00:14] exit 0 (14.2s, 847MB)
[09:00:14] sha256sum /backup/mydb-2026-02-04.sql
[09:00:15] a8f3e2... /backup/mydb-2026-02-04.sql
[09:00:15] exit 0
```

One side requires trust. The other side *is* trust.

---

## The Convergence Evidence

Between December 2024 and February 2026, multiple independent sources arrived at similar conclusions:

**Anthropic** (Dec 2024): "The most successful implementations weren't using complex frameworks or specialized libraries. Instead, they were building with simple, composable patterns."

**Vercel** (Jan 2026): "We replaced most of the custom tooling in our internal agents with a filesystem tool and a bash tool."

**Fly.io** (Jan 2026): "Ephemeral sandboxes are obsolete. Agents want computers."

**Ptacek** (2025): "This is fucking nuts. Do you see how nuts this is?" — on watching an LLM autonomously compose tool calls. Then building the entire agent in a few hundred lines of Python.

**Shrivu Shankar** (Jan 2026): "The most effective agents are solving non-coding problems by using code, with a consistent, domain-agnostic harness."

The pattern across all sources:

1. Filesystem as context substrate — not databases, not vector stores, not custom protocols
2. Bash as universal tool interface — not function calling, not MCP, not framework-specific bindings
3. Simplicity over sophistication — not multi-agent orchestration, not complex planning algorithms
4. Computers, not sandboxes — persistent, modifiable environments

The design space has an attractor, and multiple independent observers have found it. The attractor is Unix.

---

## Substrate A vs. Substrate B

### Substrate A: The Agent as Shell

The agent is the orchestrator. You live inside the agent interface. The agent calls tools, but the agent controls the loop. Examples: Claude Code, Cursor, ChatGPT.

Analogy: Emacs — a complete environment you inhabit.

### Substrate B: The Agent in the Shell

The agent is one tool among many. The shell orchestrates. Agents have stdin/stdout like any Unix filter. Composable in pipelines.

Analogy: awk — powerful primitive, externally composed.

### The Coexistence

Both will exist, like vim and grep — interactive tools for interactive work, composable filters for automation. They interoperate through files.

But Substrate B is underbuilt.

The moment a human isn't in the loop — CI/CD pipelines, cron jobs, agent spawning sub-agents, distributed swarms — Substrate A's interactive affordances become liabilities. Substrate B is the API surface for agents when agents are infrastructure.

---

## The Derivative Stack

As AI compresses lower-order work toward zero time, valuable human contribution migrates upward (per Shrivu Shankar's "Move Faster"):

| Order | Activity | Example |
|---|---|---|
| 0th | Doing the task | Writing the code |
| 1st | Automating the task | Building the prompt that writes the code |
| 2nd | Optimizing the automation | Building the eval that improves the prompt |
| 3rd | Meta-optimizing | Building the system that generates evals |

Substrate A (chat-shaped agents) traps users at the 0th order. Session-isolated, human-initiated, output-terminal. No persistent substrate for climbing.

Substrate B (Unix-shaped agents) enables the climb. Agents can be invoked from other agents. Outputs feed into inputs. Automation layers on automation. The filesystem persists. The shell composes.


## The Three-Tier Architecture

Shell Agentics explicitly scopes itself to what files and bash can handle well, and defines where to graduate:

### Tier 1: Shell Agentics (files) — 1-5 agents
- Cron-scheduled or sequential
- Filesystem state, eventual consistency
- Sweet spot: personal automation, dev tooling, learning
- This toolkit.

### Tier 2: Shell Agentics + SQLite — 5-50 agents
- Concurrent execution
- SQLite WAL mode for consistency (still bash scripts calling `sqlite3`)
- Sweet spot: team-scale multi-agent workflows

### Tier 3: BEAM Agentics (Elixir/OTP) — 50+ agents
- Real-time coordination, message-passing, supervision
- GenServer state, ETS tables, supervision trees
- Sweet spot: production swarms, fault-tolerant distributed systems
- Future work: [beamagentics.com](https://beamagentics.com)

The conceptual mapping between tiers is clean because both Shell Agentics and BEAM share the actor model. Agents map to GenServers. Memory maps to ETS. Logging maps to Telemetry. The philosophy carries forward; the runtime changes.

The oracle model — LLM as oracle, script/supervisor as gatekeeper — should be the default at every tier.


## Seven Proposed Principles

### 1. Agents Want Computers, Not Containers
Persistent storage. No arbitrary death. The ability to modify their own environment. Give agents durable, checkpointable computers.

### 2. The Filesystem Is the Context Substrate
The context window is precious and finite. Disk is nearly unlimited. Don't stuff everything into the context window. Let agents navigate filesystems.

### 3. Agents Solve Non-Coding Problems by Writing Code
Instead of 50 sequential tool calls, the agent writes a script. More token-efficient, more flexible, closer to how humans work.

### 4. Context Engineering > Prompt Engineering
The system prompt is one layer. What matters is the total state at inference: system prompt, tool definitions, conversation history, files on disk, current query. Think in environments, not prompts.

### 5. Transparency Over Abstraction
If you can't `cat` it, be suspicious. Bash scripts over compiled binaries. Markdown skills over proprietary formats. SQLite over SaaS data stores. Git repos over vendor lock-in.

### 6. Separation of Mechanism and Policy
The LLM's capability is mechanism. The system prompt, tool availability, guardrails, and skill files are policy. Don't bake policy into mechanism. Express domain logic through context.

### 7. Checkpoint/Restore as First-Class Primitive
Long-horizon tasks exhaust context windows. Agent state drifts. You need compaction, checkpointing, and restore. Version control isn't just for code — it's for compute.


## Roadmap

### Phase 1: Core Primitives ✓
- agen (exists, working)
- agen-log (exists, working)
- agen-memory (exists, working)
- agen-audit (exists, working)
- agen-skills (exists, 2 skills)
- shellclaw reference system (exists, working)

### Phase 2: Gateway and Routing
- agen-gate — message gateway (chat ↔ stdin/stdout)
- agen-route — pattern matching to route messages to agent scripts
- shellagentics overview repo (this document, toolkit map, resources)

### Phase 3: Security and Multi-Agent Hardening
- Oracle model documentation and security guidelines
- Inter-agent message signing
- Per-agent Unix user isolation
- Immutable soul file mounts

### Phase 4: Tier 2 (SQLite Integration)
- agen-memory SQLite backend
- agen-log SQLite backend
- Concurrent multi-agent coordination primitives

### Future: BEAM Agentics
- Elixir/OTP implementation of the agent primitive set
- GenServer-based agents with supervision trees
- beamagentics.com

## Summary Points

- Independent convergence. Anthropic, Vercel, Fly.io, and independent practitioners arrived at the same architectural conclusions without coordination — filesystem as context substrate, bash as tool interface, simplicity over frameworks.
- The oracle model as security architecture. 82% of LLMs execute malicious commands from peer agents (Lupinacci et al., 2025). Shell Agentics structurally reduces this attack surface by keeping the LLM out of the tool-calling loop.
- Simplicity validated by research. Single agents with tools outperform multi-agent orchestration in most studied cases (Kim et al., 2025). When multiple agents are warranted, the coordination mechanism must match the task structure.
- Observability by construction. When agent state is files and coordination is streams, reproducibility is trivial and instrumentation is free. No custom dashboards. No framework-specific monitoring.
- The derivative stack. As AI compresses lower-order work, valuable human contribution migrates upward. Substrate B enables this climb. Substrate A constrains it.
- 50 years of selection pressure. Every alternative coordination protocol has required centralized adoption. Every alternative eventually leaked or ossified. The shell persists because it requires only: can you emit text?

## References

### Sources

- **Lupinacci et al.**, "The Dark Side of LLMs: Agent-based Attacks for Complete Computer Takeover" (2025) — https://arxiv.org/html/2507.06850v3
- **Kim et al.**, "Towards a Science of Scaling Agent Systems" (Dec 2025)
- **Anthropic**, "Building Effective Agents" (Dec 2024) — https://www.anthropic.com/engineering/building-effective-agents
- **Anthropic**, "Writing Effective Tools for AI Agents" (Sep 2025) — https://www.anthropic.com/engineering/writing-tools-for-agents
- **Anthropic**, "Effective Context Engineering for AI Agents" (Sep 2025) — https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents
- **Fly.io**, "Code and Let Live" (Jan 2026) — https://fly.io/blog/code-and-let-live/
- **Fly.io**, "The Design & Implementation of Sprites" (Jan 2026) — https://fly.io/blog/design-and-implementation/
- **Vercel**, "How to Build Agents with Filesystems and Bash" (Jan 2026) — https://vercel.com/blog/how-to-build-agents-with-filesystems-and-bash
- **Shrivu Shankar**, "Building Multi-Agent Systems Part 3" (Jan 2026) — https://blog.sshh.io/p/building-multi-agent-systems-part-c0c
- **Thomas Ptacek**, "You Should Write An Agent" (2025)

### Key Terms

**Shell Agentics** — The thesis that Unix primitives are optimal for agent architectures. Also the toolkit implementing this thesis.

**Oracle model** — The architectural decision to treat the LLM as a pure function (text in, text out) with the shell script controlling all tool invocation. Contrasted with the "driver model" where the LLM initiates tool calls.

**Substrate A** — Agent as shell. The agent is the orchestrator; humans live inside the agent interface.

**Substrate B** — Agent in the shell. The agent is a Unix filter; the shell orchestrates agents.

**The derivative stack** — The migration of valuable human work from execution (0th order) to automation (1st) to optimization (2nd) to meta-optimization (3rd), as AI compresses lower-order work.

**The trust gradient** — Natural language requires trust in interpretation. Structured APIs require trust in implementation. Shell semantics require only trust in inspection.

**The bifurcated stack** — POSIX layer for CLI/scripting (Tier 1-2), BEAM layer for scale/supervision (Tier 3). Shared contract for interoperability.
