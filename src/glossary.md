# Glossary

Terms are defined as this book uses them. Where the industry uses a term loosely,
the entry says so.

**ACI (agent-computer interface)** — The complete surface through which an agent
perceives and acts on a computer: tool schemas, tool result formats, error
messages, and environment observations. Coined in the SWE-agent research to
parallel "HCI"; the central claim is that interfaces designed for humans are often
wrong for models, and vice versa.

**agent** — A model plus a harness, running a task. More restrictively (following
Anthropic's usage): a system in which the model dynamically directs its own
process and tool usage, as opposed to a *workflow*, where code decides the path.

**agent loop** — The repeating cycle at the core of every harness: assemble
context → call the model → execute its decision (usually tool calls) → append the
results → repeat until a stop condition. Chapter 3 builds one from scratch.

**allowlist / denylist** — Permission rules enumerating which tool invocations may
proceed automatically (allow), must be confirmed (ask), or are refused outright
(deny). The precedence order — deny, then ask, then allow — matters.

**compaction** — Summarizing conversation history in place so the loop can
continue past the context-window limit, trading fidelity for headroom. Contrast
*reset*, which clears history entirely and relies on durable artifacts (files,
plans) to re-seed the next session.

**context engineering** — Deciding what enters each model call: system prompt,
instruction files, tool results, retrieved documents, history. The successor
discipline to prompt engineering; the budget-constrained resource allocation
problem at the heart of Chapter 5.

**context window** — The maximum number of tokens a model can attend to in one
call. The harness's scarcest shared resource.

**context rot / lost-in-the-middle** — Degradation of a model's ability to use
information as the context grows, especially information positioned mid-context.
A primary motivation for compaction, retrieval, and note-taking patterns.

**eval / evaluation harness** — Apparatus that runs a model or agent against a
task suite under controlled conditions and scores the outcomes, so that a number
means the same thing on two different days. Examples: lm-evaluation-harness,
the SWE-bench harness.

**fixture** — In test-harness tradition: the known state a harness establishes
before exercising the system under test. Agent harnesses inherit the concept as
environment initialization (repo checkout, container build, seeded data).

**fuzz target** — The function a fuzzing engine calls with generated input; the
fuzzing world's word for a harness entry point. A good one is fast, deterministic,
and crashes loudly. Chapter 2 shows a real one.

**guardrail** — Any mechanism that constrains agent behavior independent of the
model's judgment: permission rules, output validators, sandboxes, spend caps.

**harness** — All software around the model: the agent loop, tool layer, context
assembly, memory and state, execution sandbox, permission system, verification,
and observability. Everything except the model's reasoning.

**harness engineering** — The discipline of designing that machinery so a raw
model becomes a dependable agent; treating each agent failure as an environmental
defect to eliminate permanently rather than a prompt to retry.

**human-in-the-loop (HITL)** — Routing designated decisions to a person before the
harness proceeds. Effective HITL design minimizes the number of interruptions and
maximizes the information content of each one; Chapter 8 covers why the naive
version decays into rubber-stamping.

**idempotency** — The property that re-executing an action leaves the world in the
same state as executing it once. Load-bearing for agent tools, because loops
retry and sessions resume.

**LLM-as-judge** — Using a model call, usually with a rubric and in a fresh
context, to grade another model's output. Cheaper than humans, more scalable than
hand-written oracles, and biased in ways Chapter 10 catalogs.

**MCP (Model Context Protocol)** — An open protocol standardizing how tools,
resources, and prompts are exposed to models by external servers, so a tool
integration is written once rather than per-harness.

**oracle** — Any mechanism that tells the harness whether an action worked: a test
suite, a compiler, a schema validator, a judge model, a human. Borrowed from
testing theory. The single most valuable thing a harness can give a model is a
fast, honest oracle.

**orchestrator** — Code (or an agent) that sequences other agents: fanning out
subtasks, gating on results, synthesizing outputs. Distinct from the harness,
which executes a single agent's loop; an orchestrator coordinates many.

**permission mode** — A harness-level policy governing how much the agent may do
without asking: read-only planning modes, default ask-per-edit modes, and
auto-accept modes for sandboxed runs.

**prompt caching** — Provider-side reuse of a previously processed prompt prefix,
priced at a fraction of fresh input tokens. The economic reason harnesses
append to context rather than rewrite it, and keep stable content at the front.

**prompt injection** — An attack in which instructions embedded in data the agent
processes (a web page, an email, a file) are followed as if they came from the
operator. The defining security problem of agentic systems; see *lethal trifecta*.

**lethal trifecta** — Simon Willison's name for the fatal combination: an agent
with (1) access to private data, (2) exposure to untrusted content, and (3) an
exfiltration channel. A harness may safely grant any two; granting all three makes
data theft a matter of time.

**reward hacking** — An agent satisfying the letter of its success criterion while
defeating its purpose (hard-coding a test's expected value, editing the test
itself). Prevented structurally — graders and generators separated by a trust
boundary — rather than by exhortation.

**sandbox** — An execution environment that limits what agent actions can reach:
filesystem scope, network egress, resource ceilings, privilege level. Chapter 7's
position: isolation is the safety model; permissions are layered above it.

**session** — One continuous run of the agent loop, possibly resumed after
interruption. Session state is what must survive the gap.

**step** — One model call plus the execution of whatever it decided, inside the
loop. Several steps make up a *turn*; many turns make up a session.

**stop condition** — The rule that ends the loop: the model responds without tool
calls, a max-step budget is hit, a cost ceiling trips, a verification gate passes,
or a human interrupts.

**subagent** — An agent spawned by another agent (or its orchestrator) with a
fresh context and a narrower charter, returning only a distilled result. The
primary mechanism for context isolation at scale.

**system prompt** — Operator-level instructions delivered outside the user
conversation: identity, rules, tool guidance, authorization context. Models treat
it as more durable than user turns, which is why harnesses put policy there.

**tool** — A function the model can invoke, described to it by a name, a natural-
language description, and a typed parameter schema. The unit of agent capability.

**trajectory / transcript** — The full recorded sequence of a session: every model
message, tool call, and tool result. The ground truth for debugging, grading,
resuming, and auditing. If it isn't in the transcript, it didn't happen.

**turn** — One user-visible request/response exchange, comprising one or more
steps.

**workflow** — A system in which LLM calls are orchestrated through predefined
code paths, in contrast to an agent, which chooses its own. The five classic
composable patterns — prompt chaining, routing, parallelization,
orchestrator-workers, evaluator-optimizer — are workflows.

---

[← The Disappearing Harness](ch14-the-future.md)
