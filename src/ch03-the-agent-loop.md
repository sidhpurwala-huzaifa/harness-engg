# Chapter 3 — The Agent Loop

*Every agent, from a weekend prototype to a production coding assistant, is built around the same four-beat cycle: assemble context, call the model, execute what the model decided, append the results, and repeat until some stop condition fires. This chapter builds that cycle from nothing — a complete, runnable harness in roughly a hundred lines of Python against the real Anthropic API — and then dissects it line by line. Along the way it fixes the vocabulary the rest of the book depends on (turn, step, session), traces the loop's intellectual lineage back to the ReAct paper, covers stop conditions, streaming, and parallel tool calls, and marks the exact point where loop engineering stops being prompt engineering and becomes systems engineering.*

## The cycle at the core

Strip away the product marketing, the frameworks, and the org charts of multi-agent systems, and what remains is startlingly small. Simon Willison, after years of collecting competing definitions — at one point he gathered 211 of them from his Twitter followers — settled on [a single sentence](https://simonwillison.net/2025/Sep/18/agents/): "An LLM agent runs tools in a loop to achieve a goal." Every word is load-bearing. *Tools*, because a bare language model can only emit text; tools are how its decisions touch the world. *In a loop*, because a single call with tools attached is just a fancy function router — the agent quality emerges when the model sees the result of its last action before choosing its next one. And *to achieve a goal*, because the loop is not infinite: it has stop conditions, and deciding what those are is a design act.

Anthropic's [Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents) draws the same boundary from the other direction. It distinguishes **workflows** — "LLMs and tools orchestrated through predefined code paths" — from **agents**, systems where "LLMs dynamically direct their own processes and tool usage." In a workflow, your code decides what happens next and the model fills in the blanks. In an agent, the model decides what happens next and your code executes it. The essay describes how agents operate once launched: they "plan and operate independently," obtaining "ground truth from the environment at each step (such as tool call results or code execution)," with "stopping conditions (such as a maximum number of iterations) to maintain control."

That inversion of control is precisely what makes the harness matter. When your code owns the control flow, bugs live in your branches and you debug them with a stack trace. When the model owns the control flow, the only levers you have are environmental: what the model sees (context), what it can do (tools), what happens when its actions fail (error routing), and when the whole thing halts (stop conditions). The agent loop is where all four levers attach. Using this book's glossary: the **agent loop** is the assemble-context → model call → act → append cycle, and the **harness** is everything wrapped around it.

This chapter builds that loop for Ledgerbot, this book's running example — an agent that triages and fixes failing CI builds in a mid-size Python monorepo. Chapter 1 introduced Ledgerbot in full; here we need only its skeleton: it reads files, edits code, and runs tests. Those three verbs are enough to exercise every part of the loop.

## The ReAct heritage

The loop did not appear fully formed in a vendor SDK. Its direct ancestor is the ReAct pattern from ["ReAct: Synergizing Reasoning and Acting in Language Models"](https://arxiv.org/abs/2210.03629) (Yao et al., 2022). Before ReAct, the literature treated reasoning and acting as separate capabilities: chain-of-thought prompting let models reason but gave them no way to check their reasoning against the world, while action-generation approaches let models act but without a deliberative trace. ReAct's contribution was to interleave them — the model emits "both reasoning traces and task-specific actions in an interleaved manner," so reasoning steps update the plan and action steps pull in fresh information from the environment.

The empirical case was strong. On HotpotQA and Fever, giving the model a Wikipedia-lookup action reduced the hallucination and error propagation that plagued pure chain-of-thought. On the interactive benchmarks ALFWorld and WebShop, ReAct beat imitation-learning and reinforcement-learning baselines by 34 and 10 absolute percentage points respectively, using only one or two in-context examples. The pattern in a ReAct transcript — *Thought → Action → Observation*, repeated — is the agent loop written out longhand, back when the harness had to parse the action out of free text with a regular expression.

What has changed since 2022 is that the pattern moved into the API layer. Modern model APIs accept structured tool definitions and return structured tool calls: the model signals "I want to act" through a typed content block and a dedicated stop reason rather than through text your harness must parse. The "Thought" part became native reasoning (extended thinking, in Anthropic's API); the "Action" part became the `tool_use` block; the "Observation" part became the `tool_result` block your harness sends back. The intellectual structure is unchanged. When you write an agent loop today you are writing a ReAct interpreter whose action grammar the model provider has standardized for you.

## Vocabulary: turn, step, session

Loose terminology causes real bugs — a "retry the turn" instruction means something very different depending on what a turn is — so this book fixes three words and uses them consistently.

A **step** is one model call plus the execution of whatever that call decided: one trip around the loop. A step that ends in tool calls produces tool results and another step; a step that ends in plain text usually ends the loop.

A **turn** is one user-visible request/response exchange. A single turn of Ledgerbot ("the build is red, fix it" → "fixed, here's what changed") may contain forty steps. Users see turns; harnesses see steps. Most observability failures in agent systems come from logging at turn granularity when the interesting events happen at step granularity — Chapter 11 returns to this.

A **session** is one continuous run of the loop, possibly spanning many turns, with its accumulated message history. Sessions may be suspended and resumed; Chapter 6 covers the state machinery that makes resumption safe.

These are harness-side words, deliberately distinct from API-side words. The Anthropic and OpenAI APIs are stateless: each model call receives the full message history and returns one assistant message. "Step" describes what your loop does with that call, not anything the API itself tracks.

## A complete minimal harness

Here is the whole thing. This is not pseudocode and it is not a fragment: with Python 3.11+, `pip install anthropic`, and an `ANTHROPIC_API_KEY` in the environment, it runs. It gives the model three tools — read a file, write a file, run the tests — and loops until the model stops asking to act. The model ID is pinned to a current one at the time of writing; check your provider's model list and substitute the current generation, because model IDs age faster than book chapters.

```python
"""ledgerbot_mini.py — a complete minimal agent harness.

Usage:  python ledgerbot_mini.py "The CI build is failing. Find out why and fix it."
Needs:  pip install anthropic   and   ANTHROPIC_API_KEY in the environment.
Run it from the root of the repository you want it to work on.
"""
import json
import subprocess
import sys

import anthropic

MODEL = "claude-opus-4-8"   # pin a current model; swap as generations change
MAX_STEPS = 20              # hard ceiling on trips around the loop

SYSTEM = (
    "You are Ledgerbot, an agent that triages failing CI builds in a Python "
    "repository. Investigate with your tools, make the smallest change that "
    "turns the tests green, then report what you changed and why. Run the "
    "tests to confirm before you declare success."
)

TOOLS = [
    {
        "name": "read_file",
        "description": (
            "Read a UTF-8 text file and return its contents with line numbers. "
            "Always read a file before editing it. Returns an error if the "
            "path does not exist."
        ),
        "input_schema": {
            "type": "object",
            "properties": {
                "path": {
                    "type": "string",
                    "description": "Path relative to the repository root, "
                                   "e.g. 'src/billing/invoice.py'.",
                },
            },
            "required": ["path"],
        },
    },
    {
        "name": "write_file",
        "description": (
            "Replace the entire contents of a file (creating it if absent). "
            "Include the complete file, not a fragment or a diff."
        ),
        "input_schema": {
            "type": "object",
            "properties": {
                "path": {"type": "string",
                         "description": "Path relative to the repository root."},
                "content": {"type": "string",
                            "description": "The complete new file contents."},
            },
            "required": ["path", "content"],
        },
    },
    {
        "name": "run_tests",
        "description": (
            "Run the test suite with pytest and return the exit code plus the "
            "last 50 lines of output. Pass test_path to run a subset, "
            "e.g. 'tests/test_invoice.py' — prefer that while iterating."
        ),
        "input_schema": {
            "type": "object",
            "properties": {
                "test_path": {"type": "string",
                              "description": "Optional pytest target."},
            },
            "required": [],
        },
    },
]

def read_file(path: str) -> str:
    with open(path, encoding="utf-8") as f:
        lines = f.readlines()
    return "".join(f"{n:5d}| {line}" for n, line in enumerate(lines, 1)) or "(empty file)"

def write_file(path: str, content: str) -> str:
    with open(path, "w", encoding="utf-8") as f:
        f.write(content)
    return f"Wrote {len(content)} characters to {path}."

def run_tests(test_path: str = "") -> str:
    cmd = ["python", "-m", "pytest", "-x", "-q", "--no-header"]
    if test_path:
        cmd.append(test_path)
    proc = subprocess.run(cmd, capture_output=True, text=True, timeout=300)
    tail = "\n".join((proc.stdout + proc.stderr).splitlines()[-50:])
    return f"exit code: {proc.returncode}\n{tail}"

HANDLERS = {"read_file": read_file, "write_file": write_file, "run_tests": run_tests}

def execute(name: str, args: dict) -> tuple[str, bool]:
    """Run one tool call. Returns (output, is_error) — never raises."""
    try:
        return HANDLERS[name](**args), False
    except Exception as exc:
        return f"{type(exc).__name__}: {exc}", True

def run(task: str) -> str:
    client = anthropic.Anthropic()
    messages = [{"role": "user", "content": task}]
    for step in range(MAX_STEPS):
        response = client.messages.create(
            model=MODEL,
            max_tokens=16000,
            system=SYSTEM,
            thinking={"type": "adaptive"},
            tools=TOOLS,
            messages=messages,
        )
        messages.append({"role": "assistant", "content": response.content})
        if response.stop_reason != "tool_use":          # the model is done (or stuck)
            return next((b.text for b in response.content if b.type == "text"),
                        f"(ended with stop_reason={response.stop_reason})")
        results = []
        for block in response.content:
            if block.type == "tool_use":
                print(f"[step {step}] {block.name} "
                      f"{json.dumps(block.input)[:120]}", file=sys.stderr)
                output, is_error = execute(block.name, block.input)
                results.append({"type": "tool_result", "tool_use_id": block.id,
                                "content": output, "is_error": is_error})
        messages.append({"role": "user", "content": results})
    return "Stopped: hit MAX_STEPS before the model finished."

if __name__ == "__main__":
    task = sys.argv[1] if len(sys.argv) > 1 else \
        "The CI build is failing. Find out why and fix it."
    print(run(task))
```

That is a working agent. Point it at a repository with a failing test and it will read the traceback, open the offending file, edit it, and re-run the suite until it passes or runs out of steps. Thorsten Ball made the same point with a Go version in his widely shared essay [How to Build an Agent](https://ampcode.com/how-to-build-an-agent): "It's not that hard to build a fully functioning, code-editing agent... It's an LLM, a loop, and enough tokens." His implementation ran under 400 lines, "most of which is boilerplate." The research community reached the same conclusion independently: [mini-swe-agent](https://github.com/SWE-agent/mini-swe-agent), from the Princeton and Stanford team behind SWE-bench, packs its entire agent class into about 100 lines of Python and still "scores >74% on the SWE-bench verified benchmark." The capability is in the model. The loop's job is not to add intelligence; it is to deliver the environment to the model reliably, observably, and safely — which is why the rest of this book exists.

Now take it apart.

## Dissection

**The constants.** `MODEL` is a runtime choice, not an architectural one — a well-built harness treats the model as a swappable component, because Chapter 14 will argue you should re-audit the whole harness every time it changes. `MAX_STEPS` is the crudest and most important safety property in the file: it converts "the model got confused" from an infinite-cost event into a bounded-cost event. Twenty is a placeholder; production harnesses set this per task class and often layer token and wall-clock budgets on top.

**The system prompt.** Note what it contains and what it doesn't. It states the role, the objective, the bias ("smallest change"), and one procedural requirement: verify with tests before declaring success. That last sentence is the seed of everything Chapter 10 says about verification — it wires an oracle (the test suite) into the model's own definition of done. What the prompt does *not* contain is a step-by-step procedure. The model owns the control flow; over-scripting it in the prompt fights the premise of building an agent at all.

**The tool definitions.** Each tool is a name, a natural-language description, and a JSON Schema for its inputs — the exact wire format of the [Anthropic tool-use API](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview). These definitions are rendered into the model's context on every single step, which makes them the most-read documentation in your entire system. Notice that the descriptions carry behavioral guidance, not just semantics: "Always read a file before editing it," "prefer [a subset] while iterating." The model treats tool descriptions as instructions with roughly system-prompt authority. Chapter 4 is entirely about getting this layer right; for now, observe that even a hundred-line harness cannot avoid ACI design — it can only do it carelessly or deliberately.

**The handlers.** `read_file`, `write_file`, and `run_tests` are ordinary Python functions, and two details in them are quiet acts of loop engineering. `read_file` returns line numbers, because the model will reference lines when reasoning about the traceback. `run_tests` returns the exit code plus only the last fifty lines of output — a deliberate truncation, because a full pytest log for a large suite can be tens of thousands of tokens, and every one of those tokens is resent on every subsequent step. Tool output is not a return value; it is a permanent deposit into a shared, expensive memory. Chapter 5 does the full token math.

**`execute` — errors become observations.** The wrapper converts every exception into a `(message, is_error=True)` pair instead of letting it propagate. This is one of the deepest ideas in harness design hiding in six lines. In ordinary software, an exception is a control-flow event: the caller unwinds. In an agent, a tool failure is *information* — the model asked to read a file that doesn't exist, and the most useful thing the harness can do is tell it so and let it adapt. The API supports this natively: a `tool_result` block carries an optional `is_error: true` flag, and models are trained to acknowledge flagged errors and try another approach. A harness that crashes on the model's first malformed path has confused its own failures with the agent's. (Genuine harness failures — the API is down, the sandbox died — are a different category and *should* halt the loop; Chapter 11 draws that boundary.)

**The loop body: assemble, call, act, append.** The `messages` list is the loop's entire memory, and the pattern here is what mini-swe-agent's authors call "a completely linear history — every step of the agent just appends to the messages that are passed to the LM in the next step." Assembly is trivial in this harness — the history *is* the context — which is exactly what makes the minimal version minimal. Everything Chapter 5 covers (compaction, retrieval, budget management) is elaboration of this one line: `messages=messages`.

Two subtleties in the append logic repay attention. First, the harness appends `response.content` *in its entirety* — text blocks, `tool_use` blocks, and any `thinking` blocks the adaptive-reasoning setting produced — rather than extracting just the text. The API requires the `tool_use` blocks back so it can match them to results, and reasoning blocks must round-trip intact for the model to continue coherently. Cherry-picking blocks out of an assistant message is one of the most common beginner bugs in loop code. Second, tool results are sent as a *user*-role message. There is no "tool" role in this API; results are content blocks inside a user message, each carrying the `tool_use_id` that ties it to the request. The transcript alternates assistant/user all the way down, which is also what makes transcripts replayable — a property Chapter 11 leans on hard.

**The stderr line.** One `print` per tool call, to stderr, before execution. This is the embryo of observability: a human tailing the process sees what the agent is doing in real time, and the primary output stream stays clean. Production harnesses replace this with structured, per-step logs, but the principle — *every action legible as it happens* — starts here.

## Stop conditions

A loop is defined as much by how it ends as by what it repeats. This harness ends four ways, and the taxonomy generalizes.

**The model stops itself.** When the response contains no tool calls, `stop_reason` is `end_turn` and the loop returns the final text. This is the happy path, and it is worth appreciating how much weight it carries: the model, not the harness, decides the task is done. That judgment is fallible in both directions — models declare victory early, and models keep polishing after the job is finished — which is why serious harnesses pair the model's self-declared stop with an external oracle (does the test suite actually pass?) before accepting it. Chapter 10 treats this as the single highest-leverage harness feature.

**The output cap fires.** `stop_reason: "max_tokens"` means the response was truncated mid-thought. The minimal harness surfaces it and exits; a production harness treats it as a retryable condition — raise the cap, or switch to streaming, or in the worst case re-prompt for a continuation. Silently treating a truncated message as a finished one corrupts the transcript in ways that only show up several steps later.

**The model declines.** Current-generation APIs can return a `refusal` stop reason when safety systems decline the request. A harness must handle it as a terminal outcome for that turn — not retry it in a tight loop, which burns budget and looks adversarial in the logs.

**The budget fires.** `MAX_STEPS` is a resource stop: nothing about the task ended, the harness simply refused to spend more. Production harnesses stack several — step count, cumulative token spend, wall-clock deadline — and, critically, make the budget stop *legible in the result*. "Stopped: hit MAX_STEPS" is honest; returning the last assistant text as if it were an answer is a lie your users will eventually catch. A refinement worth knowing: some APIs let you *tell the model* its budget so it can pace itself and wrap up gracefully, which converts a hard truncation into a planned landing.

There is a fifth stop the minimal harness omits: the **human interrupt**. Interactive harnesses like the terminal coding agents let a user pause the loop between steps, redirect the agent, and resume. Architecturally this is easy precisely because the loop is synchronous and the state is one list — the pause point is the top of the `for` loop.

## Streaming

The minimal harness blocks on `messages.create()` and gets the whole response at once. That is fine for a demo and wrong for production, for two compounding reasons. First, user experience: an agent that thinks for ninety seconds and then dumps everything reads as broken; the same agent streaming its reasoning and actions reads as working. Second, plumbing: long generations over a silent HTTP connection are exactly where load balancers, proxies, and client libraries impose idle timeouts. The Anthropic SDK actually refuses very large `max_tokens` values on non-streaming calls for this reason. Its [streaming interface](https://platform.claude.com/docs/en/build-with-claude/streaming) delivers server-sent events — message start, per-block deltas, block stop — and provides an accumulator so you can render tokens as they arrive and still collect the finished message:

```python
with client.messages.stream(model=MODEL, max_tokens=16000, system=SYSTEM,
                            tools=TOOLS, messages=messages) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)      # live progress for the human
    response = stream.get_final_message()    # identical object to the blocking call
```

The essential insight: streaming changes the *transport*, not the *loop*. `get_final_message()` returns the same object the blocking call would have, and the rest of the iteration — append content, check stop reason, execute tools — proceeds unchanged. Design the loop against complete messages and treat streaming as a presentation-layer concern, and you get both a clean architecture and a live UI. One wrinkle worth noting: tool call arguments stream too, as partial JSON deltas. A harness that wants to display "Ledgerbot is opening `src/billing/invoice.py`..." before the block finishes must accumulate those deltas; the SDK helpers do this for you.

## Parallel tool calls

A single assistant message may contain *several* `tool_use` blocks. Ask Ledgerbot to investigate a failure touching three modules, and a capable model will often request three `read_file` calls in one step rather than three sequential steps — a 3× saving in both latency and repeated-context cost. The harness must uphold its half of the contract, and the contract has two clauses.

First, the results for all tool calls in a step must go back in a **single** user message, one `tool_result` block per `tool_use` block, IDs matched. The minimal harness gets this right by construction: it collects `results` across the whole content array before appending once. The tempting refactor — append one user message per result inside the inner loop — produces an invalid conversation shape and, more insidiously, teaches the model over the course of a session that parallel requests get mishandled, so it stops making them. Degraded parallelism is a silent performance bug that shows up as "the agent feels slow" with nothing in the error logs.

Second, since the model batched the requests, the harness may execute them concurrently — `asyncio.gather` over async handlers, or a thread pool for blocking ones. But concurrency is a *harness* decision requiring a *harness* judgment: three `read_file` calls are safe in parallel; `write_file` and `run_tests` against the same working tree are not. The minimal harness executes serially, which is the correct conservative default. A production harness typically tags each tool as parallel-safe or not and gathers only the safe ones. When you genuinely need to prohibit batching — some legacy tools can't tolerate it — the API offers `disable_parallel_tool_use` on the `tool_choice` parameter, but reaching for it costs real throughput; fix the tools instead.

## Where loop engineering diverges from prompt engineering

It is worth being precise about what changed when we wrapped a `for` loop around the model call, because the skill set changes with it.

Prompt engineering optimizes a single mapping: one context in, one response out. Its unit of iteration is the wording. Loop engineering optimizes a *trajectory*: dozens of model calls whose contexts are built from each other's outputs, executing against an environment that pushes back. Its units of iteration are structural, and almost none of them are sentences:

- **What accumulates.** Every tool result is appended forever (in the minimal design), so return-value design *is* context design. The fifty-line truncation in `run_tests` will do more for Ledgerbot's twenty-step coherence than any phrasing change in the system prompt.
- **What happens on failure.** The `execute` wrapper's error-to-observation conversion decides whether a bad path costs one step or the whole session. Error *messages* become steering inputs — a theme Chapter 4 develops as "errors that teach."
- **When it ends.** Stop conditions are pure harness code, invisible to prompt engineering, and they bound the blast radius of every model misjudgment.
- **What you can see afterwards.** The transcript — that `messages` list, serialized — is the ground truth for debugging a trajectory. Prompt engineering debugs by reading one response; loop engineering debugs by replaying a session.

The Firecrawl harness overview [captures the runtime framing](https://www.firecrawl.dev/blog/what-is-an-agent-harness) in one line: at each step the harness "intercepts, validates, routes, and records." All four verbs are absent from a prompt engineer's vocabulary and central to a harness engineer's. And this is why the discipline earns the systems-engineering label from Chapter 1: when a twenty-step Ledgerbot run goes wrong, the durable fix is almost never "reword the prompt and retry." It is "the test tool returned 40,000 tokens of log and drowned the signal," or "the loop kept going after a truncated message," or "two write tools raced." Failures are environmental problems to fix permanently in the loop, not incantations to adjust.

The minimal harness in this chapter is a real starting point, not a toy to discard — mini-swe-agent's benchmark results are the proof that a linear-history, single-digit-tool loop remains competitive when the model is strong. What the following chapters add is everything that keeps the loop trustworthy as the tasks get longer and the stakes get higher: a deliberately designed tool layer (Chapter 4), context that survives hour-long sessions (Chapter 5), state that survives process restarts (Chapter 6), an execution environment that can't hurt you (Chapter 7), and verification that catches the model declaring victory early (Chapter 10). Every one of them bolts onto a line of code you have now read.

## Further reading

- Shunyu Yao et al., "ReAct: Synergizing Reasoning and Acting in Language Models" — [arxiv.org/abs/2210.03629](https://arxiv.org/abs/2210.03629)
- Anthropic Engineering, "Building Effective Agents" — [anthropic.com/engineering/building-effective-agents](https://www.anthropic.com/engineering/building-effective-agents)
- Simon Willison, "I think 'agent' may finally have a widely enough agreed upon definition" — [simonwillison.net/2025/Sep/18/agents/](https://simonwillison.net/2025/Sep/18/agents/)
- Thorsten Ball, "How to Build an Agent" — [ampcode.com/how-to-build-an-agent](https://ampcode.com/how-to-build-an-agent)
- SWE-agent team, mini-swe-agent (the 100-line agent) — [github.com/SWE-agent/mini-swe-agent](https://github.com/SWE-agent/mini-swe-agent)
- Anthropic, "Tool use with Claude" (API documentation) — [platform.claude.com/docs/en/agents-and-tools/tool-use/overview](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview)
- Anthropic, "Streaming Messages" (API documentation) — [platform.claude.com/docs/en/build-with-claude/streaming](https://platform.claude.com/docs/en/build-with-claude/streaming)
- Firecrawl, "What Is an Agent Harness?" — [firecrawl.dev/blog/what-is-an-agent-harness](https://www.firecrawl.dev/blog/what-is-an-agent-harness)

---

[← Lineage: Test, Fuzz, and Eval Harnesses](ch02-lineage.md) · [Tools and the Agent-Computer Interface →](ch04-tools-and-the-aci.md)
