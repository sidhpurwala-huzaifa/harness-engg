# Chapter 9 — Orchestration and Multi-Agent Patterns

*A single agent in a single loop takes you a long way — further than most teams expect, as Chapter 3 showed. But some tasks exceed what one context window can hold, some benefit from parallel exploration, and some need a skeptical second opinion that the agent doing the work cannot provide for itself. This chapter is about composition: the five workflow patterns from Anthropic's "Building Effective Agents," subagents as a context-isolation mechanism, the three-agent planner/generator/evaluator harness for long-running work, and the discipline that holds all of it together — deterministic orchestration code wrapped around nondeterministic agents, with files rather than chat messages as the handoff medium. It is also, pointedly, about when not to do any of this: multi-agent architectures multiply token cost roughly fifteenfold and introduce coordination failures that single agents never face, so the burden of proof sits with complexity.*

## Workflows are not agents

The vocabulary matters, so start with the distinction that Anthropic's ["Building Effective Agents"](https://www.anthropic.com/engineering/building-effective-agents) made canonical. **Workflows** are systems where models and tools are orchestrated through predetermined code paths: your code decides the sequence of steps, and the model fills in the judgment inside each step. **Agents** are systems where the model dynamically directs its own process and tool usage — it decides what to do next, in the loop this book has been building since Chapter 3. Workflows offer predictability and consistency for well-defined tasks; agents earn their cost when flexibility and model-driven decision-making are needed.

This distinction is the axis of the whole chapter. Orchestration is the act of deciding, in code or in a coordinating agent, how model calls and agent runs compose — and the central design question at every junction is *who holds the control flow*: your deterministic script, or a model. The consistency glossary of this book uses **orchestrator** for whichever of the two is sequencing other agents, but the engineering trade-off is very different depending on which one it is, and we will return to it at the end of the chapter.

Anthropic's post opens with advice that most multi-agent architectures ignore at their peril: "Success in the LLM space isn't about building the most sophisticated system. It's about building the right system for your needs," and developers should "consider adding complexity only when it demonstrably improves outcomes." Before reaching for any pattern below, the recommended baseline is a single model call optimized with retrieval and in-context examples. Everything in this chapter trades latency and cost — often a lot of both — for capability, and the trade only pays when the capability gap is real.

Ledgerbot, this book's running example — an agent that triages and fixes failing CI builds in a mid-size Python monorepo — will supply the worked examples. A nightly CI run produces dozens of independent failures: flaky tests, broken imports from a refactor, a dependency bump that changed an API. That shape (many mostly-independent subproblems, each needing real judgment) is exactly where orchestration questions get interesting.

## The five composable patterns

"Building Effective Agents" catalogs five workflow patterns. They are *composable*: real systems layer them, and the three-agent harness later in this chapter is essentially several of them welded together. Each pattern below gets a definition, a code sketch, and the conditions under which it is worth its overhead.

The sketches share one helper. Nothing about the patterns is provider-specific; the helper just makes them runnable with the public Anthropic SDK:

```python
import anthropic

client = anthropic.Anthropic()
MODEL = "claude-sonnet-4-5"

def llm(prompt: str, system: str = "") -> str:
    """One model call, no tools. The atomic unit of every pattern below."""
    response = client.messages.create(
        model=MODEL,
        max_tokens=4096,
        system=system or anthropic.NOT_GIVEN,
        messages=[{"role": "user", "content": prompt}],
    )
    return response.content[0].text
```

Where a sketch needs a full agent (a loop with tools, per Chapter 3) rather than a single call, it invokes `run_agent(...)` and says so.

### Prompt chaining

Prompt chaining decomposes a task into a fixed sequence of steps, where each model call consumes the previous call's output. The pattern's distinguishing feature is the **gate**: a programmatic check between steps that validates intermediate output before the harness spends more tokens on it. Use it when a task decomposes cleanly into fixed subtasks and you are willing to pay extra latency for higher per-step accuracy — each call gets a simpler, more focused job than one call doing everything.

```python
def summarize_failures(raw_log: str) -> str:
    """Chain: extract -> gate -> classify -> report."""
    extraction = llm(
        "Extract every distinct test failure from this CI log. "
        "One line per failure: test name, error type, file.\n\n" + raw_log
    )
    # Gate: deterministic sanity check between steps.
    if not extraction.strip() or len(extraction.splitlines()) > 200:
        raise ValueError("Extraction implausible; do not spend tokens downstream")

    classified = llm(
        "Group these failures by likely root cause "
        "(flaky, dependency, code regression):\n\n" + extraction
    )
    return llm(
        "Write a triage summary for the on-call engineer. "
        "Lead with the largest group:\n\n" + classified
    )
```

The gate is doing quiet but important work: it is a deterministic oracle (Chapter 10's subject) placed at the cheapest possible point. A malformed extraction caught here costs one wasted call; caught at the end, it costs three.

### Routing

Routing classifies an input and dispatches it to a specialized downstream handler — a different prompt, a different tool set, or a different model entirely. It buys separation of concerns: each handler's prompt can be tuned for its category without the categories interfering, and cheap inputs can go to cheap models. Use it when inputs fall into distinct categories that are genuinely handled better separately, and when the classification itself is reliable — a misroute sends the input to a handler tuned for the wrong problem.

```python
SPECIALISTS = {
    "flaky_test": "You are a test-reliability engineer. Diagnose nondeterminism: "
                  "timing, ordering, shared state, network. Recommend a fix or a quarantine.",
    "dependency": "You are a build engineer. Diagnose version conflicts and lockfile "
                  "drift. Recommend the minimal pin or upgrade.",
    "regression": "You are a debugger. Identify the offending change and the "
                  "smallest correct fix.",
}

def triage(failure: str) -> str:
    label = llm(
        "Classify this CI failure as exactly one of: "
        + ", ".join(SPECIALISTS) + ". Reply with the label only.\n\n" + failure
    ).strip().lower()
    system = SPECIALISTS.get(label, SPECIALISTS["regression"])  # safe default
    return llm("Diagnose and propose a fix:\n\n" + failure, system=system)
```

Note the `.get(...)` with a default. The classifier is a model; it will eventually emit a label you did not enumerate. Routing code that crashes on an unexpected label has put nondeterminism in charge of control flow — exactly the failure the pattern exists to avoid.

### Parallelization

Parallelization runs multiple model calls simultaneously and aggregates the results. Anthropic identifies two variants that look similar in code but serve opposite purposes. **Sectioning** splits a task into independent subtasks and runs them in parallel for speed and focus — separate concerns get separate, uncontaminated contexts. **Voting** runs the *same* task multiple times and aggregates for confidence — diverse samples of one judgment rather than one sample each of diverse judgments.

```python
import asyncio

aclient = anthropic.AsyncAnthropic()

async def allm(prompt: str, system: str = "") -> str:
    response = await aclient.messages.create(
        model=MODEL, max_tokens=4096,
        system=system or anthropic.NOT_GIVEN,
        messages=[{"role": "user", "content": prompt}],
    )
    return response.content[0].text

async def review_patch_sectioned(diff: str) -> list[str]:
    """Sectioning: independent aspects, parallel focused reviews."""
    aspects = ["correctness", "security", "test coverage"]
    return await asyncio.gather(*[
        allm(f"Review this diff for {aspect} problems only. "
             f"Report findings or 'none'.\n\n{diff}")
        for aspect in aspects
    ])

async def patch_is_safe_voted(diff: str, n: int = 5) -> bool:
    """Voting: same question n times, majority wins."""
    verdicts = await asyncio.gather(*[
        allm("Could this diff break production behavior? "
             "First reason briefly, then answer on the last line: SAFE or UNSAFE.\n\n" + diff)
        for _ in range(n)
    ])
    safe = sum(v.strip().splitlines()[-1].strip().upper() == "SAFE" for v in verdicts)
    return safe > n // 2
```

Voting is a verification technique as much as an orchestration one — Chapter 10 develops it into majority-vote judging and connects it to the self-consistency research that quantifies its gains. The orchestration-side lesson is about the aggregator: `asyncio.gather` plus a counter is deterministic code, and it should be. Aggregation is bookkeeping, not judgment.

### Orchestrator-workers

In the orchestrator-workers pattern, a central model **dynamically** decomposes a task, delegates the pieces to worker calls (or full worker agents), and synthesizes the results. The word *dynamically* is what separates this from sectioning: in sectioning, your code knows the subtasks in advance; here the subtasks depend on the input, so a model must invent them. Use it when the decomposition itself requires judgment — a multi-file change where the files needing edits depend on the task, or research where the promising leads emerge only after looking.

```python
import json

def orchestrate(task: str) -> str:
    plan = json.loads(llm(
        "Decompose this task into independent subtasks a worker can execute "
        "without seeing the others. Return JSON only:\n"
        '{"subtasks": [{"id": "...", "objective": "...", '
        '"output_format": "...", "boundaries": "..."}]}\n\n'
        f"Task: {task}"
    ))
    results = {
        sub["id"]: run_agent(          # full agent loop per worker, own context
            objective=sub["objective"],
            output_format=sub["output_format"],
            boundaries=sub["boundaries"],
        )
        for sub in plan["subtasks"]
    }
    return llm(
        "Synthesize these worker reports into one coherent answer for the "
        f"original task.\n\nTask: {task}\n\nReports: {json.dumps(results)}"
    )
```

The subtask schema is not decorative. Anthropic's team, describing the production orchestrator-workers system behind their Research feature, found that vague delegation caused duplicated and missing work, and that "each subagent needs an objective, an output format, guidance on the tools and sources to use, and clear task boundaries" ([How we built our multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system)). They also had to teach the orchestrator *effort scaling* explicitly — "simple fact-finding requires just 1 agent with 3–10 tool calls, direct comparisons might need 2–4 subagents with 10–15 calls each" — because without those heuristics the orchestrator either spawned fifty workers for a trivial question or one for a sprawling one. An orchestrating model needs the same resource intuitions a deterministic scheduler would encode in constants.

### Evaluator-optimizer

The final pattern puts two model roles in a loop: one generates, the other evaluates and produces feedback, and the generator revises against that feedback until the evaluator passes it or a round budget runs out. Use it when clear evaluation criteria exist and iteration demonstrably improves the output — the analogue of a human writer responding to an editor.

```python
def refine(task: str, max_rounds: int = 3) -> str:
    draft = llm(task)
    for _ in range(max_rounds):
        verdict = json.loads(llm(
            "Grade the draft strictly against the task. Return JSON only: "
            '{"pass": true/false, "feedback": "specific, actionable"}\n\n'
            f"Task: {task}\n\nDraft: {draft}"
        ))
        if verdict["pass"]:
            break
        draft = llm(
            f"Revise the draft to address this feedback.\n\n"
            f"Feedback: {verdict['feedback']}\n\nTask: {task}\n\nDraft: {draft}"
        )
    return draft
```

Two implementation notes carry most of the pattern's value. First, the round budget (`max_rounds`) belongs to the harness, not the model — an evaluator that never passes anything will otherwise loop forever at your expense. Second, the evaluator here is a *separate call with a separate prompt*, not the generator critiquing itself in the same context. That separation looks cosmetic in a ten-line sketch; it is not. Models grade their own recent work with a strong positive skew — the self-evaluation bias that Chapter 10 examines in detail — and an independent evaluator is the cheapest known correction. This little loop is the seed of the three-agent harness later in this chapter.

## Subagents as context isolation

The five patterns describe control flow. Subagents solve a different problem: the context window.

Chapter 5 established the constraint — the context window is scarce shared memory, and performance degrades as it fills. A **subagent** is a child agent spawned by a parent, with its own fresh context window, its own (often narrower) tool set, and a mandate to return a distilled result. From the parent's perspective, a subagent is a context firewall: the child can read forty files, run twenty commands, and burn a hundred thousand tokens of exploration, and all the parent's context receives is the summary.

This is the actual engineering rationale for most production subagent use, and it is worth stating plainly because "multi-agent" marketing usually obscures it: subagents are less about intelligence emerging from collaboration and more about *keeping exploration debris out of the context that matters*. The [Claude Code best-practices guide](https://code.claude.com/docs/en/best-practices) is explicit that subagents exist because "context is your fundamental constraint": research subagents "run in separate context windows and report back summaries," keeping the main conversation clean for implementation. Anthropic's research-system writeup generalizes the point — subagents act as intelligent filters whose "separation of concerns — distinct tools, prompts, and exploration trajectories — reduces path dependency and enables thorough, independent investigations," and the architecture exists precisely because research tasks routinely exceed what a single context window can hold.

The economics deserve equal billing with the benefits, because they are brutal. Anthropic reports that in their production traffic, agents use about **4× more tokens than chat interactions, and multi-agent systems use about 15× more tokens than chats**. The payoff, when the task fits, is large — their multi-agent system beat single-agent Claude Opus 4 by **90.2%** on an internal research evaluation — and their analysis found that token usage alone explained roughly **80% of the performance variance** on browsing tasks. Read those numbers together and the honest summary is: multi-agent architectures are, to a first approximation, a mechanism for *spending more tokens productively on one task than one context window permits*. If a task doesn't need that many tokens, the architecture is pure overhead.

When Ledgerbot investigates a failing build, the subagent version looks like this: the parent keeps a terse ledger of failures and verdicts; for each suspicious failure it spawns an investigator subagent whose instructions name the failing test, the output format for the verdict, and the boundary ("diagnose; do not fix"). The investigator reads test code, git history, and CI logs in its own context and returns a five-line verdict. Ten investigations cost ten disposable contexts and leave the parent's ledger small enough that the *fixing* phase — the part where accumulated coherence matters — starts with a clean, information-dense context. That division, sprawling reads in disposable contexts and coherent writes in a durable one, is the single most reliable heuristic in this chapter.

## When multi-agent is overhead

The strongest published counterargument to everything above is Cognition's essay ["Don't Build Multi-Agents"](https://cognition.com/blog/dont-build-multi-agents), and any honest treatment of orchestration has to engage it, because the failure mode it describes is real and common.

The essay advances two principles. First: **"Share context, and share full agent traces, not just individual messages."** Second: **"Actions carry implicit decisions, and conflicting decisions carry bad results."** The argument: when parallel subagents work on divided portions of one task, each makes dozens of small implicit decisions — naming, styling, architectural assumptions — that are not prescribed by the task description. Their illustration is an agent asked to build a Flappy Bird clone whose two subagents return, respectively, a Super Mario-styled background and a bird that matches neither the background nor the game; the synthesizing agent cannot reconcile artifacts built on conflicting assumptions it never saw being made. Cognition's recommended default is a single-threaded linear agent with continuous context, adding a dedicated compression model to summarize history for very long tasks — coherence over parallelism.

Anthropic's research-system team, running one of the most successful multi-agent systems in production, agrees about the boundary. Their writeup notes that multi-agent architectures excel at breadth-first, heavily parallelizable tasks whose information exceeds single context windows — and underperform on tasks that require "all agents to share the same context or involve many dependencies between agents," naming most coding tasks as the canonical example. These are not opposing camps; they are describing two sides of one criterion:

**Parallelize reads; serialize writes.** Investigation, search, review, and evaluation decompose well, because each worker's output is *information* and information merges cheaply. Construction decomposes badly, because each worker's output embodies *decisions* and conflicting decisions do not merge at all. Ledgerbot fans out investigation across ten failures without hesitation; it does not split "fix the auth module" between two agents, because the fix is a web of interlocking decisions that belongs in one context. When it must fix ten *independent* failures, it runs one agent per failure — parallel *tasks*, each internally single-threaded, which is fan-out (a scheduling decision made by deterministic code) rather than a multi-agent system in Cognition's sense.

The cost side compounds the argument: at roughly 15× chat-level token consumption, a multi-agent topology must deliver a step-change in outcome, not a marginal improvement, to justify itself. When in doubt, the single agent is the right default — which is precisely the "simplest solution possible" advice that opens "Building Effective Agents."

## Three agents and a contract: planner, generator, evaluator

For long-running autonomous work — hours of unattended building, not minutes of triage — the most instructive published design is the three-agent harness from Anthropic's ["Harness design for long-running application development"](https://www.anthropic.com/engineering/harness-design-long-running-apps), which turns a one-paragraph app idea into a working full-stack application over multiple sessions. It composes the patterns above into a stable architecture, and its structure — one agent generating, a separate agent judging, iterating against each other — deliberately echoes the generator/discriminator split of a GAN, with the crucial difference that nothing is being trained; the adversarial pressure is applied at inference time, by the harness.

The three roles:

- **Planner.** Runs once, first. Takes the user's one-to-four-sentence prompt and expands it into a comprehensive product specification — instructed to "be ambitious about scope" and to stay at the level of product intent rather than implementation detail. Its output is a durable artifact the other two agents consume for the rest of the run.
- **Generator.** Implements the specification incrementally, in sprint-sized chunks, using version control throughout, and self-checks before handing off.
- **Evaluator.** Tests the *running application* the way a user would — driving it with Playwright, clicking through features — and grades each sprint against the bugs it found and criteria covering product depth, functionality, visual design, and code quality. Its graded feedback goes back to the generator, which iterates.

Why a separate evaluator at all, when the generator could critique its own work? Because it can't — not usefully. The post's central empirical finding is that "when asked to evaluate work they've produced, agents tend to respond by confidently praising the work — even when, to a human observer, the quality is obviously mediocre." The fix is architectural, not promptual: "tuning a standalone evaluator to be skeptical turns out to be far more tractable than making a generator critical of its own work, and once that external feedback exists, the generator has something concrete to iterate against." Chapter 10 treats this bias and its remedies in depth; here the point is what it does to *topology*. Self-evaluation bias is the strongest known argument for adding a second agent to a system, considerably stronger than parallelism — parallelism buys speed, but a separated evaluator buys a capability the single agent structurally lacks.

Two coordination mechanisms make the trio work. The first is the **sprint contract**: before each chunk of work, "the generator and evaluator negotiated a sprint contract: agreeing on what 'done' looked like for that chunk of work before any code was written." This is the evaluator-optimizer pattern with the acceptance criteria moved *before* generation — the same move test-driven development makes, and for the same reason: a definition of done written after the work is done tends to describe whatever got built. The second is file-based communication, which earns its own section below.

The cost profile is stark and worth internalizing. For a retro-game-maker application, the full three-agent harness took **6 hours and about $200**; a solo agent given the same prompt took **20 minutes and $9**. Twenty-odd times the cost — and the solo version had broken gameplay while the harness version was a functional, polished application with integrated AI features. Both facts are the lesson: the architecture works, and it is far too expensive to be a default. You deploy a planner/generator/evaluator harness when the deliverable justifies hours and hundreds of dollars, not for a lint fix.

One more finding from the same post reaches beyond this architecture and into Chapter 14's territory: "every component in a harness encodes an assumption about what the model can't do on its own, and those assumptions are worth stress testing, both because they may be incorrect, and because they can quickly go stale as models improve." When a newer model (Opus 4.6, in the post's account) arrived with better planning and longer coherence, the sprint decomposition built for its predecessor became partially unnecessary, and the author simplified — moving the evaluator to a single pass at the end of the run rather than grading every sprint. Orchestration topologies are not permanent architecture. They are dated compensations for specific model weaknesses, and they need re-auditing at every model generation.

## Handoff artifacts: files over chat

Every multi-agent design has to answer a mundane question that determines most of its reliability: when agent A finishes and agent B starts, *what actually moves between them?*

The consistently successful answer in production systems is: **a file.** In the three-agent harness, agents communicate through structured files on disk — "one agent would write a file, another agent would read it and respond either within that file or a new file that the previous agent would read in turn." The specification is a file; sprint contracts are files; evaluation reports are files. The [initializer-executor pattern](https://www.firecrawl.dev/blog/what-is-an-agent-harness) that Chapter 6 introduced for multi-session work is the same idea rotated ninety degrees: an initializer agent runs once and writes the durable environment — folder structure, feature list, setup script, initial commit — and every subsequent executor session reads that environment, makes incremental progress, updates the progress file, and exits cleanly. Whether the agents are separated by role or by time, the handoff medium is the filesystem.

Files beat in-conversation handoffs for reasons that compound:

- **Durability.** A file survives the crash, timeout, or context reset of the agent that wrote it. Chat context dies with its process. For anything long-running, this alone decides the question.
- **Inspectability.** You can `cat` a sprint contract mid-run, diff two plans, or grep a week of evaluation reports. Debugging a file-mediated system is reading a directory; debugging a message-mediated one is archaeology (Chapter 11 returns to this as the transcript-is-ground-truth discipline).
- **Schema enforcement.** A handoff file can be validated — JSON schema, required fields, line limits — by deterministic code *before* the next agent spends tokens on it. A malformed handoff becomes a cheap crash at the boundary instead of an expensive confusion downstream. This is the prompt-chaining gate again, applied between agents. As Chapter 6 argued, structured formats beat prose for state precisely because they can be checked.
- **Idempotent re-entry.** If artifacts on disk fully describe progress, a replacement agent can pick up where a dead one stopped. Resumability falls out of the storage decision.

There is also a quieter benefit: a file forces distillation. An agent told to "write `verdict.json` with these five fields" must commit to conclusions; an agent handing off a conversation can gesture at forty tool results and hope its successor infers the point. Cognition's first principle — share full traces, not summaries — pulls the other way, and the tension is real: within one tightly coupled task, more shared context is better; across a trust or phase boundary (investigation → fixing, generation → evaluation), a distilled, validated artifact is better. Note the special case, though: when the *evaluator* is downstream, the distillation should be performed by the harness (the diff, the test output, the running application), not authored by the generator — Chapter 10 explains why letting the graded party write the grader's briefing undermines the grade.

Deciding which boundary you are standing at is most of the design work.

## Deterministic orchestration around nondeterministic agents

The last pattern is the one this chapter has been quietly using in every code sketch, and it deserves to be named as a principle: **keep the orchestration layer deterministic wherever determinism is possible, and spend nondeterminism only where judgment is genuinely required.**

The [Firecrawl harness taxonomy](https://www.firecrawl.dev/blog/what-is-an-agent-harness) separates the *harness* (the runtime that executes an agent with tools, memory, and state) from the *orchestrator* (the control flow deciding when and how to invoke the model). That orchestrator can be an LLM — the orchestrator-workers pattern requires it, because there the decomposition itself needs judgment. But an enormous amount of production orchestration needs no judgment at all: enumerate work items, spawn an agent per item, enforce timeouts and budgets, validate artifacts, retry failures, collect results. Every one of those is a `for` loop, a schema check, or a counter, and putting a model in charge of them converts free reliability into paid unreliability.

Here is Ledgerbot's nightly orchestrator. Note what is code and what is agent:

```python
"""nightly_fix.py — deterministic shell, nondeterministic core.

Headless agent invocations (`claude -p`, or any agent CLI with a
non-interactive mode) are just subprocesses; everything around them
is ordinary, testable Python.
"""
import json
import pathlib
import subprocess

MAX_AGENTS = 10          # budget: decided by code, not by a model
TIMEOUT_S = 900          # per-agent wall clock

failures = json.loads(pathlib.Path("ci_failures.json").read_text())

for i, failure in enumerate(failures[:MAX_AGENTS]):
    workdir = pathlib.Path(f"runs/fix_{i:03d}")
    workdir.mkdir(parents=True, exist_ok=True)
    (workdir / "task.md").write_text(render_task(failure))    # artifact in

    try:
        proc = subprocess.run(
            ["claude", "-p",
             f"Fix the CI failure described in {workdir}/task.md. "
             f"Run the failing test to confirm the fix. Write the patch to "
             f"{workdir}/fix.patch and a summary to {workdir}/result.json.",
             "--allowedTools", "Read,Edit,Bash(pytest *)",
             "--output-format", "json"],
            timeout=TIMEOUT_S, capture_output=True, check=False,
        )
        (workdir / "transcript.json").write_bytes(proc.stdout)  # ground truth out
    except subprocess.TimeoutExpired:
        (workdir / "result.json").write_text('{"status": "timeout"}')
        continue

    # Deterministic gate on the handoff artifact.
    result_file = workdir / "result.json"
    if not result_file.exists():
        result_file.write_text('{"status": "no_artifact"}')
    elif not (workdir / "fix.patch").exists():
        json_patch_missing(result_file)   # mark for human review

report = [json.loads((d / "result.json").read_text())
          for d in sorted(pathlib.Path("runs").iterdir())]
print(json.dumps({"attempted": len(report),
                  "fixed": sum(r.get("status") == "fixed" for r in report)}))
```

This is the fan-out idiom that the [Claude Code best-practices guide](https://code.claude.com/docs/en/best-practices) recommends for large migrations — loop over a generated task list, invoke `claude -p` per item with scoped `--allowedTools`, test the prompt on two or three items before running the full set. What makes it dependable is the division of labor:

- **Scheduling, budgets, timeouts** are constants and loops. The run costs what the code says it costs; no model can decide to spawn fifty workers.
- **Handoff artifacts** are written to known paths and gated by deterministic checks. An agent that produced no patch is *recorded* as having produced no patch — its own opinion of its success is not consulted (Chapter 10 explains exactly why not).
- **Failure semantics** are explicit. A timeout, a crash, and a missing artifact each land in `result.json` as distinct machine-readable states, and one straggler cannot poison the batch. Chapter 11 builds the full reliability treatment — retries with backoff, resume-from-transcript, heartbeats — on this same foundation.
- **The judgment lives inside the subprocess**, where it belongs. Diagnosing the failure and crafting the fix is the part no script can do; it gets a whole agent, a scoped tool set, and nothing else.

The principle scales down as well as up. Even inside a single-agent harness, the stop condition, the max-turns limit, the tool-result truncation, and the artifact validation are all deterministic orchestration wrapped around one nondeterministic core. And it scales to the fanciest topology in this chapter: the three-agent harness is, at bottom, a deterministic state machine — plan, then loop generate→evaluate until pass or budget — whose states happen to be agent runs. When something goes wrong at 3 a.m., the state machine is the part you can read, test, and trust; design so that as much of the system as possible lives there.

## Choosing a topology

Compressed to a decision procedure, this chapter reads:

1. **Start with one model call.** Add retrieval and examples. Only escalate if outcomes demonstrably fall short.
2. **If the task has fixed, known structure**, use a workflow — chaining for sequences (with gates), routing for categories, sectioning for independent aspects, voting for confidence.
3. **If exploration is bloating a single agent's context**, keep one primary agent and push reads into subagents. Parallelize reads; serialize writes. Budget honestly: multi-agent means roughly 15× chat-level tokens.
4. **If the decomposition itself needs judgment**, promote the orchestrator to a model — and give it explicit effort-scaling rules and a subtask schema (objective, output format, boundaries), or it will misallocate.
5. **If the work is long-running and quality-critical**, separate generation from evaluation — planner/generator/evaluator with contracts agreed before the work — and accept the ~20× cost as the price of output a solo agent cannot reach.
6. **Whatever the topology:** hand off through validated files, keep scheduling and budgets in deterministic code, and re-audit the whole structure at every model generation, because each piece of it encodes an assumption about model weakness that is quietly going stale.

The next chapter takes the component this one kept deferring to — the evaluator — and asks what makes its judgment trustworthy: tests as oracles, rubric design, adversarial verification, and the trust boundaries that keep a clever generator from gaming its own grade.

## Further reading

- Anthropic Engineering, ["Building Effective Agents"](https://www.anthropic.com/engineering/building-effective-agents) — the workflows/agents distinction and the five composable patterns; the primary source for the first half of this chapter.
- Anthropic Engineering, ["Harness design for long-running application development"](https://www.anthropic.com/engineering/harness-design-long-running-apps) — the planner/generator/evaluator harness, sprint contracts, file-based handoffs, self-evaluation bias, and the cost comparison cited here.
- Anthropic Engineering, ["How we built our multi-agent research system"](https://www.anthropic.com/engineering/multi-agent-research-system) — orchestrator-workers in production; token economics (4×/15×), the 90.2% evaluation result, effort-scaling and delegation lessons.
- Cognition, ["Don't Build Multi-Agents"](https://cognition.com/blog/dont-build-multi-agents) — the context-sharing principles and the case for single-threaded agents on tightly coupled work.
- Anthropic, [Claude Code best practices](https://code.claude.com/docs/en/best-practices) — subagents for investigation, writer/reviewer sessions, and the headless fan-out idiom.
- Firecrawl, ["What Is an Agent Harness?"](https://www.firecrawl.dev/blog/what-is-an-agent-harness) — the framework/harness/orchestrator distinction and the initializer-executor pattern.

---

[← Safety, Security, and Permissions](ch08-safety-security-permissions.md) · [Verification and Feedback Loops →](ch10-verification-feedback.md)
