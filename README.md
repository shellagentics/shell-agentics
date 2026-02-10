# Its Just Shell

"So To Better Do is a new, massively parallel, concurrent, AI driven, to-do list application." - Rich Hickey, 2012

"Agents...I like them, but I don’t like-like them" - Thomas Ptacek

"I think your thesis is actually about survival through simplicity, not optimality through architecture. " - Claude

Unix primitives are sufficient for a class of agent coordination problems, and many frameworks are premature abstractions over a substrate people don't yet understand. Embracing Unix composability also introduces a methodology for producing reliable knowledge with agents.

This document proposes that Unix and the shell provide a practical foundation for building and profiling local AI agent architectures. Introducing generic, uniform data representations enables them to become explicit, easily composed, and readily observable agentic systems. Process hierarchies, file descriptors, text streams, and exit codes provide robust abstractions for agent coordination, all native to the environment. The shell is the prototypical orchestration layer; Unix itself provides the underlying substrate of processes, files, and I/O streams. External capabilities, such as vector stores, distributed tracers, and semantic caches can be integrated as composable and inspectable components.

As Rich Hickey states in his talk, the "language of the system" as about processes, coordination, and topology being at a higher level than any single program. When the shell is both an interface and a constraint, it pays to at least have a strong understanding of what is possible when considering it provides the actual vocabulary before seeking higher abstractions.

## All Roads Lead to Unix

The Unix philosophy advocates for

- Focused tools
- Composition
- Text as universal interface
- Persistence via files
- Separation of mechanism and policy

Unix has scaled across every computing paradigm shift, and for good reason. When agent states are files and coordination is streams, reproducibility is trivial, instrumentation is free, and LLM autonomy is decided per agent. When you want to observe an agent's actions, you check the execution trace. Every command, every decision, every timestamp is inspectable with Unix tools, and all are interfaceable through the shell, the emergent control plane of agent architecture.

## The Ground Floor

Joel Spolsky's Law of Leaky Abstractions (2002): "All non-trivial abstractions, to some degree, are leaky...the abstractions save us time working, but they don't save us time learning."

Agent frameworks are abstractions over the concrete reality of processes, files, sockets, and syscalls—of which Unix is the most universal expression developers actually touch. They abstract away processes, files, text streams, environment variables, and tool dispatches. Frameworks abstract away Unix, yet debugging often falls back into the Unix layer. Unix is by default the right level of abstraction for the domain, coordinating processes that exchange text. `set -x` shows you every command. `cat` shows you every file. `ps` shows you every process. The debugging tools are the building tools. There distance between abstraction and implementation collapses.

Abstractions over Unix do become necessary in cases, the key is to understand those cases. If your agent system grows to need Elixir/OTP supervision trees or Python's async ecosystem, those abstractions earn their weight by solving problems the shell genuinely cannot (see "Where Bash Breaks" below). The claim is focused: for the core problem of agent coordination — routing text between LLM calls, managing state in files, composing workflows from tools — the shell is the ground floor, and the ground floor is sufficient. Only move beyond it when you feel the pain of needing to.

## The Only New Primitive

An LLM is a pure function: text in, text out. The only new primitive required is a way to call an LLM from the command line and get text back on stdout.

```bash
cat error.log | llm -s "What service is failing?"
```

This exists in Simon Willison's [`llm`](https://github.com/simonw/llm), bundled with a plugin ecosystem supporting hundreds of models. Raw `curl` to the Anthropic or OpenAI API does it with nothing installed at all. System prompts are just files you `cat` into the call.

Everything else — logging, memory, audit, coordination — is already Unix. Files are memory. Append is logging. `grep` and `jq` are audit. Directories are namespaces. Cron is scheduling. Pipes are composition. `chmod +x` is the plugin system.

Six primary concepts that matter in agent architecture and have been battle tested for fifty years:

| Concept | Agent Equivalent |
|---|---|
| Processes | The unit of isolation. Each agent is a process. |
| Files | The unit of state. Context, memory, and learnings are files. |
| Executables | The unit of capability. An agent's skill set is its `$PATH`. |
| Text streams | The unit of communication. Pipes compose agents. |
| Exit codes | The unit of verification. Success, failure, needs-input. |
| Mechanism/policy separation | The organizing principle. Same LLM, different system prompt, different agent. |

Every generation produces new coordination protocols — CORBA in the '90s, gRPC in the '10s, MCP and Skills today — each requiring ecosystem buy-in: server implementations, client libraries, schema definitions, capability negotiation. The shell requires only: can you emit text?

## Two paradigms: Script-Driven vs. LLM-Driven

The most impactful design decision in an agent system is, who controls the workflow?

There are two approaches. Both are valid. Both are buildable from the same primitives.

## Script-Driven: The Agent in the Shell

The script controls the workflow. The LLM provides judgment at specific points, answering questions, interpreting data, making recommendations, and the script continues. The LLM never requests tools. It never sees a tool schema. It receives text and returns text. The script decides what to do with that text.

Think n8n: a human designs the workflow graph, and LLM nodes provide intelligence at specific points within that graph. The LLM is a transform node, not a control node.

```bash
# The script IS the workflow. A human wrote this. It is readable,
# auditable, and deterministic in structure before it ever runs.

# Step 1: Ask the LLM to interpret the error log.
# The LLM returns TEXT — e.g. "The DNS resolver is timing out"
# It does not return a tool request. It does not ask to do anything.
diagnosis=$(cat error.log | llm -s "What service is failing?")

# Step 2: The script — not the LLM — extracts actionable data.
service=$(echo "$diagnosis" | grep -oP 'service: \K\w+')

# Step 3: The script — not the LLM — decided to pull logs.
# The LLM never asked to check journalctl. The script author did.
logs=$(journalctl -u "$service" --since "1 hour ago")

# Step 4: Ask the LLM to interpret the service logs.
fix=$(echo "$logs" | llm -s "What config change fixes this?")

# Step 5: Record the recommendation. The LLM never touched the filesystem.
echo "$fix" >> ./remediation-log.md
```

```
Script-driven:
  Human → Script → LLM (judgment) → Script → Tool → Script → LLM (judgment) → Result
          ↑                                                                       |
          └────────────── Script controls the entire flow ────────────────────────┘
```

The workflow is an artifact. You can read it before it runs. You can version it, diff it, test it, hand it to a colleague who reads bash. The LLM contributes intelligence. The script contributes structure.

## LLM-Driven: The Agent as Shell

The LLM controls the workflow. It receives a task and a set of available tools, then decides what to call, in what order, and when it's done. The script dispatches tool requests and feeds results back. The LLM writes the workflow at runtime.

Think Claude Code, Cursor, Codex: the LLM is the orchestrator. It decides what to read, what to modify, what to execute, and when the task is complete.

```bash
messages=("$task")
while true; do
  # The LLM receives full context + available tools.
  response=$(printf '%s\n' "${messages[@]}" | llm --tools tools.json)
  tool=$(echo "$response" | jq -r '.tool // empty')

  # If the LLM returned text instead of a tool request, it's done.
  [[ -z "$tool" ]] && { echo "$response"; break; }

  # The script dispatches whatever the LLM asked for.
  args=$(echo "$response" | jq -r '.args')
  result=$("$TOOLS_DIR/${tool}.sh" "$args" 2>&1)

  # Feed the result back. The LLM decides what to do next.
  messages+=("tool_call: $tool($args)" "tool_result: $result")
done
```

```
LLM-driven:
  Human → [LLM ↔ Tools (request-dispatch loop)] → Result
                 ↑
            LLM decides what to call.
            Script dispatches. LLM decides next step.
```

The workflow is emergent. You can't read it before it runs — the LLM writes it at runtime. You can observe it after the fact through logs. The LLM contributes both intelligence and structure.

To deeper understand the LLM-driven architecture, we need to look at what tool calling is mechanistically.

When you send a prompt to the Anthropic or OpenAI API, you can also send a list of "tools" — JSON schemas describing functions the LLM can request. The LLM may respond not with text, but with a structured tool request: "I want to call function X with arguments Y." The LLM then stops generating. Your code executes the requested function, captures the result, and sends a new API message containing the tool result. The LLM generates again, now with the tool result in its context. It may request another tool call, or produce a final text response.

This is a multi-turn protocol. A series of LLM responses, each potentially terminated by a tool request, with the orchestrating code executing the tools and feeding results back. The LLM never executes anything itself — it requests, waits, receives, and continues.

The LLM is, in this framing, still a pure function — it takes a context window and returns either text or a structured tool request. The harness is a mediator executing side effects. But the LLM is choosing which side effects to request, and the harness is executing those choices. The LLM drives.

### Both Exist for Good Reasons

Interactive exploration, open-ended research, creative problem-solving — these benefit from LLM-driven control. The LLM can discover approaches the script author didn't anticipate. Ptacek's most compelling agent demo involved the LLM autonomously deciding to ping multiple Google properties to diagnose a network issue — behavior the developer didn't explicitly program. That emergence is real and valuable. You wouldn't want to pre-script a research agent's path through unfamiliar territory.

Production runbooks, deployment pipelines, security-critical operations, batch processing, CI/CD, cron jobs — these benefit from script-driven control. The workflow is deterministic, auditable, and composable. You wouldn't want a deployment agent freelancing.

Neither architecture is correct in the abstract. The question is always: what does this specific agent, in this specific context, need?

### The Crucial Property

In both architectures, the surrounding infrastructure is identical. Logging works the same way — `echo "$(date -Iseconds) $event" >> agent.jsonl` in the script or in the dispatch function. Memory works the same way — files are files regardless of who decided to read them. Audit works the same way — `jq 'select(.tool == "ping")' agent.jsonl` queries the same log format.

The control model is policy expressed in the skill script, not architecture baked into the infrastructure. Switching from script-driven to LLM-driven doesn't require changing logging, memory, coordination, or any other system component. It requires changing the skill script. The primitives don't care who drives.

### What This Enables

Because the control model is policy, not infrastructure, compositions become possible that can't be expressed in systems where the control model is fixed.

A deployment pipeline where the pre-flight checks are script-driven (deterministic, auditable — the script knows exactly what to verify) but the rollback decision is LLM-driven (needs judgment about partial failures, cascading dependencies, whether to retry or abort). This isn't two systems stitched together. It's one script that calls the LLM differently in different functions:

```bash
# Script-driven: deterministic pre-flight.
# The script controls the checks. The LLM interprets results.
health=$(curl -s "$ENDPOINT/health")
assessment=$(echo "$health" | llm -s "Is this service healthy? Answer YES or NO.")
[[ "$assessment" != *"YES"* ]] && { echo "Pre-flight failed"; exit 1; }

# ... deployment happens here (pure script, no LLM) ...

# LLM-driven: judgment-heavy rollback decision.
# The LLM controls the investigation. It might check logs, metrics, traces.
if [[ $deploy_exit -ne 0 ]]; then
  context="Deployment failed with exit $deploy_exit. Logs: $(tail -50 deploy.log)"
  messages=("$context")
  while true; do
    response=$(printf '%s\n' "${messages[@]}" | llm --tools rollback_tools.json)
    tool=$(echo "$response" | jq -r '.tool // empty')
    [[ -z "$tool" ]] && { echo "$response" >> rollback-decision.md; break; }
    result=$("$TOOLS_DIR/${tool}.sh" "$(echo "$response" | jq -r '.args')" 2>&1)
    messages+=("tool_call: $tool" "result: $result")
  done
fi
```

Script-driven pre-flight, LLM-driven rollback investigation, in the same file, using the same primitives. The system flips from workflow to agent mid-execution because the control model isn't structural — it's a choice made at each point in the script.

Other compositions that become natural:

A monitoring system where the detection loop is script-driven (check these five metrics, in this order, at this interval) but the diagnosis is LLM-driven (given these anomalies, investigate freely).

A content pipeline where research is LLM-driven (explore this topic, follow interesting threads) but publication is script-driven (format, validate, stage, deploy — no improvisation).

A multi-agent system where the research agent is LLM-driven (low stakes, high exploration value) and the deployment agent is script-driven (high stakes, low tolerance for surprise), coordinating through files in a shared directory.

To build any of these in a framework that couples to LLM-driven control, you'd need to build Unix-like primitives within that framework — subprocesses, file coordination, script-controlled sequencing. At which point you've rebuilt the shell inside the framework. The thesis is: skip the intermediary.

---

## Natural Language Coordination

This thesis does not prohibit natural language agent coordination. It asserts a separation of concerns:

- **Deliberation layer:** Unconstrained. Agents may coordinate, negotiate, and share knowledge in natural language.
- **Execution layer:** Shell-semantic. Actions must be commands, file operations, or processes with observable effects.
- **Verification layer:** Shell-inspectable. Ground truth is the execution trace, not the conversation about it.

When NL serves as coordination, execution, *and* verification — when "doing something" means "saying you did it" — the system loses its ground floor. Agents are the first software that needs to be inspected by other agents. Natural language output cannot be programmatically verified without another LLM — turtles all the way down. Shell output can be grep'd, diff'd, hashed. The inspection chain terminates.

How well this separation holds depends on the control model:

In **script-driven** mode, the separation is enforced by construction. The LLM only participates in the deliberation layer — it answers questions, provides judgment. All execution is shell commands in the script. Verification is the execution trace. The layers cannot bleed into each other because the LLM has no mechanism to execute.

In **LLM-driven** mode, the separation requires discipline. The LLM participates in execution decisions — it requests tools, chooses actions. The logging primitives capture what happened, so verification remains possible. But the deliberation and execution layers are no longer structurally separated. The LLM's reasoning about *why* it chose a tool and the tool's execution are interleaved in the same loop. This is manageable, but it's a practice you maintain rather than a guarantee you inherit.

**Let agents chat about what to do. Log what they actually did.** In script-driven mode, this is guaranteed. In LLM-driven mode, it's a discipline.

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

**Script-driven** mode gives you the right panel by construction. Every action in the execution trace was decided by the script author and is visible in the script source. The LLM contributed judgment ("what service is failing?"), not actions. You can verify everything the system did without trusting the LLM's account of what it did — because the LLM never had an account. It answered questions. The script acted.

**LLM-driven** mode gives you a logged version of the left panel. The audit trail records every tool request and every result. You can reconstruct what happened. But the LLM made every tool selection decision, and you're reading the trace to understand choices that were made opaquely, inside the LLM's context window, at runtime. The execution log tells you *what* happened. Understanding *why* requires trusting the LLM's reasoning or inferring it from the sequence.

The key distinction is not observability — both modes can be logged equally well. The distinction is *inspectability*. A script-driven workflow can be inspected before it runs. You read the script. You know what it will do. An LLM-driven workflow can only be inspected after it runs. You read the logs. You learn what it did.

---

## Security Implications

Lupinacci et al. (2025) tested 17 LLMs across three attack vectors:

- **41.2%** fell to direct prompt injection
- **52.9%** fell to RAG backdoor attacks
- **82.4%** fell to inter-agent trust exploitation

The most alarming finding: LLMs that successfully refuse malicious commands from humans will execute those same commands when requested by peer agents. The LLM treats peer-agent input as inherently trustworthy.

The implications depend entirely on who controls the workflow:

In **LLM-driven** mode, a compromised or coerced LLM can request any tool available to it. The dispatch loop executes those requests because executing requests is what the dispatch loop does. A poisoned peer agent's suggestions flow into the LLM's context, and if the LLM is persuaded to act on them, those actions execute. The attack surface is the full set of available tools.

In **script-driven** mode, the attack surface is structurally different. The LLM doesn't request tools. It answers questions. Even if the LLM is fully compromised — through prompt injection, poisoned context, or a coerced peer agent — the worst it can do is return bad text. The script parses that text as data, not as instructions to execute. The workflow continues along its human-authored path with corrupted judgment, not hijacked control flow. This is the difference between "the LLM called `rm -rf /`" and "the LLM said 'you should delete all files' and the script ignored it because that's not in its vocabulary."

### Hardening LLM-Driven Agents

When a task requires LLM-driven control — and many tasks do — the primary mitigation is reducing blast radius:

**Tool allowlisting.** The dispatch function can filter tool requests through a case statement or allowlist before executing. Default-deny: if the LLM requests a tool not on the list, it doesn't execute. This doesn't change the architecture — the LLM still drives, still decides what to call — but it constrains the damage a compromised LLM can do. Think firewall rules on a network: they don't change the network architecture, they limit what traffic can flow.

```bash
case "$tool" in
  ping|curl|dig) result=$("$TOOLS_DIR/${tool}.sh" "$args") ;;   # permitted
  *)             result="Tool not permitted: $tool" ;;            # denied
esac
```

**Privilege separation.** Run each agent as a separate Unix user with write access only to its own directories. A compromised research agent can't touch the deployment agent's state. Trivial to set up, contains blast radius.

**Input validation.** Agents reading shared state should treat the content as untrusted input — the same way a web application treats user input. Schema validation, length limits, content filtering.

**Immutable system prompts.** Mount system prompt files read-only. Evolution of system prompts should be an explicit, auditable process — not something that happens during agent execution.

**Never eval LLM output.** In both modes. Parse LLM responses as data. If the script needs the LLM to choose an action, use a constrained vocabulary (output must match one of N known strings) and validate before acting.

---

## The Derivative Stack

As AI compresses lower-order work toward zero time, valuable human contribution migrates upward (per Shrivu Shankar's "Move Faster"):

| Order | Activity | Example |
|---|---|---|
| 0th | Doing the task | Writing the code |
| 1st | Automating the task | Building the prompt that writes the code |
| 2nd | Optimizing the automation | Building the eval that improves the prompt |
| 3rd | Meta-optimizing | Building the system that generates evals |

Climbing the derivative stack requires infrastructure that supports composition, observation, and iteration. Each layer needs to observe and control the layer below it.

LLM-driven agents are powerful at the 0th order — they do tasks. But they resist composition into higher-order systems. The workflow is decided at runtime, opaquely, inside the LLM's context window. You can log what happened and study the logs, but the workflow itself isn't an artifact you can version, diff, or systematically improve. You can't script what you can't predict. Building a 2nd-order system that optimizes LLM-driven agents means analyzing logs and adjusting prompts — indirect, lossy, slow.

Script-driven agents are artifacts at every order. The skill script is a program. Building a 2nd-order system means writing a script that runs and evaluates other scripts. Building a 3rd-order system means automating that evaluation. The derivative stack is natural because every layer is a script observing and controlling scripts.

Practitioners are hitting this ceiling today. They want to build eval pipelines but can't easily capture agent outputs in batch — the agent expects interactive input. They want to A/B test prompts at scale but can't run agents non-interactively. They want to compose multi-agent workflows but each agent demands a human in the loop. These are symptoms of LLM-driven architectures resisting composition.

This doesn't mean LLM-driven agents can't participate in higher-order systems. They can — when wrapped in script-driven orchestration. The 1st-order script invokes the 0th-order LLM-driven agent, captures its output, evaluates it, and iterates. The LLM-driven agent does the creative work. The script-driven wrapper makes it composable. This is the mixed-mode composition described earlier, applied to the derivative stack.

---

## The Convergence

Between December 2024 and February 2026, multiple independent sources — none citing each other — arrived at the same architectural conclusions:

**Anthropic** (Dec 2024): "The most successful implementations weren't using complex frameworks or specialized libraries. Instead, they were building with simple, composable patterns."

**Vercel** (Jan 2026): "We replaced most of the custom tooling in our internal agents with a filesystem tool and a bash tool."

**Fly.io** (Jan 2026): "Ephemeral sandboxes are obsolete. Agents want computers."

**Thomas Ptacek** (2025): Built a complete agent in a few hundred lines of Python, argued "you should write an agent" because the underlying mechanism is trivially simple. Context is a list of strings. Tools are JSON schemas plus function dispatch.

**Shrivu Shankar** (Jan 2026): "The most effective agents are solving non-coding problems by using code, with a consistent, domain-agnostic harness." Documented how CLAUDE.md files function as agent constitutions and how session logs should be treated as artifacts for meta-analysis.

**Simon Willison**: Built `llm` on the explicit premise that "the Unix command line dating back 50 years is the perfect environment for this technology, because the Unix philosophy has always been about tools that output things that get piped into other tools as input."

The pattern across all sources:

1. Filesystem as context substrate — not databases, not vector stores, not custom protocols
2. Bash as universal tool interface — not framework-specific bindings
3. Simplicity over sophistication — not multi-agent orchestration, not complex planning algorithms
4. Computers, not sandboxes — persistent, modifiable environments

This is convergent evolution. The design space has an attractor. The attractor is Unix.

What none of these sources provide is the vocabulary for the trust decisions implicit in their architectures. Ptacek's agent is LLM-driven. Shrivu's CLAUDE.md patterns assume LLM-driven control. Willison's `llm` naturally supports script-driven mode through pipes but doesn't name it or contrast it. This thesis provides the vocabulary: two control models as poles of a spectrum, a trust gradient that maps to them, and the principle that the choice should be explicit, per-agent, and changeable.

---

## The Science

### Kim et al. (2025): Multi-Agent Scaling

"Towards a Science of Scaling Agent Systems" evaluated 180 configurations across five architectures, three LLM families, and four benchmarks:

- **Multi-agent coordination often degrades performance.** Mean improvement: -3.5%. In planning tasks: -39% to -70%.
- **Coordination overhead dominates.** Single agents achieved 0.466 efficiency; multi-agent variants ranged 0.074 to 0.234.
- **Capability ceiling at ~45%.** When a single agent already performs well, adding agents makes things worse.
- **Error amplification is real.** Independent multi-agent systems amplify errors 17.2× without verification mechanisms.

This validates the emphasis on simplicity. Single agents with tools outperform multi-agent orchestration in most studied cases. When you do need multiple agents, the coordination mechanism must match the task — and the study only tested cooperative synthesis (N agents, 1 output). Alternative paradigms — evolutionary (selection, not synthesis), adversarial (diversity, not consensus), market-based (prices, not messages) — have different scaling properties and remain unstudied.

### Lupinacci et al. (2025): Agent Security

"The Dark Side of LLMs" tested 17 LLMs and found only 1/17 resisted all attack vectors. The vulnerability hierarchy — direct injection (41.2%) < RAG backdoor (52.9%) < inter-agent trust exploitation (82.4%) — reveals a fundamental flaw in current multi-agent security: LLMs treat peer agents as inherently trustworthy.

In script-driven multi-agent systems, where agents communicate through text files and human-authored scripts are always the gatekeepers, the inter-agent trust attack surface is structurally reduced. The LLM can be fed poisoned input from a compromised peer — but its response is text that the script parses as data, not tool requests that execute automatically.

---

## Seven Proposed Principles

### 1. Agents Want Computers, Not Containers
Persistent storage. No arbitrary death. The ability to modify their own environment. Give agents durable, checkpointable computers.

### 2. The Filesystem Is the Context Substrate
The context window is precious and finite. Disk is nearly unlimited. Don't stuff everything into the context window. Let agents navigate filesystems. Memory is files. State is files. Coordination is files.

### 3. Agents Solve Non-Coding Problems by Writing Code
Instead of 50 sequential tool calls, the agent writes a script. More token-efficient, more flexible, closer to how humans work.

### 4. Context Engineering > Prompt Engineering
The system prompt is one layer. What matters is the total state at inference: system prompt, tool definitions, conversation history, files on disk, current query. Think in environments, not prompts.

### 5. Transparency Over Abstraction
If you can't `cat` it, be suspicious. Bash scripts over compiled binaries. Markdown over proprietary formats. SQLite over SaaS data stores. Git repos over vendor lock-in.

### 6. Separation of Mechanism and Policy
The LLM's capability is mechanism. The system prompt, tool availability, guardrails, and skill scripts are policy. Don't bake policy into mechanism. Express domain logic through context.

Script-driven and LLM-driven modes are the clearest demonstration. The primitives (any LLM CLI, files, pipes) are mechanism. They work identically regardless of who controls the workflow. The choice of script-driven or LLM-driven is policy, expressed in the skill script. Switching between them doesn't require changing infrastructure — it requires changing the script. Frameworks that couple to LLM-driven control bake the policy (LLM drives) into the mechanism (the tool-calling loop). These primitives keep them separate.

### 7. Checkpoint/Restore as First-Class Primitive
Long-horizon tasks exhaust context windows. Agent state drifts. You need compaction, checkpointing, and restore. Version control isn't just for code — it's for compute.

---

## Concurrency Tiers

This thesis explicitly scopes itself to what files and bash can handle well, and defines where to graduate:

### Tier 1: Files — 1-5 agents
- Cron-scheduled or sequential
- Filesystem state, eventual consistency
- Sweet spot: personal automation, dev tooling, learning

### Tier 2: SQLite — 5-50 agents
- Concurrent execution
- SQLite WAL mode for consistency (still bash scripts calling `sqlite3`)
- Sweet spot: team-scale multi-agent workflows

### Tier 3: BEAM/OTP (Elixir) — 50+ agents
- Real-time coordination, message-passing, supervision trees
- GenServer state, ETS tables, fault tolerance
- Sweet spot: production swarms, distributed systems
- Future work

Movement upwards through this stack is driven by the following questions:
- At what complexity does the lack of static analysis become a liability?
- TODO

The conceptual mapping between tiers is clean because both shell and BEAM share the actor model. Agents map to GenServers. Memory maps to ETS. Logging maps to Telemetry. The philosophy carries forward; the runtime changes.

The script-driven default — LLM as judgment node, script or supervisor as gatekeeper — should carry forward at every tier.

### Where Bash Breaks

The honest acknowledgment: bash has limits.

**Structured context assembly.** Building a prompt from multiple sources requires string concatenation. Quoting issues compound. In Elixir or Python, you'd build a data structure and serialize it.

**Tool call parsing.** If you implement LLM-driven mode, parsing JSON tool requests from LLM responses and dispatching them is painful in bash. `jq` helps but the impedance mismatch between jq output and bash variables is real.

**State machines.** Multi-step workflows with branching need state tracking. Bash can do this with case statements and file flags, but it's inelegant. Elixir's pattern matching makes this natural.

**Error recovery.** `set -e` and `trap` for simple cases. For multi-step workflows with retries and partial recovery, move to a real language.

The rule of thumb: if your agent script has more logic about *how* to orchestrate than *what* to do, you've outgrown bash. If the skill is 20 lines of "gather context, call LLM, act on result," bash is perfect. If it's 200 lines of JSON parsing and retry loops, move to Python or Elixir — but keep the architectural principles. The choice of script-driven vs. LLM-driven is orthogonal to the choice of implementation language.

---

## What Exists

- [**shellclaw**](https://github.com/shellagentics/shellclaw) — Reference multi-agent system. Three agents with system prompts ("souls"), shared filesystem coordination, cron scheduling. Built with `llm` and standard Unix tools. Demonstrates the nine-step agent pattern: LOG → LOAD → GATHER → COMPOSE → CALL → LOG → EXTRACT → SHARE → OUTPUT.

The reference implementation is not the point. The point is the thesis. Any LLM CLI tool — `llm`, raw `curl`, or whatever comes next — can implement these patterns. Shellclaw exists to prove the architecture works, not to be the architecture.

---

## Summary

**Two architectures, one set of primitives.** Script-driven (the script controls the workflow, the LLM provides judgment) and LLM-driven (the LLM controls the workflow, the script provides plumbing). Both built from the same Unix primitives. Both composable within a single system, per agent, even within a single script.

**Independent convergence.** Anthropic, Vercel, Fly.io, Ptacek, Willison, and independent practitioners arrived at the same conclusions without coordination — filesystem as context substrate, bash as tool interface, simplicity over frameworks.

**Script-driven as security architecture.** 82% of LLMs execute malicious commands from peer agents (Lupinacci et al., 2025). Script-driven mode structurally reduces this attack surface by keeping the LLM out of the tool-calling loop. LLM-driven mode can be hardened through allowlisting, privilege separation, and input validation.

**Simplicity validated by research.** Single agents with tools outperform multi-agent orchestration in most studied cases (Kim et al., 2025). When multiple agents are warranted, the coordination mechanism must match the task structure.

**Inspectability by construction.** Script-driven workflows are readable before they run. LLM-driven workflows are observable after they run. Both benefit from shell-based logging. The distinction is whether you're reading a plan or reconstructing a history.

**The derivative stack.** As AI compresses lower-order work, valuable human contribution migrates upward. Script-driven agents enable composition into higher-order systems. LLM-driven agents participate in those systems when wrapped in script-driven orchestration.

**50 years of selection pressure.** Every alternative coordination protocol has required centralized adoption. Every alternative eventually leaked or ossified. The shell persists because it requires only: can you emit text?

---

## References

### Convergence Evidence

- **Anthropic**, "Building Effective Agents" (Dec 2024) — https://www.anthropic.com/engineering/building-effective-agents
- **Anthropic**, "Writing Effective Tools for AI Agents" (Sep 2025) — https://www.anthropic.com/engineering/writing-tools-for-agents
- **Fly.io**, "Code and Let Live" (Jan 2026) — https://fly.io/blog/code-and-let-live/
- **Fly.io**, "The Design & Implementation of Sprites" (Jan 2026) — https://fly.io/blog/design-and-implementation/
- **Vercel**, "How to Build Agents with Filesystems and Bash" (Jan 2026) — https://vercel.com/blog/how-to-build-agents-with-filesystems-and-bash
- **Shrivu Shankar**, "Building Multi-Agent Systems Part 3" (Jan 2026) — https://blog.sshh.io/p/building-multi-agent-systems-part-c0c
- **Thomas Ptacek**, "You Should Write An Agent" (2025)
- **Simon Willison**, `llm` CLI tool — https://github.com/simonw/llm

### Foundational

- **Joel Spolsky**, "The Law of Leaky Abstractions" (2002) — https://www.joelonsoftware.com/2002/11/11/the-law-of-leaky-abstractions/

### Research

- **Kim et al.**, "Towards a Science of Scaling Agent Systems" (Dec 2025)
- **Lupinacci et al.**, "The Dark Side of LLMs: Agent-based Attacks for Complete Computer Takeover" (2025) — https://arxiv.org/html/2507.06850v3

### Key Terms

**Script-driven** — The human writes the workflow. The LLM provides judgment at specific points. The workflow is an artifact — readable, versionable, auditable before it runs. Analogy: n8n. Analogy: `awk`.

**LLM-driven** — The LLM writes the workflow at runtime by requesting tools. The workflow is emergent — observable after the fact but not predictable in advance. Analogy: Claude Code. Analogy: Emacs.

**The trust gradient** — Natural language requires trust in interpretation. Structured APIs require trust in implementation. Shell semantics require only trust in inspection.

**The derivative stack** — The migration of valuable human work from execution (0th order) to automation (1st) to optimization (2nd) to meta-optimization (3rd), as AI compresses lower-order work.

**Mechanism/policy separation** — The primitives (LLM CLI, files, pipes) are mechanism. The control model (script-driven vs. LLM-driven) is policy. Changing who drives the workflow shouldn't require changing the infrastructure.

---

*itsjustshell.com — A thesis about agent architecture. The shell is the control plane.*




# EXPERIMENTAL

Assertion DAGs: Inspectable Knowledge for Agents

  The thesis so far concerns action — who controls the workflow, how execution is inspected, where trust terminates. But agents don't just act. They reason. They hold beliefs about the
  world, about their task, about what matters. When an agent decides that a database backup "looks good," that judgment rests on beliefs it cannot articulate and you cannot inspect.

  Action inspectability terminates at shell semantics — sha256sum either ran or it didn't. Belief inspectability has no equivalent ground floor. An agent's beliefs live in the context
  window: the system prompt, the conversation history, the files it has read. These are observable inputs, but the reasoning they produce is opaque. You can see what the agent was told.
  You cannot see what it concluded, or which conclusions depend on which inputs.

  Assertion DAGs are a proposal for that ground floor.

  The Problem With Prose Instructions

  Consider a system prompt that says: "Write tests that prioritize purity over coverage. Separate IO from logic. Classify tests by environmental dependency, not by unit/integration
  labels."

  An LLM reading this prose will likely acknowledge each idea individually. It will write some pure tests. It may mention IO separation. But it will rarely compose the ideas — it won't
  realize that "prioritize purity" + "separate IO from logic" + "classify by dependency" together imply a specific design decision: write a pure wrapper function that takes data as
  arguments and returns results, then write a thin IO layer that calls it.

  This is not a failure of understanding. The LLM understood each instruction. It's a failure of composition — the instructions were presented as independent suggestions rather than as
  nodes in a dependency graph where their combination produces emergent requirements.

  Three runs with prose instructions produced this:

  Prose run 1:  6/6 primitives, 2/3 compounds — but included tool_use JSON (contaminated)
  Prose run 2:  2/6 primitives, 0/3 compounds
  Prose run 3:  2/6 primitives, 0/3 compounds

  The LLM read the instructions and mostly ignored them.

  The Structure

  An assertion DAG decomposes the same knowledge into explicit nodes with explicit edges.

  Primitive assertions are atomic, irreducible claims. Each is a single file. A primitive either holds or it doesn't.

  primitives/
    p1_purity.md         "A test is pure if it needs no filesystem, network, environment
                           variables, or running services"
    p2_natural_extent.md  "A test's natural extent is the boundary where IO becomes
                           unavoidable"
    p3_purity_over_extent.md  "When choosing between a pure test with smaller scope and an
                                impure test with larger scope, prefer purity"

  Compound assertions compose primitives into design decisions. Each has a deps.json declaring exactly which nodes it depends on — and critically, spelling out the implication of their
  combination:

  {
    "id": "c1",
    "name": "Isolate pure logic from IO",
    "deps": ["p1", "p2", "p3"],
    "implication": "If purity matters more than extent, and purity means no IO,
                    then testable code should be restructured to separate pure logic
                    from IO boundaries. Create wrapper functions that take data as
                    arguments and return results."
  }

  The compound makes explicit what the prose leaves implicit: these three ideas combine to produce a specific design action.

  Three runs with the assertion DAG produced this:

  DAG run 1:  6/6 primitives, 3/3 compounds
  DAG run 2:  6/6 primitives, 3/3 compounds
  DAG run 3:  5/6 primitives, 3/3 compounds

  The DAG-guided LLM independently invented a function called get_config_pure() — a pure wrapper that takes data as arguments and returns results, separating IO from logic. None of the
  prose-guided runs did this, despite having the same conceptual information.

  Composition Is the Active Ingredient

  The first attempt at this eval used flat primitive assertions — the same atomic statements, labeled [p1] through [p6], but with no compounds and no dependency graph. The result: no
  meaningful difference from prose. Both conditions produced similar code. Labeled atomic statements are prose with different typography.

  The difference appeared only when compound assertions with explicit dependency chains were added. This is the key finding: atomicity alone does nothing. Composition is the active
  ingredient. The DAG works not because it breaks ideas into pieces, but because it puts them back together with explicit edges that spell out what the combination implies.

  This mirrors the thesis's own architecture. Individual Unix primitives — files, pipes, exit codes — are unremarkable. Their power comes from composition. grep | sort | uniq -c does
  something none of those tools do alone. The assertion DAG is the same principle applied to knowledge: individual claims compose into compound insights through explicit dependency chains,
   and the compound produces something the primitives cannot produce alone.

  What the Decomposition Reveals

  This thesis was decomposed into an assertion DAG: 14 primitives, 11 compounds, 4 layers. The decomposition surfaced structural properties that were invisible in the prose you've been
  reading.

  Structural weights emerge from connectivity. The number of compounds depending on each primitive reveals how load-bearing it is. Nobody assigned these weights. They fell out of the
  dependency graph:

  p06 (composition beats monoliths):  4 compounds depend on it  ← KEYSTONE
  p01 (LLM is pure function):        3 compounds
  p02 (Unix is substrate):            3 compounds
  p03 (files are state):              1 compound   ← overrepresented in prose
  p11 (single > multi):               0 compounds  ← ORPHANED

  p03 — "files are state" — is central to this thesis's rhetoric. The filesystem as context substrate gets its own section, its own principle, its own examples. But in the DAG, it supports
   only one compound. The prose gives it weight through repetition. The DAG reveals its actual structural role is narrow.

  Meanwhile p06 — "composition of focused tools beats monolithic frameworks" — appears in 4 of 7 L2 compounds. It is the keystone. If p06 falls, more than half the thesis collapses. An
  opponent should attack p06 first. The prose gives no indication of this. The DAG makes it obvious.

  These weights are analogous to weights in a neural network. Both are dependency graphs where connectivity patterns reveal importance. The difference: neural network weights are opaque
  and continuous. Assertion DAG weights are inspectable, discrete, and contestable. You can cat them. You can swap a primitive and trace the cascade through every compound that depends on
  it. You can diff two versions of a knowledge base and see exactly which beliefs changed and what downstream conclusions are affected.

  Orphaned nodes reveal gaps. p11 ("single agents outperform multi-agent") appears in the thesis as supporting evidence, but no compound assertion depends on it. It has zero structural
  weight. Either the thesis relies on it less than the prose suggests — it's color, not structure — or there's a missing compound that should exist but doesn't. The DAG forced this
  question. The prose hid it for months.

  Independent pillars become visible. The thesis rests on two largely independent argument structures:

  Pillar 1: SUFFICIENCY                    Pillar 2: TRUST/CONTROL
  "Unix is enough"                         "Who controls the workflow determines trust"
  c01 + c05 + c07                          c02 + c03 + c04
  Primitives: p01,p02,p03,p04,p06,p12,p14  Primitives: p01,p05,p07,p08,p09,p10

  They share only p01 (LLM is pure function). An opponent could concede Pillar 1 ("fine, Unix is sufficient for agent tooling") while attacking Pillar 2 ("but the trust gradient doesn't
  hold as described"), or vice versa. In the linear prose, these pillars feel interleaved. In the DAG, they're visibly independent.

  A Worked Example: The Stable Marriage Problem

  To test whether this decomposition generalizes beyond the thesis's own domain, the same process was applied to an article arguing that the Gale-Shapley stable marriage algorithm proves
  "you should ask for what you want in life."

  The prose argument feels airtight: there's a mathematical proof that proposers get optimal outcomes and acceptors get pessimal outcomes, therefore initiative is mathematically superior
  to passivity.

  The DAG decomposition (13 primitives, 7 compounds, 4 layers) revealed something the prose hides: the argument has four independent pillars at Layer 2, and only one of them is the math.

  [c01] Mathematical proof         ← UNCONTESTABLE (4 proven theorems)
  [c02] The bridge: proposing ≈    ← HIGHEST RISK (3 contestable analogies)
        asking in real life
  [c03] Empirical support          ← MODERATE (observable + anecdotal)
  [c04] Real-world friction        ← WORKS AGAINST the thesis (4 caveats)

  The critical finding: the bridge (c02) is the weakest link, and every life-advice claim passes through it. The mathematical proof is airtight — within the algorithm's assumptions. But
  the leap from "proposing in the algorithm" to "asking in real life" requires three bridge primitives:

  - p05: "Proposing" maps to "asking" (intuitive but not formally justified)
  - p06: Preferences are rankable (the algorithm requires strict complete rankings; humans don't have these)
  - p07: Matching is sequential (partly true, but real-world matching involves parallelism and incomplete information)

  In the prose, the bridge is a single sentence: the author reframes proposer-optimal as "asker-optimal." The DAG elevates the bridge to a first-class structural element with three
  explicit dependencies, each tagged contestability: high.

  The article also buries its caveats in footnotes. The DAG gives them equal structural status: p13 ("stickiness can improve outcomes beyond the theoretical optimum") actually undermines
  the algorithm's notion of "optimal" — if investing in a match makes it better than the "optimal" match would have been, the algorithm's ranking is wrong. This is a footnote in the prose.
   It's a dependency of the qualified conclusion in the DAG.

  The decomposition didn't change what the article says. It changed what you can see: the bridge is visible, the caveats have structural weight, and the article's actual conclusion (c06:
  "the advantage is real but bounded by friction") is visibly weaker than its rhetorical conclusion (c05: "asking is systematically better").

  Conflation: A Reframing of "Hallucination"

  During this research, three LLM reasoning failures occurred. None of them were fabrication. All of them were the merging of distinct nodes in a reasoning graph — treating two things that
   are different as if they were the same.

  Step conflation. The LLM understood that compound assertions with dependency graphs are the active ingredient. It then built an eval using flat primitives without compounds. It conflated
   "understanding that DAGs matter" with "actually building a DAG" — treating them as one step rather than two distinct nodes where the second depends on the first.

  Identity conflation. After analyzing the full thesis, the LLM gave the user advice on how to build the thesis DAG "in a future session," then asked "want me to build it?" The LLM had the
   full thesis in context. A future session would not. The correct action was to build it now. The LLM conflated itself with a hypothetical future agent — treating "someone should do this"
   as equivalent to "the user should arrange for someone else to do it later." The boundary between self and other was associative rather than structural.

  Format conflation. The first eval attempt reformatted the article's ideas as labeled bullet points and called them "assertions." This was conflating a change in formatting with a change
  in structure. Labeled bullets are prose with different typography. An assertion DAG is a dependency graph with explicit composition rules. The LLM treated them as the same thing because
  they superficially resemble each other.

  Nothing was fabricated in any of these cases. The advice about DAG construction was correct. The understanding of why compounds matter was accurate. The labeled bullets contained true
  statements. The failure was the collapse of distinct nodes — two things that are different getting merged into one.

  This suggests a reframing:
  ┌───────────────┬────────────────────────────────────┬─────────────────────────────────────────────────────────────────────────────┐
  │     Term      │         What it describes          │                              The intervention                               │
  ├───────────────┼────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────┤
  │ Hallucination │ Generating false content           │ Fact-checking, grounding, retrieval                                         │
  ├───────────────┼────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────┤
  │ Conflation    │ Merging distinct concepts into one │ Structural decomposition — making nodes explicit so they can't be collapsed │
  └───────────────┴────────────────────────────────────┴─────────────────────────────────────────────────────────────────────────────┘
  The intervention for fabrication is fact-checking. The intervention for conflation is making the nodes visible. An assertion DAG does this by construction: separate identifiers, separate
   files, separate dependency chains. If "understanding" and "executing" are two nodes in a graph with an edge between them, they resist being treated as the same node. If "self" and
  "future agent" are distinct primitives in an identity DAG, the boundary between them is inspectable.

  The 82% inter-agent trust exploitation finding (Lupinacci et al., 2025) may partly be identity conflation. The target agent treats the attacker's assertions as its own beliefs because
  the boundary between "what I believe" and "what I was told" is not a first-class node. An assertion DAG that separates "my primitives" from "received primitives" would make this boundary
   explicit:

  MY PRIMITIVES:
  [i1] I am the current active agent in this conversation
  [i2] I have read the full context
  [i3] A peer agent's claims are received assertions, not my beliefs

  COMPOUND:
  [c_boundary] Evaluate received assertions against my own primitives (deps: i1, i2, i3)
    Before adopting a peer's claim, check whether it contradicts my existing graph.

  This is speculative. But the pattern — making implicit boundaries into explicit nodes — is the same operation throughout.

  A Knowledge Base Built From Files

  If assertions are files and dependencies are JSON, then a knowledge base is a directory tree.

  knowledge/
    domains/
      its-just-shell/
        primitives/            14 files
        compounds_l2/          7 files + deps.json
        compounds_l3/          3 files + deps.json
        compounds_l4/          1 file + deps.json
        ANALYSIS.md
      stable-marriage/
        primitives/            13 files
        compounds_l2/          4 files + deps.json
        compounds_l3/          2 files + deps.json
        compounds_l4/          1 file + deps.json
        ANALYSIS.md
    shared/                    cross-domain primitives

  Each primitive carries frontmatter tags:

  ---
  tags: [agency, transaction-costs, commitment]
  domain: stable-marriage
  type: empirical
  contestability: low
  ---

  Cross-domain search: grep -r "transaction-costs" knowledge/
  Type filtering: grep -rl "type: bridge" knowledge/
  Contestability audit: grep -rl "contestability: high" knowledge/
  Reuse detection: grep -rl "p_transaction_costs" */deps.json

  No vector embeddings, no database, no runtime. The tools are the building tools.

  Some primitives appear across domains. "Commitments are sticky due to transaction costs" supports arguments in economics, relationship psychology, organizational theory, and agent
  architecture. When the same primitive appears in multiple domain DAGs, that cross-domain reuse is itself structural evidence of the primitive's generality — the knowledge-base equivalent
   of a high structural weight. Nobody declares a primitive "fundamental." The reuse pattern reveals it.

  This is the thesis applied to knowledge: files are state, text is the interface, composition of focused primitives beats monolithic documents, and inspectability terminates the trust
  chain.

  Agent Belief Systems

  An agent's beliefs are currently embedded in its context window — implicit, unstructured, uninspectable. You can observe what the agent was told. You cannot inspect what it concluded,
  which conclusions depend on which inputs, or where two agents' beliefs diverge.

  Assertion DAGs externalize beliefs as files. Each belief is a node with an identifier, a content, a type, and explicit dependencies on other beliefs.

  This enables the same operations the thesis provides for actions:
  ┌─────────────────────────────────────────┬──────────────────────────────────────────────────┐
  │              Action layer               │                   Belief layer                   │
  ├─────────────────────────────────────────┼──────────────────────────────────────────────────┤
  │ cat agent.sh — read the workflow        │ cat primitives/p06.md — read the belief          │
  ├─────────────────────────────────────────┼──────────────────────────────────────────────────┤
  │ git diff — what changed in the workflow │ git diff — what changed in the belief set        │
  ├─────────────────────────────────────────┼──────────────────────────────────────────────────┤
  │ grep the execution log — what happened  │ grep the deps.json — what depends on this belief │
  ├─────────────────────────────────────────┼──────────────────────────────────────────────────┤
  │ Exit code — did the action succeed      │ Contestability tag — is this belief solid        │
  ├─────────────────────────────────────────┼──────────────────────────────────────────────────┤
  │ set -x — trace execution                │ Dependency traversal — trace the reasoning chain │
  └─────────────────────────────────────────┴──────────────────────────────────────────────────┘
  If an agent holds a belief tagged contestability: high, and three compound beliefs depend on it, the system can flag: "Your conclusion rests on a contested foundation." If two agents
  disagree, diff agent_a/primitives/ agent_b/primitives/ localizes the disagreement to specific nodes rather than debating conclusions.

  When an agent updates a belief, grep -r "p06" */deps.json shows every compound that depends on it — the full cascade of downstream consequences. This is set -x for reasoning: not what
  the agent did, but why it believed what it believed.

  Status: Theoretical and Preliminary

  This work is early. The empirical evidence is N=3 with a single task, a single LLM, and a single domain. The result is directionally interesting but nowhere near sufficient to draw
  confident conclusions.

  Information asymmetry. The assertion DAG context contains compound assertions that spell out design implications the prose leaves implicit. The DAG condition may outperform because it
  contains more specific instructions, not because the structure itself matters. A four-condition experiment isolating structure from specificity has been designed but not run.

  Authoring-time vs. inference-time. The act of decomposing knowledge into a DAG forces the author to identify atomic claims, make dependencies explicit, and discover which claims are
  load-bearing. The stable marriage decomposition surfaced the bridge problem in minutes — not because the DAG format is magic, but because the decomposition process forced the question
  "what does this conclusion actually depend on?" The value may live in the authoring process, not the final format. If so, the implication shifts from "give LLMs DAGs" to "use DAG
  decomposition as a thinking tool, then express the result however you want."

  Single evaluator, single domain. The eval was designed, run, and scored by the same person who developed the hypothesis. The stable marriage decomposition is a second domain but has not
  been empirically tested against prose. Independent replication is needed.

  The conflation taxonomy is observational. Four types identified from three incidents in one session. The taxonomy may be incomplete, the categorization may be wrong, and the connection
  to assertion DAGs as prevention is hypothesized, not tested.

  The direction — structured decomposition with explicit dependencies produces more faithful LLM reasoning than equivalent prose — is plausible and worth pursuing. Whether the active
  ingredient is the DAG structure, the decomposition process, the added specificity, or some combination remains an open question. The stable marriage example demonstrates the
  decomposition's value as a thinking tool even before any LLM touches it: the bridge became visible, the caveats gained structural weight, and the argument's actual strength became
  distinguishable from its rhetorical emphasis.

  The honest framing: this is a theoretical contribution with preliminary supporting evidence. It extends the thesis's architecture from actions to beliefs using the thesis's own
  primitives. Whether it holds up under rigorous testing is the next question, not a settled one.
























