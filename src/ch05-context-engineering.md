# Chapter 5 — Context Engineering

*An agent's context window is the only memory the model has while it works, and everything in the harness competes for it: the system prompt, tool definitions, instruction files, conversation history, and every tool result the loop appends. This chapter treats the window as what it is — a scarce, expensive, shared resource — and develops the discipline of deciding what enters each model call. We put real numbers on context budgets and on prompt-caching economics, weigh compaction against reset for long-running sessions, and work through the filesystem-as-context patterns that let an agent handle far more information than its window can hold. The through-line is a single idea: the smallest set of high-signal tokens that maximizes the likelihood of the outcome you want.*

## The window is scarce shared memory

Chapter 3 built the agent loop: assemble context, call the model, execute the decision, append the results, repeat. Every part of that loop except the model call is harness code, and the assemble-context step is where most of the leverage lives. **Context engineering** — deciding what enters each model call — is the discipline this chapter develops. Anthropic's engineering team frames the core question as "what configuration of context is most likely to generate our model's desired behavior?" and describes context as "a finite resource with diminishing marginal returns" ([Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)).

Two facts make the resource genuinely scarce rather than merely finite.

The first is architectural. Transformer attention computes pairwise relationships between tokens, so an n-token context implies on the order of n² interactions. Models are trained on a distribution in which shorter sequences dominate, and as context grows, the model's ability to attend to any particular relationship "gets stretched thin," as the Anthropic article puts it. The team calls the resulting budget an *attention budget*: every token added depletes it a little.

The second fact is empirical. Models degrade with context length even on tasks well inside their advertised windows. The research team at Chroma evaluated eighteen state-of-the-art models — including GPT-4.1, Claude 4, and Gemini 2.5 — on deliberately simple tasks and found that every model's reliability declined as input length grew; no model held flat across its advertised window, and models advertising million-token windows did not behave like million-token models much past roughly 200K tokens ([Context Rot](https://research.trychroma.com/context-rot)). The phenomenon has acquired a name, **context rot**: the longer the context, the less accurately the model recalls and uses what is in it.

So a 200,000-token window is not a 200,000-token workspace. It is a budget that buys less per token the more of it you spend. That reframing changes how you build the harness. Failures of coherence in long sessions — the agent forgetting an instruction from an hour ago, re-doing work it already did, fixating on an early misreading of a file — are usually not model failures to be fixed by prompt tweaks. They are environmental problems, in the sense Chapter 1 established: the harness put too much low-signal material into the window, and the fix is to change what the harness puts there.

Ledgerbot, this book's running example — an agent that triages and fixes failing CI builds in a mid-size Python monorepo — makes the problem concrete. A single failing CI run produces a log that can easily reach two megabytes. At the common English-text approximation of roughly four characters per token, that log alone is on the order of 500,000 tokens: it does not fit in the window at all, and even a truncated 200 KB excerpt would consume a quarter of a 200K window before Ledgerbot has read a single source file. Context engineering for Ledgerbot is not an optimization; it is the difference between an agent that works and one that cannot start.

## Anatomy of an assembled context

Before budgeting, itemize. On every step of the loop, the request the harness sends to the model is assembled from a small number of distinct parts, and each behaves differently over the life of a session:

- **The system prompt** — the durable "who am I, what are my rules" framing. Written once, sent on every call, ideally never changing mid-session.
- **Tool definitions** — the JSON-schema descriptions from Chapter 4. Also fixed for the session, also sent on every call.
- **Instruction files** — project-level context loaded at startup, such as the `CLAUDE.md` file Claude Code reads from the repository root ([Claude Code best practices](https://www.anthropic.com/engineering/claude-code-best-practices)) or the `AGENTS.md` convention other coding agents use. Fixed per project.
- **Conversation history** — every prior assistant message and tool call in the session. Grows monotonically.
- **Tool results** — the outputs the harness appended after executing the model's decisions. Usually the fastest-growing component and the least information-dense.
- **The current turn** — the newest user message or tool result the model must respond to.

Here is a plausible startup budget for Ledgerbot against a 200,000-token window:

| Component | Tokens | Share of window |
|---|---:|---:|
| System prompt | 1,500 | 0.8% |
| Repository instruction file | 2,500 | 1.3% |
| Tool definitions (12 tools) | 4,000 | 2.0% |
| Reserved for model output | 8,000 | 4.0% |
| **Available for history and tool results** | **184,000** | **92.0%** |

That looks roomy until you run the loop. Suppose an average step appends 2,500 tokens of history — a short assistant message plus a truncated tool result. The 184,000-token budget supports about 73 steps before the window is full, and context rot means quality decays well before the hard limit. Now suppose tool results are *not* truncated: a single `pytest` run on a large suite can emit 30,000 tokens of output, and five such results consume most of the budget on their own. The arithmetic explains a rule that every mature harness converges on: tool results are budgeted, truncated, or diverted to disk — never appended raw. Chapter 4 discussed designing tools whose return format respects a token budget; this chapter supplies the other half, what the harness does when output exceeds it.

## Position matters: lost in the middle

Token count is not the only variable; position inside the window matters too. Liu et al. showed that language-model performance on multi-document question answering and key-value retrieval follows a U-shaped curve: models are best at using information at the very beginning or very end of the context and measurably worse when the relevant material sits in the middle — a finding that held even for models explicitly extended for long contexts ([Lost in the Middle: How Language Models Use Long Contexts](https://arxiv.org/abs/2307.03172)). Chroma's follow-up work found the same shape at modern scales, with accuracy dropping by tens of percentage points when the relevant document was buried mid-context ([Context Rot](https://research.trychroma.com/context-rot)).

For harness builders this yields three placement rules. First, durable instructions belong at the top: the system prompt and instruction files sit at the start of the window, one of the two privileged positions. Second, the actionable ask belongs at the bottom: the current task, the current error, the question the model must answer next should be the most recent thing in the window. Third — and least obvious — anything important that has drifted into the middle should be *re-surfaced* rather than trusted to attention. The Manus team calls this **recitation**: their agent maintains a `todo.md` file and rewrites it at the end of the context as work progresses, which "pushes the global plan into the model's recent attention span, avoiding lost-in-the-middle issues" on long multi-step tasks ([Context Engineering for AI Agents: Lessons from Building Manus](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus)). The recitation is redundant by design — the plan already exists earlier in the history — but redundancy at the window's end is cheap insurance against a fifty-step-old goal fading from attention.

## The economics of re-reading: prompt caching

The agent loop has a cost structure that surprises people coming from single-shot API use. The API is stateless: every step re-sends the entire conversation so far. Step 40 of a session doesn't pay for the 2,500 new tokens it appended — it pays for all ~110,000 tokens of accumulated context, again. Input tokens therefore dominate agent economics. The Manus team reports an average input-to-output ratio around 100:1 for their production agent ([Manus](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus)), and they name the consequence bluntly: cache hit rate is "the single most important metric for a production-stage AI agent."

**Prompt caching** lets the provider reuse the computation for a prefix of the request it has recently seen, charging a steep discount for the reused portion. The mechanics differ by provider, but the published economics, as of this writing, are:

- **Anthropic** — caching is explicit: you place up to four `cache_control` breakpoints in the request. Cache *reads* cost 0.1× the base input price. Cache *writes* cost 1.25× base for the default 5-minute TTL, or 2× base for a 1-hour TTL; a read refreshes the 5-minute TTL at no charge ([prompt caching docs](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)).
- **OpenAI** — caching is automatic for prompts of 1,024 tokens or more, with reads discounted 50% on the GPT-4o-era models where the feature launched ([announcement](https://openai.com/index/api-prompt-caching/)) and a larger discount — cached input at roughly 10% of the base rate — on newer model families; cache entries persist through 5–10 minutes of inactivity, up to about an hour ([prompt caching guide](https://developers.openai.com/api/docs/guides/prompt-caching)).
- **Google** — Gemini 2.5 models cache implicitly by default; the discount on cached tokens was 75% at launch ([Gemini 2.5 implicit caching](https://developers.googleblog.com/gemini-2-5-models-now-support-implicit-caching/)) and was later raised to 90% for 2.5-era models. Explicit caching, where you create a cache object with its own TTL, additionally bills storage per token-hour ([context caching docs](https://ai.google.dev/gemini-api/docs/caching)).

Verify these against the providers' pricing pages before budgeting a system — cache pricing has changed more than once — but the shape is stable: reads are 2× to 10× cheaper than fresh input, and (on Anthropic) writes carry a premium.

The write premium gives a break-even worth internalizing. On Anthropic's 5-minute tier, caching a span costs 1.25× to write plus 0.1× per subsequent read. If the span is read once more, you have paid 1.35× for what would have cost 2× uncached — caching pays for itself on the *second* request. On the 1-hour tier the write costs 2×, so two requests cost 2.2× versus 2× uncached: you need at least three requests before the hour-long TTL wins. The hour tier exists for bursty traffic with gaps longer than five minutes between calls, not as a default.

Now the arithmetic that matters for agents. Consider a 40-step Ledgerbot session with a 12,000-token fixed prefix (system prompt, tools, instruction file) and 2,500 tokens of new history per step. The context at step *k* is 12,000 + 2,500(k−1) tokens, so total input across the session is:

    40 × 12,000 + 2,500 × (0 + 1 + ⋯ + 39)
  = 480,000 + 2,500 × 780
  = 2,430,000 input tokens

At Claude Sonnet 4.5's published $3 per million input tokens, the uncached input bill is **$7.29**. With caching and an append-only history, every token is *written* exactly once and *read* on every subsequent step. The distinct tokens number 12,000 + 40 × 2,500 = 112,000; written at 1.25× ($3.75/MTok) they cost $0.42. The remaining ~2.32 million token-reads at 0.1× ($0.30/MTok) cost $0.70. Total input cost: **about $1.12 — a 6.5× reduction**, and the ratio improves as sessions get longer because the read-heavy term grows quadratically while the write term grows linearly. Output cost (40 steps × ~700 tokens × $15/MTok ≈ $0.42) is unchanged either way. For a team running hundreds of Ledgerbot sessions a day, caching is not a nice-to-have; it is the difference between a viable unit economics and an unviable one.

The catch is that caching is a *prefix* match. A cache hit requires the request bytes to be identical up to the breakpoint; a single differing token invalidates everything after it — Manus notes that "even a single-token difference can invalidate the cache from that token onward." That constraint turns caching from a billing detail into an architectural requirement with concrete design rules:

- **Keep the prefix frozen.** No timestamps, request IDs, or "current date" interpolated into the system prompt. If the model needs the date, put it in a message near the end of the context.
- **Append, never edit.** Treat history as append-only. Rewriting an earlier tool result — even to fix a typo — invalidates the cache from that point forward.
- **Serialize deterministically.** JSON encoders in most languages do not guarantee key order. An unsorted `json.dumps` of a tool result can silently produce different bytes for identical data, and with it a 10× cost regression that no error message will ever surface.
- **Don't mutate the tool set mid-session.** Tool definitions render at the front of the request; adding or removing one invalidates the whole cache. When Manus needed to restrict which tools the agent could use at a given moment, they masked token logits during decoding rather than removing definitions, precisely to preserve the cache ([Manus](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus)).

In code, the explicit form on the Anthropic API looks like this — a breakpoint on the system prompt caches the tools-plus-system prefix, and a second breakpoint on the last message caches the accumulated history:

```python
import anthropic

client = anthropic.Anthropic()

response = client.messages.create(
    model="claude-sonnet-4-5",
    max_tokens=4096,
    system=[{
        "type": "text",
        "text": SYSTEM_PROMPT,                      # frozen for the session
        "cache_control": {"type": "ephemeral"},     # breakpoint 1: tools + system
    }],
    tools=TOOL_DEFINITIONS,                         # stable order, never mutated
    messages=history_with_cache_breakpoint(history) # breakpoint 2: last block
)

u = response.usage
print(f"fresh={u.input_tokens} written={u.cache_creation_input_tokens} "
      f"read={u.cache_read_input_tokens}")
```

The `usage` fields are the observability hook: if `cache_read_input_tokens` stays at zero across steps of the same session, something in your prefix is churning, and finding it is worth an afternoon.

## Compaction versus reset

Caching makes a long session affordable; it does nothing to make it *fit*. Sooner or later the history approaches the window, and the harness must shed tokens. The consistency glossary for this book fixes the two verbs: **compaction** summarizes history in place; **reset** clears it.

Compaction takes the conversation, produces a summary that preserves what matters — decisions made, files touched, current state of the task, unresolved errors — and reinitiates the context with the summary standing in for the detail ([Anthropic on context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)). Claude Code implements this as `auto-compact`, triggering automatically as the session nears its window, and exposes a manual `/compact` command that accepts instructions about what to preserve ("/compact keep the failing test names and the fix strategy"); after compaction, content loaded from disk at startup — the instruction file, memory files — is re-injected, since it can be reconstructed at full fidelity from its source ([Claude Code docs](https://code.claude.com/docs/en/sessions)). That detail generalizes: *never summarize what you can reload.* A summary of `CLAUDE.md` is strictly worse than `CLAUDE.md`.

Compaction's virtue is continuity; its vice is lossiness, and the loss compounds. A summary of a summary of a summary drifts, and the model cannot tell which details the summarizer discarded — including, sometimes, the one detail that mattered. The safest early candidates for compaction are old tool results: a raw log inspected forty steps ago is almost pure dead weight once its conclusion has been extracted, and clearing deep-history tool outputs is among the least destructive interventions available.

Reset is the blunter instrument: end the session, start a new one, and carry forward only what was deliberately written down. It sounds like a failure mode, but for well-structured work it is often the better engineering choice. Anthropic's team building a harness for long-running application development found that clearing context entirely between work sessions, rather than compacting, kept the model more coherent — accepting "token overhead, and latency" as the price — because each session started from a clean, high-signal state assembled from durable artifacts on disk rather than from an accreted summary ([Harness design for long-running application development](https://www.anthropic.com/engineering/harness-design-long-running-apps)). Reset forces the discipline that makes it work: the harness must externalize state into files a fresh session can read, which is exactly the machinery Chapter 6 builds under the names checkpoint, state file, and initializer-executor.

The same article documents a subtler failure that context management must design around: **context anxiety**. Claude Sonnet 4.5, aware of its remaining context, would "begin wrapping up work prematurely" as it believed it was running out — summarizing, cutting scope, proposing to stop — even when ample budget remained. Two mitigations follow. Don't surface a token countdown to the model unless you must; and if the model starts wrapping up early on long tasks, an explicit reassurance in the prompt that context is ample, or a harness-enforced session boundary that arrives before the model starts worrying, both work. This is a clean example of Chapter 1's thesis that harness components encode assumptions about model behavior — and, as Chapter 14 discusses, an example likely to go stale as models are trained out of the behavior.

The decision between the two verbs comes down to task shape. Conversational, exploratory work — where the value is in the accumulated nuance of the exchange — favors compaction, because there is no natural artifact that captures "the vibe of the investigation so far." Structured work with checkpointable state — Ledgerbot fixing an enumerable list of failing tests — favors reset at task boundaries, because a fresh window plus a good state file beats a stale window every time. Production harnesses use both: compaction as the emergency pressure valve inside a session, reset at every natural boundary.

## The filesystem as context

The strongest pattern in this chapter inverts the problem. Instead of asking "how do we fit the information into the window?", ask "how do we leave the information *outside* the window and let the agent fetch what it needs?" Give the agent a filesystem — real or virtual — and the window stops being storage and becomes what it should have been all along: working memory. Manus states the position maximally: treat "the file system as the ultimate context... unlimited in size, persistent by nature, and directly operable by the agent itself" ([Manus](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus)).

Three concrete patterns cover most of the ground.

**Spill large tool results to disk.** When a tool produces more output than its token budget allows, the harness writes the full output to a scratch file and returns a *pointer plus a preview*: the path, the size, the first and last few kilobytes, and an instruction to search the file for specifics. The compression is lossless-by-reference — nothing is destroyed, only deferred:

```python
from pathlib import Path

MAX_INLINE_CHARS = 8_000  # ≈ 2,000 tokens

def deliver_tool_result(raw: str, scratch: Path, name: str) -> str:
    """Return tool output inline if small, else spill to disk and
    return a pointer with head/tail preview."""
    if len(raw) <= MAX_INLINE_CHARS:
        return raw
    path = scratch / f"{name}.log"
    path.write_text(raw, encoding="utf-8")
    head, tail = raw[:2_000], raw[-2_000:]
    return (
        f"[output: {len(raw):,} chars — full text saved to {path}]\n"
        f"--- first 2,000 chars ---\n{head}\n"
        f"--- last 2,000 chars ---\n{tail}\n"
        f"Search the file (e.g. grep -n 'FAILED\\|Error' {path}) "
        f"rather than requesting the full output."
    )
```

For Ledgerbot this one function converts the impossible 500K-token CI log into a ~1,200-token tool result that still contains what usually matters (failures cluster at the end of a log) and a durable handle to everything else. Note the last line: a good spill message *teaches the recovery move*, in the spirit of Chapter 4's error-messages-that-teach.

**Structured note-taking.** The agent writes its own notes — a `NOTES.md`, a running plan, an insight file — and the harness pulls them back into context at the right moments. Anthropic describes agents that persist notes outside the window and retrieve them later to maintain coherence across long horizons ([Effective context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)); Manus's recitation `todo.md` is the same pattern tuned for attention placement. The harness's job is mostly to make the pattern reliable: create the notes file at session start, instruct the model that it exists and when to update it, and re-inject it after any compaction or reset.

**Just-in-time retrieval over pre-stuffing.** The retrieval-augmented default of 2023 was to embed everything, search, and stuff the top-k results into the prompt up front. The agentic alternative is to keep *lightweight identifiers* in context — file paths, query names, URLs — and give the agent tools to dereference them at runtime. Claude Code is the canonical hybrid: the instruction file is loaded up front because it is small and always relevant, while `glob` and `grep` primitives let the agent "navigate its environment and retrieve files just-in-time" ([Anthropic](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)). Pre-stuffing pays the token cost of everything that *might* be relevant; just-in-time pays a latency cost, plus a few hundred tokens, for exactly what *is*. For a codebase — where relevance is impossible to predict before the investigation starts — just-in-time wins decisively. The pragmatic rule: pre-load what is small and certain (instructions, the task, the state file); fetch on demand what is large or uncertain (source files, logs, documentation).

A fourth pattern belongs to Chapter 9 but deserves a sentence here: **subagents as context isolation**. When a subtask would flood the parent's window — read thirty files, chase a dependency graph — spawn a fresh agent with a clean window, let it burn its own budget, and take back only a condensed summary; Anthropic reports these summaries typically run 1,000–2,000 tokens regardless of how much the subagent read. The parent's context stays clean because the exploration happened in someone else's window.

## Keeping errors in context

One counter-intuitive rule closes the loop. When the agent tries something and it fails — a bad command, a wrong file path, a test that still fails after the "fix" — the instinct is to tidy the transcript: drop the failed attempt, retry cleanly. Manus argues the opposite: "leave the wrong turns in the context." A model that can see its own failed action and the resulting stack trace implicitly updates away from repeating it; a model whose failures are laundered out of the transcript will happily repeat them ([Manus](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus)). Failed attempts are among the highest-signal tokens in the window. Truncate their *verbosity* — the spill pattern applies to error output too — but never their *existence*. Compaction summaries should likewise carry forward what was tried and didn't work, or the post-compaction agent will re-walk every dead end.

## A context policy for Ledgerbot

Pulling the chapter together into the policy an actual harness would ship. Ledgerbot's context assembly, stated as design decisions:

1. **Fixed prefix, frozen and cached.** System prompt, twelve tool definitions, repository instruction file: ~8K tokens, byte-stable for the session, `cache_control` breakpoint at the end. No timestamps anywhere in it.
2. **Tool results budgeted at 2,000 tokens.** Anything larger spills to `/tmp/ledgerbot/<session>/` with a head/tail preview and a grep hint.
3. **State on disk, pointers in context.** The failing-test list, the plan, and progress notes live in files (Chapter 6 gives them schemas). The context carries their paths and, for the plan, a recited copy near the window's end, refreshed as work proceeds.
4. **Reset at task boundaries, compact only under pressure.** Each failing test suite is a fresh session initialized from the state file. If a single test's investigation threatens the window, auto-compaction preserves the failing test name, the hypothesis, files touched, and attempts made — and re-injects the instruction file afterward.
5. **Errors stay.** Failed fixes remain in history, truncated but present.
6. **Cache hit rate on the dashboard.** `cache_read_input_tokens / total input` per session, alerting when it drops — because a silent prefix-churn bug is a 6× cost regression that produces no error log, only a bill.

None of these decisions is a prompt. All of them are harness code — which is the point. The model brings the reasoning; the harness decides what the reasoning gets to see. Chapter 6 takes the natural next step: when the window resets, where does the agent's knowledge live, and how does the next session pick it up?

## Further reading

- Anthropic Engineering, [Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) — attention budgets, context rot, compaction, note-taking, just-in-time retrieval.
- Anthropic Engineering, [Harness design for long-running application development](https://www.anthropic.com/engineering/harness-design-long-running-apps) — context anxiety, reset-over-compaction, three-agent harness.
- Anthropic Engineering, [Claude Code best practices](https://www.anthropic.com/engineering/claude-code-best-practices) — CLAUDE.md and instruction-file patterns.
- Manus, [Context Engineering for AI Agents: Lessons from Building Manus](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus) — KV-cache hit rate, append-only design, tool masking, filesystem-as-context, recitation, keeping errors.
- Nelson F. Liu et al., [Lost in the Middle: How Language Models Use Long Contexts](https://arxiv.org/abs/2307.03172) — the U-shaped position curve.
- Chroma, [Context Rot: How Increasing Input Tokens Impacts LLM Performance](https://research.trychroma.com/context-rot) — degradation with length across 18 models.
- Anthropic, [Prompt caching documentation](https://platform.claude.com/docs/en/build-with-claude/prompt-caching) — cache read/write multipliers, TTLs, breakpoints.
- OpenAI, [Prompt caching guide](https://developers.openai.com/api/docs/guides/prompt-caching) and [launch announcement](https://openai.com/index/api-prompt-caching/) — automatic caching, discounts, retention.
- Google, [Gemini 2.5 implicit caching](https://developers.googleblog.com/gemini-2-5-models-now-support-implicit-caching/) and [context caching docs](https://ai.google.dev/gemini-api/docs/caching).
- Claude Code documentation, [Manage sessions](https://code.claude.com/docs/en/sessions) — auto-compact, `/compact`, `/clear`.

---

[← Tools and the Agent-Computer Interface](ch04-tools-and-the-aci.md) · [Memory, State, and Sessions →](ch06-memory-state-sessions.md)
