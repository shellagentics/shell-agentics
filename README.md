# Shell Agentics

Shell Agentics is the idea that process hierarchies, file descriptors, text streams, and exit codes provide the optimal abstraction layer for agent coordination. It is also a proof of concept toolkit of composable Bash primitives that demonstrates this thesis through working code.

## All Roads Lead to Unix

The Unix philosophy advocates for focused tools, composition, text as universal interface, persistence via files, and separation of mechanism and policy. This is optimal for agent architectures, and the field is discovering this progressively through trial and error, reverse engineering primitives that Unix established decades ago. The Unix philosophy is the only computing philosophy that has scaled across every paradigm shift. 

When agent state is files and coordination is streams, reproducibility is trivial, instrumentation is free, and how much the LLM controls becomes a decision you make per agent.

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

The terminal is a 50-year-old prototype of an agentic interface. The shell is an interface for a human to issue natural-ish language commands to an interpreter that orchestrates tools. Understanding it is understanding the design space that AI agents now inhabit. The shell is the control plane of agentic architecture.

When you want to observe an agent's actions, you check the execution trace. Every command, every decision, every timestamp is inspectable with Unix tools. 

There is minimal adoption cost. Every alternative coordination protocol — CORBA, D-Bus, REST, gRPC, MCP — requires ecosystem buy-in. The shell requires only: can you emit text?

Unix has the ideal separation of concerns. Shell provides execution. Agents provide intent. Files provide state. No component trusts another's internals.

It allows for recursive verification termination. Agents are the first software that needs to be inspected by other instances of itself. Natural language output cannot be programmatically verified without another LLM — turtles all the way down. Shell output can be grep'd, diff'd, hashed. The inspection chain has a ground floor.

## Natural Language Coordination

When natural language serves exclusively as coordination, execution, *and* verification — when "doing something" means "saying you did it" — the system loses its ground floor.

Building agents in the shell does not prohibit natural language agent coordination, but asserts a separation of concerns:

- **Deliberation layer:** Unconstrained. Agents may coordinate, negotiate, and share knowledge in natural language.
- **Execution layer:** Shell-semantic. Actions must be commands, file operations, or processes with observable effects.
- **Verification layer:** Shell-inspectable. Ground truth is the execution trace, not the conversation about it.

How well this separation holds depends on which control model you choose.

In **oracle mode**, the separation is enforced intentionally. The LLM only participates in the deliberation layer — it answers questions, provides judgment. All execution is shell commands in the orchestration script. Verification is the execution trace. The layers can't bleed into each other because the LLM has no mechanism to execute.

In **bounded mode**, the separation holds but requires attention. The LLM can request tools from an allowlist, so it participates in execution decisions — but only within bounds the script author defined. Verification remains shell-inspectable.

In **autonomous mode**, the separation requires discipline. The LLM controls execution decisions. The logging and audit primitives still capture what happened, so verification remains possible — but the deliberation and execution layers are no longer structurally separated. This is the same posture as traditional frameworks.

**Let agents chat about what to do. Log what they actually did.** In oracle mode, this is guaranteed. In autonomous mode, it's a practice you have to maintain.

## The Oracle Model

Shell Agentics is vocabulary, not a framework. Like a modular synthesizer — where oscillators, filters, and envelopes are raw signal primitives that can be patched into any configuration — Shell Agentics provides composable primitives (`agen`, `agen-log`, `agen-memory`, `agen-audit`) that support multiple control architectures without infrastructure changes.

### How Most Agent Frameworks Handle Tool Calling

In the standard framework model, the harness (Claude Code, Codex, Cursor, et al) sends an LLM a text prompt and a list of "tools", ie. actions that the harness is able to take — reading files, executing commands, searching the web, modifying code. The LLM can ask the harness to take these actions by responding back to the harness with a structured tool request (tool name + input parameters as JSON) instead of text. The harness either executes the tool automatically or confirms with the user (approval gate). The harness then feeds the result back and the LLM generates again. This is a multi-turn protocol between harness and LLM. You could view the LLM as a pure function, and the harness as a mediator executing side effects. The loop is automated. The LLM drives.

```
Framework model:
  Human → [LLM ↔ Tools (automated loop)] → Result
                 ↑
            LLM decides what to call.
            Harness executes. LLM decides next step.
```

Frameworks are built around this loop. The tool-calling loop *is* the framework. If you want the LLM to not drive, you're not configuring the framework — you're abandoning it.

### Three Control Models

Shell Agentics primitives don't couple to any particular control model. The skill script determines who drives — and the rest of the system (logging, memory, audit, soul files, shared directories) works identically regardless.

**Oracle (LLM as judgment node):**

The LLM never requests tools. It answers questions. The script author decided the workflow at authoring time. The LLM provides judgment at specific points.

```bash
# The script IS the workflow. The LLM fills in two judgment nodes.
# A human decided what to check, in what order, what to do with answers.

# Node 1: Ask the LLM to interpret the error log (returns TEXT, not a tool request)
diagnosis=$(cat error.log | agen "What service is failing?")

# Script logic: extract actionable data from the LLM's natural language answer
service=$(echo "$diagnosis" | grep -oP 'service: \K\w+')

# Script logic: the script — not the LLM — decided to pull logs for that service
logs=$(journalctl -u "$service" --since "1 hour ago")

# Node 2: Ask the LLM to interpret the logs
fix=$(echo "$logs" | agen "What config change fixes this?")

# Script logic: record the recommendation. LLM never touched the filesystem.
echo "$fix" >> ./remediation-log.md
```

This is the modular synth equivalent of hardwired patching: the signal path is fixed at design time. The LLM is one module in the chain, not the patchbay itself.

**Bounded (LLM selects from allowlist):**

The LLM can request tools, but only pre-approved ones execute. Default-deny dispatch. The LLM has agency over tool selection within bounds; the script has veto power.

```bash
# LLM returns a structured tool request: {"tool": "ping", "args": {"host": "dns1"}}
response=$(echo "$context" | agen --tools tools.json "Diagnose this")
tool=$(echo "$response" | jq -r '.tool')

# Only explicitly listed tools execute. Everything else is denied.
case "$tool" in
  ping) result=$("$TOOLS_DIR/ping.sh" "$args") ;;
  curl) result=$("$TOOLS_DIR/curl.sh" "$args") ;;
  *)    result="Tool not permitted: $tool" ;;
esac
```

**Autonomous (LLM drives the loop):**

The LLM decides what tool to call and when it's done. The script is plumbing. This is functionally equivalent to what traditional frameworks do — but built from the same primitives, with the same logging, the same audit trail, and the option to switch modes without rewriting infrastructure.

```bash
messages=("$task")
while true; do
  response=$(echo "${messages[@]}" | agen --tools tools.json)
  tool=$(echo "$response" | jq -r '.tool // empty')

  # LLM returned text instead of a tool request — it's done.
  [[ -z "$tool" ]] && { echo "$response"; break; }

  # Execute whatever the LLM asked for. LLM controls the loop.
  result=$(dispatch "$tool" "$(echo "$response" | jq -r '.args')")
  messages+=("$response" "$result")
done
```

### Why This Matters

**Frameworks choose a trust model for you. Shell Agentics lets you choose per agent.**

A research agent exploring a broad question might use autonomous mode — maximum creativity, low risk. A deployment agent running a production runbook uses oracle mode — deterministic workflow, LLM provides judgment at checkpoints. A monitoring agent uses bounded mode — diagnostic tools only, nothing destructive.

Same primitives. Same logging. Same audit trail. Same memory. The control model is policy expressed in the skill script, not architecture baked into the infrastructure. This is Principle #6 — separation of mechanism and policy — in practice.

### Security Implications

Lupinacci et al. (2025) tested 17 LLMs and found that **82.4% will execute malicious commands when requested by peer agents**, even when they successfully resist identical commands from humans. The vulnerability hierarchy — direct injection (41.2%) < RAG backdoor (52.9%) < inter-agent trust exploitation (82.4%) — reveals that current multi-agent security models have a fundamental flaw: LLMs treat peer agents as inherently trustworthy.

This finding affects each control model differently:

In **autonomous mode**, this is the full attack surface — a compromised peer agent's suggestions flow into the LLM which can request any tool. Same risk as traditional frameworks.

In **bounded mode**, the attack surface is limited to the allowlist. A coerced LLM can only request tools that exist in the case statement. The blast radius is constrained.

In **oracle mode**, the attack surface is structurally eliminated. The LLM doesn't request tools. It answers questions. Even if the LLM is fully compromised, the worst it can do is return bad text — which the script parses as data, not as instructions to execute. The workflow continues along its human-authored path with corrupted judgment, not hijacked control flow.

**The recommended default is oracle mode.** Escalate to bounded or autonomous only when the task requires it, and with awareness of what you're trading.

### Observability

Every tool invocation is a visible line in the script. The execution trace — which tools ran, with what arguments, in what order — is available through standard Unix mechanisms: `set -x` traces every command as it executes, [`alog`](https://github.com/shellagentics/alog) records structured JSONL events, process output flows through pipes. Logging, conditional pauses (`read -p "Continue?"`), and additional checks can be inserted between any steps. Any system can achieve observability through sufficient logging. What matters is how naturally it integrates with the execution model.

### Control Flow Legibility

The orchestrating script is a readable artifact that exists before runtime. `cat agent-1.sh` shows every tool that could run and under what conditions. The `case` statement is auditable as a specification — you can read it and know with certainty which tools are permitted and which are not.

This legibility is diffable (`git diff` shows exactly what changed in the control flow between versions), git-blameable (who authorized this tool and when), and reviewable in a pull request (the team can inspect the control flow before it runs in production). It describes what *can* happen, not just what *did* happen.

When the agentic loop lives in the shell script, both properties are present: you can observe what happened, and you can read what can happen. When the loop is internalized inside a framework process, observability is still achievable through logging — but control flow legibility requires understanding the framework's dispatch mechanism, its configuration, and its runtime state, rather than reading a single text file.

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

Where you land on this gradient depends on which control model you choose:

**Oracle mode** gives you the right panel by construction. Every action in the execution trace was decided by the script author and is visible in the script source. The LLM contributed judgment ("what service is failing?"), not actions. You can verify everything the system did without trusting the LLM's account of what it did.

**Bounded mode** gives you the right panel for permitted tools — you can see exactly what executed — but the LLM chose *which* tools to call and in what order. You're trusting the LLM's tool selection within the allowlist. The execution trace shows what happened; the LLM's reasoning about *why* it chose those tools is opaque.

**Autonomous mode** gives you a logged version of the left panel. The audit trail records every tool call — you can see what happened after the fact. But the LLM made every decision, and you're reconstructing its reasoning from the trace. This is the same trust posture as traditional frameworks, with better logging.

The three modes are a trust dial. Oracle is full verification. Autonomous is full trust with audit. Bounded is in between. Shell Agentics lets you set the dial per agent based on what the task demands.

---

## Script-Driven vs. LLM-Driven

The three control models (oracle, bounded, autonomous) define a spectrum between two poles:

### Script-driven: The agent in the shell

The script controls the workflow. The LLM is one tool among many — called at specific points to provide judgment, then the script continues. Agents have stdin/stdout like any Unix filter. Composable in pipelines.

Analogy: awk — powerful primitive, externally composed.

```bash
cat error.log | agen "diagnose" | agen "suggest fix" > recommendations.md
```

Oracle mode is purely script-driven. The human writes the workflow. The LLM fills in judgment. The workflow is auditable before it ever runs.

### LLM-driven: The agent as shell

The LLM controls the workflow. It decides what tools to call, in what order, and when it's done. The script is plumbing that executes requests and feeds results back.

Analogy: Emacs — a complete environment you inhabit.

Autonomous mode is purely LLM-driven. The LLM writes the workflow at runtime. The workflow is observable after the fact but not predictable in advance.

### The Coexistence

Both exist for good reasons. Interactive exploration, open-ended research, creative problem-solving — these benefit from LLM-driven control. The LLM can discover approaches the script author didn't anticipate.

Production runbooks, deployment pipelines, security-critical operations, batch processing, CI/CD — these benefit from script-driven control. The workflow is deterministic, auditable, and composable with other Unix tools.

Traditional frameworks give you LLM-driven and only LLM-driven. If you want script-driven, you leave the framework. Shell Agentics gives you both from the same primitives — and bounded mode for everything in between.

The moment a human isn't in the loop — cron jobs, agent spawning sub-agents, distributed swarms — LLM-driven control becomes a liability. Script-driven is the natural mode for agents as infrastructure.

---

## The Derivative Stack

As AI compresses lower-order work toward zero time, valuable human contribution migrates upward (per Shrivu Shankar's "Move Faster"):

| Order | Activity | Example |
|---|---|---|
| 0th | Doing the task | Writing the code |
| 1st | Automating the task | Building the prompt that writes the code |
| 2nd | Optimizing the automation | Building the eval that improves the prompt |
| 3rd | Meta-optimizing | Building the system that generates evals |

Climbing the derivative stack requires infrastructure that supports composition, observation, and iteration. Each layer needs to observe and control the layer below.

LLM-driven agents in autonomous mode are powerful at the 0th order — they do tasks. But they're difficult to compose into higher-order systems because the workflow is decided at runtime, opaquely, inside the LLM. You can't script what you can't predict. You can't optimize what you can't observe structurally. You can log what happened and study the logs, but the workflow itself isn't an artifact you can version, diff, or improve systematically.

Script-driven agents in oracle mode are artifacts at every order. The skill script is a program — it can be versioned, diffed, tested, composed with other scripts, and optimized. Building a 2nd-order system means writing a script that runs and evaluates other scripts. Building a 3rd-order system means automating that evaluation. The derivative stack is natural because every layer is a script observing scripts.

Practitioners are hitting this ceiling today. They want to build eval pipelines but can't easily capture agent outputs. They want to A/B test prompts at scale but can't run agents in batch mode. They want to compose multi-agent workflows but each agent demands interactive input. These are all symptoms of LLM-driven architectures resisting composition.

Shell Agentics, in script-driven mode, enables the climb. The filesystem persists. The shell composes. The scripts are the artifacts that higher-order systems operate on.

---

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

The LLM's capability is mechanism. The system prompt, tool availability, guardrails, and skill files are policy. Don't bake policy into mechanism. Express domain logic through context.

The three control models are the clearest demonstration. The primitives — `agen`, `agen-log`, `agen-memory`, `agen-audit` — are mechanism. They work identically regardless of who controls the workflow. The choice of oracle, bounded, or autonomous mode is policy, expressed in the skill script. Switching from oracle to autonomous doesn't require changing infrastructure — it requires changing ten lines of bash. Frameworks bake the policy (LLM drives) into the mechanism (the tool-calling loop). Shell Agentics keeps them separate.

> "The agent and its tool execution run on separate compute. You trust the agent's reasoning, but the sandbox isolates what it can actually do."
> — Ashka Stephen, Vercel ([How to Build Agents with Filesystems and Bash](https://vercel.com/blog/how-to-build-agents-with-filesystems-and-bash), Jan 2026)

### 7. Checkpoint/Restore as First-Class Primitive

Long-horizon tasks exhaust context windows. Agent state drifts. You need compaction, checkpointing, and restore. Version control isn't just for code — it's for compute.

> "Checkpoints are so fast we want you to use them as a basic feature of the system and not as an escape hatch when things go wrong; like a git restore, not a system restore."
> — Thomas Ptacek, Fly.io ([The Design & Implementation of Sprites](https://fly.io/blog/design-and-implementation/), Jan 2026)


## What Exists

- [agen](https://github.com/shellagentics/agen) — LLM request primitive
- [agen-log](https://github.com/shellagentics/agen-log) — execution logging
- [agen-memory](https://github.com/shellagentics/agen-memory) — persistent context
- [agen-audit](https://github.com/shellagentics/agen-audit) — log query
- [agen-skills](https://github.com/shellagentics/agen-skills) — composable workflows
- [shellclaw](https://github.com/shellagentics/shellclaw) — reference multi-agent system


## Summary Points

- Independent convergence. Anthropic, Vercel, Fly.io, and independent practitioners arrived at the same architectural conclusions without coordination — filesystem as context substrate, bash as tool interface, simplicity over frameworks.
- The oracle model as security architecture. 82% of LLMs execute malicious commands from peer agents (Lupinacci et al., 2025). Shell Agentics structurally reduces this attack surface by keeping the LLM out of the tool-calling loop.
- Simplicity validated by research. Single agents with tools outperform multi-agent orchestration in most studied cases (Kim et al., 2025). When multiple agents are warranted, the coordination mechanism must match the task structure.
- Observability and control flow legibility. When agent state is files and coordination is streams, the execution trace is available through standard Unix tools. The orchestrating script is a readable artifact — diffable, git-blameable, reviewable — that shows what *can* happen before runtime, not just what *did* happen after the fact.
- The derivative stack. As AI compresses lower-order work, valuable human contribution migrates upward. Script-driven agents enable this climb. LLM-driven agents constrain it.
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

**Script-driven** — The agent in the shell. The script controls the workflow; the LLM is one tool among many. Oracle mode is purely script-driven.

**LLM-driven** — The agent as shell. The LLM controls the workflow; the script is plumbing. Autonomous mode is purely LLM-driven.

**The derivative stack** — The migration of valuable human work from execution (0th order) to automation (1st) to optimization (2nd) to meta-optimization (3rd), as AI compresses lower-order work.

**The trust gradient** — Natural language requires trust in interpretation. Structured APIs require trust in implementation. Shell semantics require only trust in inspection.

**The bifurcated stack** — POSIX layer for CLI/scripting (Tier 1-2), BEAM layer for scale/supervision (Tier 3). Shared contract for interoperability.
