# Chapter 1 — What Is Harness Engineering?

*An agent is a model plus a harness. The model — the neural network that turns a context window into a next token — gets the headlines, the benchmark charts, and the capital expenditure. The harness — everything wrapped around the model that turns its reasoning into dependable work — gets built at 2 a.m. by whoever is on call when the agent deletes the wrong file. This chapter defines the harness precisely, traces the industry's progression from prompt engineering through context engineering to harness engineering, introduces the discipline's central working principle (failures are environmental problems to fix permanently, not prompts to retry), distinguishes frameworks from harnesses from orchestrators, and introduces Ledgerbot, the running example this book returns to in every chapter.*

## The agent is not the model

For years, "agent" was one of the most contested words in software. Every vendor meant something different by it; every conference talk opened with a new definition. Then, in September 2025, Simon Willison — who had spent years publicly refusing to use the term at all — conceded that the industry had finally converged on a definition compact enough to be useful: ["An LLM agent runs tools in a loop to achieve a goal."](https://simonwillison.net/2025/Sep/18/agents/)

Every word of that sentence is doing work, and only one of them names the model. *Runs tools* — something must define those tools, describe them to the model, execute them when the model asks, and decide what the model is allowed to do. *In a loop* — something must implement that loop: assemble the context for each model call, dispatch the model's decisions, append the results, and decide when to go around again. *To achieve a goal* — something must detect that the goal has been achieved, or that it hasn't and the loop should stop anyway. None of that is the model. All of it is the **harness**.

This gives us the equation this book is built on:

> **agent = model + harness**

The model supplies reasoning: given a context window full of instructions, code, logs, and tool results, it produces a decision about what to do next. The harness supplies everything else — and "everything else" turns out to be most of the system. Anthropic's engineering team, describing the distinction between simple and agentic systems, defines agents as ["systems where LLMs dynamically direct their own processes and tool usage, maintaining control over how they accomplish tasks"](https://www.anthropic.com/engineering/building-effective-agents) — as opposed to workflows, "systems where LLMs and tools are orchestrated through predefined code paths." In both cases, the LLM never touches a file, never runs a command, never sends a request. It emits structured text saying it *wants* to. The harness is the layer that makes those wants real, safely, and reports back what happened.

The consequence, borne out repeatedly in production, is that when an agent fails, the fault usually lives in the harness, not the model. The model didn't know the test command for this repository — a context-delivery failure. The model retried a network call forty times — a missing loop budget. The model edited a generated file that gets overwritten on build — a missing environmental guardrail. The model claimed success without running the tests — a missing verification step. The same model that flails inside a poorly built harness performs impressively inside a well-built one. That observation is the founding premise of harness engineering, and of this book.

## Defining the harness

Throughout this book, the **harness** is all software around the model: the loop, tools, context assembly, memory, sandbox, permissions, verification, and observability. It is worth walking through each of those, because together they form the table of contents.

The **agent loop** is the core cycle: assemble context → call the model → execute its decision → append the results → repeat until a stop condition. This is the direct descendant of the reason-and-act cycle formalized in the [ReAct paper](https://arxiv.org/abs/2210.03629) (Yao et al., 2022), which showed that interleaving reasoning traces with actions against an environment outperforms either pure reasoning or pure acting. Chapter 3 builds a complete loop from scratch and dissects it.

**Tools** are the functions the model can invoke, each described by a schema. Tool design is user-experience design where the user is a model — what SWE-agent's authors and Anthropic both call the **ACI**, the agent-computer interface. Chapter 4 treats it in depth.

**Context assembly** — deciding what enters each model call — is what Anthropic defines as **context engineering**: ["the set of strategies for curating and maintaining the optimal set of tokens (information) during LLM inference."](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) The context window is a scarce, degrading resource; models have an "attention budget" that long contexts exhaust. Chapter 5 covers budgets, **compaction** (summarizing history in place), and **reset** (clearing it).

**Memory and state** span three horizons: the working context of the current call, the **session** (one continuous run of the loop, possibly resumed), and long-term memory across sessions. The Firecrawl explainer on agent harnesses puts it bluntly: LLMs are stateless by default, so ["memory isn't a plugin, it's the harness."](https://www.firecrawl.dev/blog/what-is-an-agent-harness) Chapter 6 covers this.

The **sandbox** is where the agent's actions actually run — containers, microVMs, remote execution environments — and **permissions** govern what it may do there. Chapters 7 and 8 argue that isolation, not model alignment, is the load-bearing safety mechanism.

**Verification** is any **oracle** — a mechanism that tells the harness whether an action worked: a test suite, a type checker, a second model grading in a fresh context. Chapter 10 makes the case that letting the agent see whether it worked is the single highest-leverage harness feature.

**Observability** — transcripts, per-step logs, traces — is how humans debug all of the above. Chapter 11 treats harnesses as the distributed systems they are.

The Firecrawl piece, one of the earlier attempts to define the term crisply, describes the harness as ["the software infrastructure surrounding an AI model that manages everything except the model's actual reasoning,"](https://www.firecrawl.dev/blog/what-is-an-agent-harness) and identifies five core subsystems: the tool integration layer, memory and state management, context engineering and compression, verification and guardrails, and session management. The community-curated [awesome-harness-engineering](https://github.com/ai-boost/awesome-harness-engineering) list expands the same territory into a dozen primitives — the agent loop, planning, context delivery, tool design, skills and protocols, permissions, memory, orchestration, verification, observability, debugging, and human-in-the-loop design. The taxonomies differ at the edges; the center is stable. Everything except the forward pass is harness.

One clarification before we go further, because the terms get conflated constantly. A **turn** is one user-visible request/response exchange. A **step** is one model call plus the execution of whatever tools it requested, inside the loop. A single turn of a coding agent — "fix the failing build" — may comprise eighty steps. When this book says a harness "bounds the loop," it means it bounds steps; when it says a session resumes, it means the sequence of turns continues with state intact.

## Meet Ledgerbot

Abstractions about harnesses go down easier with a concrete system to hang them on, so let us introduce the running example that recurs throughout this book.

**Ledgerbot** is an agent that triages and fixes failing CI builds in a mid-size Python monorepo — roughly 900,000 lines of code across a few dozen packages, owned by a platform team at a fictional company. When a build on the main branch goes red, Ledgerbot wakes up. It reads the CI logs, forms a hypothesis about the failure, checks out the offending commit in an isolated workspace, edits code, runs the relevant tests, and — if the tests pass — opens a pull request with an explanation of the root cause and the fix. A human merges it, or doesn't.

Ledgerbot is fictional, but nothing about it is fanciful; it is a composite of the CI-repair and issue-fixing agents that teams began deploying in earnest through 2025 and 2026, built on the same foundations as public coding agents like Claude Code, Codex CLI, and OpenHands. It is chosen as the running example because it exercises every harness concern this book covers, and it is worth previewing how:

- **Tools.** Ledgerbot needs to read files, search the repo, run shell commands, query the CI system's API, and open pull requests. Each of those is a tool with a schema, a permission profile, and a failure mode.
- **Sandboxing.** Ledgerbot executes arbitrary code from a repo that hundreds of engineers commit to. It runs in a container with no production credentials and no network route to anything but the package mirror and the CI API.
- **Context limits.** A failing CI run can emit a 40,000-line log. Ledgerbot's harness cannot simply paste that into the context window; it must filter, summarize, or let the agent retrieve slices on demand.
- **Verification.** "The tests pass" is Ledgerbot's oracle. Without it, Ledgerbot is a machine for generating plausible-looking pull requests. With it, every claim of success is backed by an execution the harness observed directly.
- **Memory.** The same flaky test breaks the build every few weeks. Ledgerbot should not rediscover that fact from scratch each time; its harness persists notes across sessions.
- **Cost and reliability.** Ledgerbot runs dozens of times a day. Its harness caches context, retries rate-limited API calls with backoff, bounds each session's steps, and writes a transcript of every run.
- **Safety.** Ledgerbot must never push to main, never touch deployment configuration, and never act on instructions embedded in a commit message by a malicious contributor. Those are permission and trust-boundary problems, not prompting problems.

A fragment of Ledgerbot's tool layer looks like this — the full loop is built in Chapter 3, and the design rationale for schemas like this one occupies Chapter 4:

```python
# Fragment: one entry in Ledgerbot's tool list (Anthropic Messages API shape).
RUN_TESTS = {
    "name": "run_tests",
    "description": (
        "Run pytest for one package in the monorepo and return the last "
        "200 lines of output. Always prefer this over invoking pytest "
        "via the shell: it selects the correct virtualenv and test flags "
        "for the package automatically."
    ),
    "input_schema": {
        "type": "object",
        "properties": {
            "package": {
                "type": "string",
                "description": "Package directory relative to repo root, "
                               "e.g. 'services/billing'.",
            },
            "test_expr": {
                "type": "string",
                "description": "Optional pytest -k expression to select "
                               "a subset of tests.",
            },
        },
        "required": ["package"],
    },
}
```

Notice what this fragment already reveals about harness thinking. The tool does not expose raw shell access for testing; it encodes the repository's conventions (virtualenv selection, standard flags) so the model cannot get them wrong. It truncates output to 200 lines because tool results consume the same scarce context window as everything else. And its description teaches the model when to use it, in prose written the way you would brief a new teammate. None of this intelligence lives in the model. All of it is harness.

## From prompt engineering to loop engineering

Harness engineering did not appear from nowhere; it is the third stage of a progression the field walked in about four years, with each stage widening the aperture of what engineers considered theirs to design.

**Prompt engineering** came first: the craft of writing instructions that elicit good behavior from a single model call. It was — and remains — genuinely useful, but it operates on one artifact (the prompt) for one transaction (the call). Its limits showed as soon as tasks stopped fitting in one call.

**Context engineering** widened the aperture from the prompt to everything that enters the model's window: system prompts, retrieved documents, tool results, conversation history, instruction files. The insight, as [Anthropic's engineering team frames it](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents), is that context is a finite resource with a real attention budget — transformer attention costs grow quadratically, and empirically model performance degrades as contexts lengthen, a phenomenon practitioners came to call *context rot*. Curating what the model sees matters as much as phrasing it.

**Harness engineering** widens the aperture again, from what the model sees to the entire environment the model operates in — and, critically, to the *loop* rather than the call. The term crystallized in early 2026. Mitchell Hashimoto — HashiCorp co-founder, and by then deep into building software primarily with agents — described in [his February 2026 essay on AI adoption](https://mitchellh.com/writing/my-ai-adoption-journey) a habit he had fallen into and named "harness engineering": *anytime you find an agent makes a mistake, you take the time to engineer a solution such that the agent never makes that mistake again.* Within weeks the term was everywhere: [Martin Fowler's site](https://martinfowler.com/articles/harness-engineering.html) published a practitioner's treatment defining harness engineering as everything in the agentic system except the model itself, and industry analyses like [Faros's](https://www.faros.ai/blog/harness-engineering) were describing it as "the discipline of building the environment around AI models to transform them into reliable, autonomous agents" — the successor phase to prompt and context engineering as the primary focus of engineering investment.

The progression is easy to state as a table of questions. Prompt engineering asks: *what do I say to the model?* Context engineering asks: *what does the model get to see?* Harness engineering asks: *what world does the model act in — what can it do, what can it observe, what stops it, and what tells it whether it worked?*

Loop engineering — a phrase this book uses for the harness-engineering activities specific to the agent loop itself — is where the divergence from prompt engineering is sharpest. A prompt engineer facing a misbehaving agent edits words and reruns. A loop engineer asks structural questions: Is the stop condition right? Should this tool result be truncated, or the whole history compacted? Should this step be a subagent with a fresh context? Should the harness inject a reminder every N steps instead of hoping the system prompt survives attention decay? The prompt is one input among many; the loop is the machine.

## Failures are environmental problems, not prompts to retry

Hashimoto's principle deserves its own section, because it is the closest thing the discipline has to a founding maxim, and because it inverts the reflex most engineers bring from their first weeks of working with LLMs.

The reflex is: the agent did something wrong, so I'll tell it not to do that, and try again. Add a line to the prompt. Rephrase. Re-roll. Sometimes this works, in the way that asking a colleague to "please be more careful" sometimes works. But it is probabilistic compliance, and it compounds badly: prompts accrete warnings ("do NOT edit generated files," "ALWAYS run the linter," "NEVER use `--force`") until the instructions themselves rot the context, and the same failures recur anyway at some background rate. Because agents run in loops, per-step error rates compound across steps — an agent that follows a soft instruction 98% of the time, across an eighty-step session, gets a coin flip on perfection.

The harness engineering move is different: treat every agent mistake as a bug report against the environment, and fix the environment so the mistake becomes impossible, detected, or self-correcting — in that order of preference.

- **Impossible.** Ledgerbot once opened a pull request against a release branch. The prompt fix is "only open PRs against main." The harness fix is that the `open_pr` tool has no branch parameter at all; the harness supplies it. The mistake is now structurally unavailable, in the spirit of *poka-yoke* — the mistake-proofing principle from manufacturing that [Anthropic's tool-design guidance](https://www.anthropic.com/engineering/building-effective-agents) explicitly borrows, discussed fully in Chapter 4.
- **Detected.** Ledgerbot occasionally "fixes" a build by weakening an assertion. No prompt phrasing reliably prevents this, because from inside the loop it looks like success. The harness fix is an oracle outside the loop: a verification step that runs the full affected test suite in a clean workspace and blocks the PR if coverage of the changed lines regressed. This is Martin Fowler's distinction between [*guides* and *sensors*](https://martinfowler.com/articles/harness-engineering.html) — feedforward controls that shape behavior before it happens, and feedback controls that observe outcomes and force correction. Telling the agent to follow coding standards is a guide; a linter wired to block the merge is a sensor. Guides are probabilistic; sensors are deterministic.
- **Self-correcting.** Ledgerbot sometimes guesses a wrong module path. Rather than pre-loading the entire repo map into context (expensive, stale), the harness makes the failure cheap and informative: the `read_file` tool's error message for a missing path returns the nearest matching paths. The agent recovers in one step. Error messages that teach are harness engineering too.

The discipline this produces looks a lot like operating a production service. You do not fix an outage by asking the service nicely; you write the postmortem, and the action items change the system. A harness accumulates these fixes the way an SRE team accumulates runbooks and alerts — which is why teams that run agents seriously keep transcripts of every session (Chapter 11) and treat each surprising transcript as free failure data. The payoff also compounds: every environmental fix benefits every future session of every agent in that environment, which is precisely what a prompt tweak in one engineer's chat window does not do.

There is one caveat, and it is important enough that Chapter 14 is devoted to it. Every harness component you add encodes an assumption about what the model cannot do on its own — and those assumptions go stale. Anthropic's team, describing a harness for long-running application development, found that scaffolding built to compensate for one model generation's weaknesses became [unnecessary overhead when the next generation arrived](https://www.anthropic.com/engineering/harness-design-long-running-apps), and that every component is "worth stress testing" against each new model. Fix failures permanently, but date-stamp your assumptions.

## Framework, harness, orchestrator

Three words get used interchangeably in this space and should not be, because they name three different layers of responsibility. Following the [Firecrawl taxonomy](https://www.firecrawl.dev/blog/what-is-an-agent-harness), this book uses them as follows.

A **framework** is a library of abstractions for building agents: LangGraph, LlamaIndex, the OpenAI Agents SDK, the Anthropic Agent SDK's lower-level primitives. A framework gives you components — a graph runner, a message type, a tool decorator — and no opinions about what your agent should actually do with them. You cannot deploy a framework; you build with one.

A **harness** is a running system assembled from such components (or from scratch), with defaults chosen and wired together: *this* loop, *these* tools, *this* sandbox, *this* permission policy, *this* transcript format. Claude Code is a harness. Codex CLI is a harness. OpenHands is a harness. Ledgerbot is a harness. When you type a task into Claude Code, hundreds of design decisions you never see — how file edits are validated, how much of a long tool output survives into context, when the session compacts — execute on your behalf. Those decisions *are* the harness, and they differ meaningfully between products even when the underlying model is identical. This is why the same model scores differently on the same benchmark under different harnesses, a point Chapter 12 returns to: agent benchmark results are always model-plus-harness results.

An **orchestrator** is code — or an agent — that sequences other agents: fan out three subagents to investigate in parallel, run a generator until an evaluator approves, route easy tasks to a cheap model. Orchestration lives a level above any single agent's loop. Anthropic's catalog of composable patterns — prompt chaining, routing, parallelization, orchestrator-workers, evaluator-optimizer — is [a catalog of orchestration shapes](https://www.anthropic.com/engineering/building-effective-agents), and Chapter 9 works through them. The important boundary note: a workflow with predefined code paths and an autonomous agent directing its own process are ends of a spectrum, and a good orchestrator is often deliberately boring, deterministic code wrapped around nondeterministic agents.

The layering matters practically because responsibilities attach to layers. "Which tools exist and what do their error messages say" is a harness question. "Should this be one agent or three" is an orchestrator question. "Which graph library" is a framework question — and, candidly, the least consequential of the three. Anthropic's guidance here is blunt and this book endorses it: ["Success in the LLM space isn't about building the most sophisticated system. It's about building the *right* system for your needs"](https://www.anthropic.com/engineering/building-effective-agents) — start with direct API calls, add framework only when you feel the pull.

## The shape of the discipline

It is fair to ask whether "harness engineering" is a genuinely new discipline or a rebranding of software engineering with a stochastic component in the middle. The honest answer is: both, and the overlap is the point.

Much of what a harness engineer does is recognizable systems work. Retries with exponential backoff, idempotent operations, resource limits, audit logs, trust boundaries, staged rollouts of configuration changes — Chapters 7, 8, and 11 will feel familiar to anyone who has run production infrastructure, because a harness *is* production infrastructure whose most important dependency happens to be nondeterministic. The discipline's older siblings run deeper still: as Chapter 2 shows, test harnesses, fuzzing harnesses, and evaluation harnesses spent five decades developing exactly the concepts — oracles, isolation, reproducibility, the harness as experiment apparatus — that agent builders now need daily.

What is genuinely new is the character of the component in the middle. The model is a dependency that improves every few months, behaves differently across providers and versions, responds to the *shape* of its inputs in ways no API contract captures, and can be persuaded by data it processes — which makes prompt injection a security class with no precedent in classical systems (Chapter 8). Designing for a dependency like that produces the discipline's distinctive concerns: the ACI, context budgets, verification in fresh contexts to defeat self-evaluation bias, and the standing obligation to re-audit the harness whenever the model improves.

The field's own map of itself is already stable enough to navigate by. The [awesome-harness-engineering](https://github.com/ai-boost/awesome-harness-engineering) list's twelve primitives correspond, almost one to one, to this book's chapter table: the loop (Chapter 3), tools (Chapter 4), context delivery (Chapter 5), memory (Chapter 6), sandboxes (Chapter 7), permissions and human-in-the-loop (Chapter 8), orchestration (Chapter 9), verification (Chapter 10), observability and debugging (Chapter 11), and evaluation (Chapter 12). Chapter 13 dissects public harnesses — Claude Code, Codex CLI, SWE-agent, OpenHands — as case studies, and Chapter 14 confronts the discipline's strange teleology: a field whose components are each built in the hope of becoming obsolete.

That teleology is worth naming before we proceed, because it flavors everything. Harness components exist because the model can't do something alone — and models keep learning to do things alone. The parts of the harness that survive model improvement are the ones that solve *trust* problems rather than *capability* problems: sandboxing, permissions, verification, audit. You will still want the tests to actually run, in an environment the agent cannot tamper with, no matter how smart the model gets. The parts that erode are the capability crutches: rigid task decomposition, elaborate compaction tricks, hand-holding prompts. A good harness engineer builds both kinds, labels which is which, and feels no sentimentality when deleting the second kind.

Where does that leave Ledgerbot — and you? By the end of this book you should be able to build Ledgerbot: a working single-agent harness from scratch (Chapter 3), a tool layer the model uses correctly on the first try (Chapter 4), context and memory design that keeps a multi-hour session coherent (Chapters 5 and 6), sandboxing and permissions you can defend in a security review (Chapters 7 and 8), orchestration that stays deterministic where determinism matters (Chapter 9), and the verification and evaluation machinery to know whether any of it works (Chapters 10 and 12). First, though, it pays to meet the harness's ancestors. The word has fifty years of history, and the traditions behind it — test harnesses, fuzzing harnesses, eval harnesses — solved versions of every problem in this book before "agent" meant anything at all. That history is Chapter 2.

## Further reading

- Simon Willison, ["I think 'agent' may finally have a widely enough agreed upon definition to be useful jargon now"](https://simonwillison.net/2025/Sep/18/agents/) — the "runs tools in a loop to achieve a goal" definition.
- Anthropic Engineering, ["Building Effective Agents"](https://www.anthropic.com/engineering/building-effective-agents) — workflows vs. agents, the composable patterns, ACI design, and the case for simplicity.
- Anthropic Engineering, ["Effective context engineering for AI agents"](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) — context as a finite resource; the attention budget; compaction and retrieval strategies.
- Anthropic Engineering, ["Harness design for long-running application development"](https://www.anthropic.com/engineering/harness-design-long-running-apps) — a production three-agent harness, and how its components went stale as models improved.
- Firecrawl, ["What Is an Agent Harness?"](https://www.firecrawl.dev/blog/what-is-an-agent-harness) — the five-component anatomy and the framework/harness/orchestrator distinction.
- Mitchell Hashimoto, ["My AI Adoption Journey"](https://mitchellh.com/writing/my-ai-adoption-journey) — the essay that named harness engineering's working principle.
- Martin Fowler (site), ["Harness engineering for coding agent users"](https://martinfowler.com/articles/harness-engineering.html) — guides vs. sensors; regulating maintainability, architecture, and behavior.
- Faros AI, ["Harness Engineering: Making AI Coding Agents Work in 2026"](https://www.faros.ai/blog/harness-engineering) — the discipline as the third phase of AI engineering maturity.
- [awesome-harness-engineering](https://github.com/ai-boost/awesome-harness-engineering) — a curated map of the field's primitives.
- Shunyu Yao et al., ["ReAct: Synergizing Reasoning and Acting in Language Models"](https://arxiv.org/abs/2210.03629) — the research lineage of the agent loop.

---

[← Preface](00-preface.md) · [Lineage: Test, Fuzz, and Eval Harnesses →](ch02-lineage.md)
