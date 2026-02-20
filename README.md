# Its Just Shell

"So To Better Do is a new, massively parallel, concurrent, AI driven, to-do list application." - Rich Hickey, 2012

"Agents...I like them, but I don’t like-like them" - Thomas Ptacek

"I think your thesis is actually about survival through simplicity, not optimality through architecture. " - Claude

Unix primitives are sufficient for a class of agent coordination problems, and many frameworks are premature abstractions over a substrate people don't yet understand. Embracing Unix composability introduces a methodology for producing reliable architecture with agents.

This document proposes that Unix and the shell provide a practical foundation for building and profiling local AI agent architectures. Introducing generic, uniform data representations enables them to become explicit, easily composed, and readily observable agentic systems. Process hierarchies, file descriptors, text streams, and exit codes provide robust abstractions for agent coordination, all native to the environment. The shell is the prototypical orchestration layer; Unix itself provides the underlying substrate of processes, files, and I/O streams. External capabilities, such as vector stores, distributed tracers, and semantic caches can be integrated as composable and inspectable components.

This document also argues that the word "agent" misattributes system-level behavior to the LLM component, and that misattribution can cause real engineering failures.

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

Agent frameworks are abstractions over the concrete reality of processes, files, sockets, and syscalls—of which Unix is the most universal expression developers actually touch. They abstract away processes, files, text streams, environment variables, and tool dispatches. Frameworks abstract away Unix, yet debugging often falls back into the Unix layer. Unix is by default the right level of abstraction for the domain, coordinating processes that exchange text. `set -x` shows you every command. `cat` shows you every file. `ps` shows you every process. The debugging tools are the building tools. The distance between abstraction and implementation collapses.

Abstractions over Unix do become necessary in cases, the key is to understand those cases. If your agent system grows to need Elixir/OTP supervision trees or Python's async ecosystem, those abstractions earn their weight by solving problems the shell genuinely cannot (see "Where Bash Breaks" below). The claim is focused: for the core problem of agent coordination — routing text between LLM calls, managing state in files, composing workflows from tools — the shell is the ground floor, and the ground floor is sufficient. Abstract above it when you feel the pain of needing to.

## The Only New Primitive

An LLM is a pure function: text in, text out. The only new primitive required is a way to call an LLM from the command line and get text back on stdout.

```bash
cat error.log | llm -s "What service is failing?"
```

This exists in Simon Willison's [`llm`](https://github.com/simonw/llm), bundled with a plugin ecosystem supporting hundreds of models. Raw `curl` to the Anthropic or OpenAI API does it with nothing installed at all. System prompts are just files you `cat` into the call.

Everything else — logging, memory, audit, coordination — is already Unix. Files are memory. Append is logging. `grep` and `jq` are audit. Directories are namespaces. Cron is scheduling. Pipes are composition. `chmod +x` is the plugin system. `--describe` is the plugin API.

Six primary concepts that matter in agent architecture and have been battle tested for fifty years:

| Concept | Agent Equivalent |
|---|---|
| Processes | The unit of isolation. Each agent is a process. |
| Files | The unit of state. Context, memory, and learnings are files. |
| Executables | The unit of capability. An agent's skill set is its `$PATH`. A tool that responds to `--describe` with its own JSON schema is self-documenting — no registry required. |
| Text streams | The unit of communication. Pipes compose agents. |
| Exit codes | The unit of verification. Success, failure, needs-input. |
| Mechanism/policy separation | The organizing principle. Same LLM, different system prompt, different agent. |

Every generation produces new coordination protocols — CORBA in the '90s, gRPC in the '10s, MCP and Skills today — each requiring ecosystem buy-in: server implementations, client libraries, schema definitions, capability negotiation. The shell requires only: can you emit text? And for tool discovery: can you describe yourself when asked? A directory of executables that each respond to `--describe` with a JSON schema is a tool catalog derived at runtime — `ls` is the registry, the tool is the documentation. No server process, no handshake, no capability negotiation.

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

When you send a prompt to the Anthropic or OpenAI API, the harness (Claude Code, Opencode, et al) also sends a list of "tools" — JSON schemas describing functions the LLM can request. The LLM may respond with a structured tool request: "I want to call function X with arguments Y from this tool." The harness executes the requested function, captures the result, and sends a message back to the llm containing the tool result. The LLM generates again, now with the tool result in its context. It may request another tool call, or produce a final text response.

This is a multi-turn protocol. A series of LLM responses, each potentially terminated by a tool request, with the orchestrating code executing the tools and feeding results back. The LLM never executes anything itself — it requests, waits, receives, and continues.

The LLM is, in this framing, still a pure function — it takes a context window and returns either text or a structured tool request. The harness is a mediator executing side effects. But the LLM is choosing which side effects to request, and the harness is executing those choices. The LLM drives.

### Both Exist for Good Reasons

Interactive exploration, open-ended research, creative problem-solving — these benefit from LLM-driven control. The LLM can discover approaches the script author didn't anticipate and demonstrate behavior the developer didn't explicitly program. That emergence is real and valuable. You wouldn't want to pre-script a research agent's path through unfamiliar territory.

Production runbooks, deployment pipelines, security-critical operations, batch processing, CI/CD, cron jobs — these however require script-driven control. The workflow is deterministic, auditable, and composable. You wouldn't want a deployment agent demonstrating emergent unpredictable behavior.

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

To build any of these in a framework that couples to LLM-driven control, you'd need to build Unix-like primitives within that framework — subprocesses, file coordination, script-controlled sequencing. At which point you've rebuilt the shell inside the framework. 

---

## Natural Language Coordination

This thesis does not prohibit natural language agent coordination. It asserts a separation of concerns:

- **Deliberation layer:** Unconstrained. Agents may coordinate, negotiate, and share knowledge in natural language.
- **Execution layer:** Shell-semantic. Actions must be commands, file operations, or processes with observable effects.
- **Verification layer:** Shell-inspectable. Ground truth is the execution trace, not the conversation about it.

When NL serves as coordination, execution, *and* verification — when "doing something" means "saying you did it" — the system loses its ground floor. Agents are the first software that needs to be inspected by other agents. Natural language output cannot be programmatically verified without another LLM. Shell output can be grep'd, diff'd, hashed. The inspection chain terminates.

How well this separation holds depends on the control model:

In **script-driven** mode, the separation is enforced by construction. The LLM only participates in the deliberation layer — it answers questions, provides judgment. All execution is shell commands in the script. Verification is the execution trace. The layers cannot bleed into each other because the LLM has no mechanism to execute.

In **LLM-driven** mode, the separation requires discipline. The LLM participates in execution decisions — it requests tools, chooses actions. The logging primitives capture what happened, so verification remains possible. But the deliberation and execution layers are no longer structurally separated. The LLM's reasoning about *why* it chose a tool and the tool's execution are interleaved in the same loop. This is manageable, but it's a practice you maintain rather than a guarantee you inherit.

**Let agents chat about what to do. Log what they actually did.** In script-driven mode, this is guaranteed. In LLM-driven mode, it's an optional discipline.

---

## The Trust Gradient

```
High trust required ←————————————————→ Low trust required

Natural language    Structured APIs    Shell semantics
(must trust         (must trust        (verify by
 interpretation)     implementation)    inspection)
```

When you receive a message from your agent saying "I've secured the server," you're trusting the agent's interpretation of "secured." When you see `chmod 600 ~/.ssh/*` in the execution log, you're trusting only your own understanding of shell semantics - or, your supervising agent's understanding of them.

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

The key distinction is not observability — both modes can be logged equally well. The distinction is *inspectability*. A script-driven workflow can be inspected before and after it runs. You read the script. You know what it will do. An LLM-driven workflow can only be inspected after it runs. You read the logs. You learn what it did.

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

LLM-driven agents are powerful at the 0th order — they do tasks. But they resist composition into higher-order systems. The workflow is decided at runtime, opaquely, inside the LLM's context window. You can log what happened and study the logs, but the workflow itself isn't an artifact you can version, diff, or systematically improve. You can't script what you can't predict. Building a 2nd-order system that optimizes LLM-driven agents means analyzing logs and adjusting prompts — indirect, lossy, slow. However, in shell, you can contain this runtime opaqueness to the `llm` primitive, not relegate the entire harness to a monolithic TUI or IDE GUI that routes you away from thinking in terms of inspectability about the components that in fact are.

Script-driven agents are artifacts at every order. The skill script is a program. Building a 2nd-order system means writing a script that runs and evaluates other scripts. Building a 3rd-order system means automating that evaluation. The derivative stack is natural because every layer is a script observing and controlling scripts.

Practitioners are hitting this ceiling today. They want to build eval pipelines but can't easily capture agent outputs in batch — the agent expects interactive input. They want to A/B test prompts at scale but can't run agents non-interactively. These are symptoms of LLM-driven architectures resisting composition.

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
The system prompt is one layer. What matters is the total state at inference: system prompt, tool definitions, conversation history, files on disk, current query. Think in environments, not prompts. Context assembly is itself a composition problem: a `soul.md` identity file, context modules from a `context/` directory loaded alphabetically, tool schemas derived from `--describe` — each a focused file, composed into the full context at call time. This is pipes applied to prompt construction.

### 5. Transparency Over Abstraction
If you can't `cat` it, be suspicious. Bash scripts over compiled binaries. Markdown over proprietary formats. SQLite over SaaS data stores. Git repos over vendor lock-in.

### 6. Separation of Mechanism and Policy
The LLM's capability is mechanism. The system prompt, tool availability, guardrails, and skill scripts are policy. Don't bake policy into mechanism. Express domain logic through context.

Script-driven and LLM-driven modes are the clearest demonstration. The primitives (any LLM CLI, files, pipes) are mechanism. They work identically regardless of who controls the workflow. The choice of script-driven or LLM-driven is policy, expressed in the skill script. Switching between them doesn't require changing infrastructure — it requires changing the script. Frameworks that couple to LLM-driven control bake the policy (LLM drives) into the mechanism (the tool-calling loop). These primitives keep them separate.

Testing follows the same principle. Set an environment variable (`SHELLCLAW_STUB=1`) and the same tool interface returns canned responses instead of calling real APIs. The tool doesn't know it's being tested. The dispatch doesn't know it's running a stub. The mechanism is identical; only the policy changes. This is how you test agent systems offline — not by mocking frameworks, but by swapping policy.

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

- [**shellclaw**](https://github.com/Its-Just-Shell/shellclaw) — Library of composable primitives for agent architecture. Five orthogonal libraries (log, config, session, llm, compose) with a thin entry point that is one example composition. Self-describing tools via `--describe`, runtime catalog discovery, context module assembly, and example orchestration scripts (chat loop, pipe summarization, batch processing, soul swapping). Built with `llm` and standard Unix tools. The v1 nine-step agent pattern (LOG → LOAD → GATHER → COMPOSE → CALL → LOG → EXTRACT → SHARE → OUTPUT) decomposed into five reusable primitives — the primitives are more fundamental than the steps, and different compositions arrange them differently.

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


























