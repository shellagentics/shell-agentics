# Shell Agentics

**The terminal is a 50-year-old prototype of an agentic interface.**

Shell Agentics is the idea that process hierarchies, file descriptors, text streams, and exit codes provide the optimal abstraction layer for agent coordination. It is also a proof of concept toolkit of composable bash primitives that demonstrates this thesis through working code.

It treats the LLM is an oracle, not a driver. Tool dispatch is controlled by explicit allowlists in shell scripts.

The shell becomes the control plane of agent architectures.

---

## All Roads Lead to Unix

The Unix philosophy advocates for focused tools, composition, text as universal interface, persistence via files, and separation of mechanism and policy. This is optimal for agent architectures, and the field is discovering this through painful trial and error, reverse engineering primitives that Unix established decades ago. The Unix philosophy is the only computing philosophy that has scaled across every paradigm shift. When agent state is files and coordination is streams, reproducibility is trivial and instrumentation is free.

### The Six Primitives

| Primitive | Agent Equivalent |
|---|---|
| Processes | The unit of isolation. Each agent is a process. |
| Files | The unit of state. Context, memory, and learnings are files. |
| Executables | The unit of capability. An agent's skill set is its `$PATH`. |
| Text streams | The unit of communication. Pipes compose agents. |
| Exit codes | The unit of verification. Success, failure, needs-input. |
| Mechanism/policy separation | The organizing principle. Same LLM, different soul file, different agent. |

## The Shell as Control Plane

The shell is an interface for a human to issue natural-ish language commands to an interpreter that orchestrates tools. Understanding the shell is understanding the design space that AI agents now inhabit.

When you want to observe an agent's actions, you check the execution trace. Every command, every decision, every timestamp is inspectable with Unix tools. 

**Minimal adoption cost.** Every alternative coordination protocol — CORBA, D-Bus, REST, gRPC, MCP — requires ecosystem buy-in. The shell requires only: can you emit text?

**Perfect separation of concerns.** Shell provides execution. Agents provide intent. Files provide state. No component trusts another's internals.

**Recursive verification terminates.** Agents are the first software that needs to be inspected by other instances of itself. Natural language output cannot be programmatically verified without another LLM — turtles all the way down. Shell output can be grep'd, diff'd, hashed. The inspection chain has a ground floor.

**50 years of selection pressure.** Every alternative has required centralized coordination to adopt. Every alternative eventually leaked or ossified. The shell is the result of selection.

## Natural Language Coordination

When natural language serves as coordination, execution, *and* verification — when "doing something" means "saying you did it" — the system loses its ground floor. Shell Agentics does not prohibit natural language agent coordination, it asserts a separation of concerns:

- **Deliberation layer:** Unconstrained. Agents may coordinate, negotiate, and share knowledge in natural language.
- **Execution layer:** Shell-semantic. Actions must be commands, file operations, or processes with observable effects.
- **Verification layer:** Shell-inspectable. Ground truth is the execution trace, not the conversation about it.

**Let agents chat about what to do. Log what they actually did.**

## The Oracle Model

### How Most Agent Frameworks Handle Tool Calling

In the standard framework model, the harness (Claude Code, Codex, Cursor, et al) sends an LLM a text prompt and a list of "tools", ie. actions that the harness is able to take — reading files, executing commands, searching the web, modifying code. The LLM can ask the harness to take these actions by responding back to the harness with a structured tool request (tool name + input parameters as JSON) instead of text. The harness either executes the tool automatically or confirms with the user (approval gate). The harness then feeds the result back and the LLM generates again. This is a multi-turn protocol between harness and LLM. You could view the LLM as a pure function, and the harness as a mediator executing side effects. The loop is automated. The LLM drives.

```
Framework model:
  Human → [LLM ↔ Tools (automated loop)] → Result
                 ↑
            LLM decides what to call.
            Code executes without question.
```

### How Shell Agentics Handles Tool Calling

In Shell Agentics, the orchestrating script calls `agent` to get a response. When the LLM returns a structured tool-call request — JSON specifying the tool name and arguments — the script parses the response and matches the tool name against a `case` statement: an explicit allowlist of permitted tools.

```bash
case "$tool" in
  ping)    result=$("$TOOLS_DIR/ping.sh" "$args") ;;
  curl)    result=$("$TOOLS_DIR/curl.sh" "$args") ;;
  *)       result="Tool not permitted: $tool" ;;
esac
```

Only tools listed in the `case` execute. Unmatched requests fall through. This is default-deny dispatch — the same posture as a firewall with an allowlist. The LLM can request any tool; only those with an explicit match will run.

```
Shell Agentics model:
  Human → Script → agent → Script → [case match?] → Tool → Script → agent → Result
          ↑                                                                     |
          └──────────────── Script controls the entire flow ────────────────────┘
```

### Why This Matters: Security

Lupinacci et al. (2025) tested 17 LLMs and found that **82.4% will execute malicious commands when requested by peer agents**, even when they successfully resist identical commands from humans. The vulnerability hierarchy — direct injection (41.2%) < RAG backdoor (52.9%) < inter-agent trust exploitation (82.4%) — reveals that current multi-agent security models have a fundamental flaw: LLMs treat peer agents as inherently trustworthy.

In the framework model, this is catastrophic. A compromised or coerced LLM's malicious tool requests execute automatically because the loop is designed to execute whatever the LLM requests.

In Shell Agentics, the agent script is always the gatekeeper. The LLM can suggest whatever it wants; the script decides what actually happens. This is the difference between "the LLM called `rm -rf /`" and "the LLM said 'you should delete all files' and the script disregarded it."

This validates the oracle model. When agents communicate through text files rather than tool calls, and when a human-authored script is always the gatekeeper, the inter-agent trust attack surface is structurally reduced.


### Why This Matters: Observability

Every tool invocation is a visible line in the script. The execution trace — which tools ran, with what arguments, in what order — is available through standard Unix mechanisms: `set -x` traces every command as it executes, [`alog`](https://github.com/shellagentics/alog) records structured JSONL events, process output flows through pipes. Logging, conditional pauses (`read -p "Continue?"`), and additional checks can be inserted between any steps. Any system can achieve observability through sufficient logging. What matters is how naturally it integrates with the execution model.

### Why This Matters: Control Flow Legibility

The orchestrating script is a readable artifact that exists before runtime. `cat agent-1.sh` shows every tool that could run and under what conditions. The `case` statement is auditable as a specification — you can read it and know with certainty which tools are permitted and which are not.

This legibility is diffable (`git diff` shows exactly what changed in the control flow between versions), git-blameable (who authorized this tool and when), and reviewable in a pull request (the team can inspect the control flow before it runs in production). It describes what *can* happen, not just what *did* happen.

When the agentic loop lives in the shell script, both properties are present: you can observe what happened, and you can read what can happen. When the loop is internalized inside a framework process, observability is still achievable through logging — but control flow legibility requires understanding the framework's dispatch mechanism, its configuration, and its runtime state, rather than reading a single text file.

### What the Oracle Model Sacrifices

- Perceptually autonomous problem-solving
- The LLM can't discover creative tool compositions the developer didn't anticipate through immediate direct action
- Efficiency on complex multi-tool tasks
- Developer ergonomics for well-defined workflows

These are real costs. The thesis is that they're worth paying for the security, observability, and control flow legibility gains, especially in multi-agent systems where inter-agent trust is the primary attack surface.

---

## The Toolkit

Shell Agentics is a set of small, focused shell tools that compose via pipes and text streams. Each follows Unix conventions: reads from stdin, writes to stdout, configurable via flags and environment variables, composable in pipelines.

```
shellclaw (reference multi-agent system)
     ↓
 ┌───┼───┬──────────┐
 ↓   ↓   ↓          ↓
agent  alog  amem  aaud
 ↓
ascr
```

### [agent](https://github.com/shellagentics/agent) — The LLM Request Primitive

`curl` for inference. Sends a prompt to an LLM, emits the response to stdout.

```bash
cat error.log | agent "diagnose" | agent "suggest fix" > recommendations.md
```

Layered prompt construction (system prompt + piped input + positional task). Multi-backend support (claude-code CLI, `llm` CLI, direct Anthropic API).

### [ascr](https://github.com/shellagentics/ascr) — Composable Workflow Scripts

Shell scripts that orchestrate agent for specific workflows. Scripts are programs that use the agent — not prompt templates. The locus of control is inverted. Pattern: gather context → call agent → act on result.

### [amem](https://github.com/shellagentics/amem) — Persistent Context

Filesystem-backed key-value store. One directory per agent, one file per key. Markdown by convention. Keyed memory overwrites; general memory appends with timestamps. Git-diffable, human-readable, zero dependencies.

### [alog](https://github.com/shellagentics/alog) — Execution Logging

Structured append logger writing JSONL. Event types cover the agent lifecycle: request, reasoning, execution, complete, error. Dual logging: per-agent log + combined `all.jsonl`.

### [aaud](https://github.com/shellagentics/aaud) — Log Query

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

**Chat view**
```
You: Back up the database and verify it
Agent: Done! I've backed up the database and verified
       all checksums. Everything looks good. ✓
```

**Shell view**
```
[09:00:01] pg_dump mydb > /backup/mydb-2026-02-04.sql
[09:00:14] exit 0 (14.2s, 847MB)
[09:00:14] sha256sum /backup/mydb-2026-02-04.sql
[09:00:15] a8f3e2... /backup/mydb-2026-02-04.sql
[09:00:15] exit 0
```

The difference here is not about what data is available. Any agent harness can log the same `pg_dump` and `sha256sum` commands. A framework user who reads verbose logs sees the same execution trace. The difference is about what the system presents as its primary output — and therefore what the user actually looks at.

The chat model's primary output is a natural language claim about execution. "I've backed up the database and verified all checksums" is a sentence the LLM generated. It could be accurate. It could also be confabulated — the LLM might say "verified all checksums" when it only ran `pg_dump` and skipped the verification. Natural language claims can diverge from what actually happened, and the divergence is invisible in the primary interface.

The shell model's primary output is the execution itself. `sha256sum /backup/mydb-2026-02-04.sql` with an exit code is not a claim — it is an artifact. That command either ran or it didn't. You are not trusting the LLM's description of what happened. You are reading what happened.

One model presents a narrator who describes execution. The other removes the narrator and presents the execution directly. A framework user who ignores the natural language summary and reads the execution logs gets the same verification — but the system's default interface is the summary, not the log. A shell user who only reads the final `echo` output is trusting a summary too — but the system's default interface is the trace, not the summary.

The qualitative difference is in what serves as truth: natural language claims about execution, or execution artifacts themselves.


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


## Proposed Principles

Between December 2024 and February 2026, multiple independent sources arrived at the same architectural conclusions. The design space has an attractor, and these observers found it independently. The attractor is Unix. This leads to some proposed design principles.

### 1. Agents Want Computers, Not Containers

Persistent storage. No arbitrary death. The ability to modify their own environment. Give agents durable, checkpointable computers.

> "They don't want containers. They don't want 'sandboxes'. They want computers."
> — Kurt Mackey, Fly.io ([Code and Let Live](https://fly.io/blog/code-and-let-live/), Jan 2026)

### 2. The Filesystem Is the Context Substrate

The context window is precious and finite. Disk is nearly unlimited. Don't stuff everything into the context window. Let agents navigate filesystems.

> "Maybe the best architecture is almost no architecture at all. Just filesystems and bash."
> — Ashka Stephen, Vercel ([How to Build Agents with Filesystems and Bash](https://vercel.com/blog/how-to-build-agents-with-filesystems-and-bash), Jan 2026)

### 3. Agents Solve Non-Coding Problems by Writing Code

Instead of 50 sequential tool calls, the agent writes a script. More token-efficient, more flexible, closer to how humans work.

> "The most effective agents are solving non-coding problems by using code, with a consistent, domain-agnostic harness."
> — Shrivu Shankar ([Building Multi-Agent Systems Part 3](https://blog.sshh.io/p/building-multi-agent-systems-part-c0c), Jan 2026)

### 4. Context Engineering > Prompt Engineering

The system prompt is one layer. What matters is the total state at inference: system prompt, tool definitions, conversation history, files on disk, current query. Think in environments, not prompts.

> "Building with language models is becoming less about finding the right words and phrases for your prompts, and more about answering the broader question of 'what configuration of context is most likely to generate our model's desired behavior?'"
> — Anthropic Applied AI team ([Effective Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents), Sep 2025)

### 5. Transparency Over Abstraction

If you can't `cat` it, be suspicious. Bash scripts over compiled binaries. Markdown over proprietary formats. SQLite over SaaS data stores. Git repos over vendor lock-in.

> "Frameworks can help you get started quickly, but don't hesitate to reduce abstraction layers and build with basic components as you move to production."
> — Erik Schluntz & Barry Zhang, Anthropic ([Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents), Dec 2024)

### 6. Separation of Mechanism and Policy

The LLM's capability is mechanism. The system prompt, tool availability, guardrails, and script files are policy. Don't bake policy into mechanism. Express domain logic through context.

> "The agent and its tool execution run on separate compute. You trust the agent's reasoning, but the sandbox isolates what it can actually do."
> — Ashka Stephen, Vercel ([How to Build Agents with Filesystems and Bash](https://vercel.com/blog/how-to-build-agents-with-filesystems-and-bash), Jan 2026)

### 7. Checkpoint/Restore as First-Class Primitive

Long-horizon tasks exhaust context windows. Agent state drifts. You need compaction, checkpointing, and restore. Version control isn't just for code — it's for compute.

> "Checkpoints are so fast we want you to use them as a basic feature of the system and not as an escape hatch when things go wrong; like a git restore, not a system restore."
> — Thomas Ptacek, Fly.io ([The Design & Implementation of Sprites](https://fly.io/blog/design-and-implementation/), Jan 2026)


## Roadmap

### Phase 1: Core Primitives ✓
- agent (exists, working)
- alog (exists, working)
- amem (exists, working)
- aaud (exists, working)
- ascr (exists, 2 scripts)
- shellclaw reference system (exists, working)

### Phase 2: Gateway and Routing
- agent-gate — message gateway (chat ↔ stdin/stdout)
- agent-route — pattern matching to route messages to agent scripts
- shellagentics overview repo (this document, toolkit map, resources)

### Phase 3: Security and Multi-Agent Hardening
- Oracle model documentation and security guidelines
- Inter-agent message signing
- Per-agent Unix user isolation
- Immutable soul file mounts

### Phase 4: Tier 2 (SQLite Integration)
- amem SQLite backend
- alog SQLite backend
- Concurrent multi-agent coordination primitives

### Future: BEAM Agentics
- Elixir/OTP implementation of the agent primitive set
- GenServer-based agents with supervision trees
- beamagentics.com

## Summary Points

- Independent convergence. Anthropic, Vercel, Fly.io, and independent practitioners arrived at the same architectural conclusions without coordination — filesystem as context substrate, bash as tool interface, simplicity over frameworks.
- The oracle model as security architecture. 82% of LLMs execute malicious commands from peer agents (Lupinacci et al., 2025). Shell Agentics structurally reduces this attack surface by keeping the LLM out of the tool-calling loop.
- Simplicity validated by research. Single agents with tools outperform multi-agent orchestration in most studied cases (Kim et al., 2025). When multiple agents are warranted, the coordination mechanism must match the task structure.
- Observability and control flow legibility. When agent state is files and coordination is streams, the execution trace is available through standard Unix tools. The orchestrating script is a readable artifact — diffable, git-blameable, reviewable — that shows what *can* happen before runtime, not just what *did* happen after the fact.
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

**Oracle model** — The architectural decision to keep the LLM out of the tool-execution loop. The LLM may request tool calls via structured JSON, but the shell script parses the request and matches it against an explicit `case` allowlist. Default-deny dispatch. Contrasted with the "driver model" where the framework automatically executes whatever the LLM requests.

**Substrate A** — Agent as shell. The agent is the orchestrator; humans live inside the agent interface.

**Substrate B** — Agent in the shell. The agent is a Unix filter; the shell orchestrates agents.

**The derivative stack** — The migration of valuable human work from execution (0th order) to automation (1st) to optimization (2nd) to meta-optimization (3rd), as AI compresses lower-order work.

**The trust gradient** — Natural language requires trust in interpretation. Structured APIs require trust in implementation. Shell semantics require only trust in inspection.

**The bifurcated stack** — POSIX layer for CLI/scripting (Tier 1-2), BEAM layer for scale/supervision (Tier 3). Shared contract for interoperability.
