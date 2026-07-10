# Chapter 11 — Reliability and Observability

*A harness that works in a demo and a harness that works in production differ in exactly the ways a script differs from a distributed system. An agent run is a long-lived, stateful computation spread across an unreliable network: model API calls that rate-limit and time out, tool executions that fail halfway, processes that die mid-session. This chapter treats the harness as the distributed system it is. We cover retries with backoff and jitter, rate-limit handling, resuming a session from its transcript, designing tools that survive re-execution, layered timeouts, heartbeats for long silent stretches, and structured tracing — including the OpenTelemetry conventions for agents and the append-only JSONL transcript pattern. We close with the discipline that ties it all together: when something goes wrong, the transcript is the ground truth, and debugging an agent means reading it.*

## The harness is a distributed system

Consider what actually happens when Ledgerbot — this book's running example, an agent that triages and fixes failing CI builds in a mid-size Python monorepo — works a single failing build. The harness makes forty to a hundred model calls over HTTP to a provider on the other side of the internet. Each call carries a context that took dozens of network round trips to accumulate. Between calls, the harness executes tools: shell commands in a container, file edits, test runs, a `git push`. The whole session runs for twenty minutes to several hours. During that window, any of the following will eventually happen: the model API returns a 429 because another team's batch job consumed the org's rate limit; a 500 or an overloaded-error arrives mid-stream; the container's test run hangs on a network-dependent test; the orchestrating process is OOM-killed; a deploy restarts the host.

None of these are exotic. At forty calls per session and a few hundred sessions per day, a failure mode with a 0.1% per-call probability fires several times daily. The naive harness — a `while` loop around `client.messages.create()` with no persistence — responds to each of them by losing the entire session: twenty minutes of work, several dollars of tokens, and whatever partial progress the agent had made in its sandbox.

The distributed-systems literature has spent fifty years on exactly this class of problem, and its standard toolkit transfers almost unchanged: retry transient failures with exponential backoff and jitter; make operations idempotent so retries are safe; persist state in an append-only log so a crashed process can recover; bound every operation with a timeout; emit heartbeats so a supervisor can distinguish *slow* from *stuck*; and trace everything so failures can be diagnosed after the fact. Anthropic's engineering team, describing their production research agent, put the stakes plainly: agents are stateful across many tool calls, so errors can't be handled by simply restarting — the system must be able to [resume from where the agent was when the error occurred](https://www.anthropic.com/engineering/multi-agent-research-system), combining "retry logic and regular checkpoints" with the agent's own adaptability.

One property of agents makes this toolkit *easier* to apply than in a classical service, and one makes it harder. Easier: the agent loop from Chapter 3 is already structured as a sequence of discrete steps — one model call plus its tool executions — and each step's inputs and outputs are serializable messages. That gives you a natural unit of retry, a natural checkpoint boundary, and a natural trace span. Harder: the component at the center of the system is nondeterministic. Two runs from the same state diverge, so you cannot reproduce a failure by replaying inputs the way you would with a deterministic service. That asymmetry shapes everything in this chapter: we lean hard on the deterministic scaffolding (logs, retries, checkpoints) precisely because the nondeterministic core cannot be relied on to behave the same way twice.

## Retries, backoff, and the 429

Start with the most frequent failure: the model API call. Provider errors split into two categories, and conflating them is the most common retry bug.

**Retryable:** HTTP 429 (rate limited), 500-class errors, 529/overloaded, request timeouts, and connection resets. These are transient by definition — the same request is expected to succeed later.

**Not retryable:** 400 (malformed request), 401/403 (auth), 404 (bad model ID), 413 (request too large). Retrying these burns quota and delays the real fix. A 400 caused by a context that outgrew the window will never succeed on retry; it needs compaction (Chapter 5), not patience.

The official SDKs already retry the transient class. The [Anthropic Python SDK](https://github.com/anthropics/anthropic-sdk-python) retries connection errors, 408, 409, 429, and ≥500 responses with exponential backoff, twice by default (`max_retries=2`); the OpenAI SDK behaves similarly. For a chat app, the default is fine. For a harness running unattended overnight, two retries spanning a few seconds is nowhere near enough: rate-limit storms and provider incidents last minutes, not milliseconds. The harness needs its own outer retry layer with a much longer horizon.

Two refinements matter beyond the textbook `sleep(2 ** attempt)`. First, **honor `retry-after`.** A 429 usually carries a `retry-after` header stating exactly when capacity frees up (see the [Anthropic rate-limit docs](https://platform.claude.com/docs/en/api/rate-limits) and the [OpenAI cookbook's rate-limit guide](https://cookbook.openai.com/examples/how_to_handle_rate_limits)); guessing with backoff when the server has told you the answer is wasteful in both directions. Second, **add jitter.** When ten parallel Ledgerbot sessions hit the same rate limit, deterministic backoff retries them in synchronized waves that re-trigger the limit — the thundering herd. AWS's classic analysis, [Exponential Backoff and Jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/), shows that "full jitter" — sleeping a uniform random duration between zero and the exponential cap — both reduces total work and finishes sooner than correlated backoff.

Here is the outer layer, written against the Anthropic SDK's typed exceptions but structurally identical for any provider:

```python
import random
import time

import anthropic

RETRYABLE_STATUS = {408, 409, 429, 500, 502, 503, 529}

def call_model_with_retry(
    client: anthropic.Anthropic,
    *,
    max_attempts: int = 8,
    base_delay: float = 1.0,
    max_delay: float = 300.0,
    **params,
):
    """Model call with full-jitter exponential backoff. Honors retry-after."""
    for attempt in range(max_attempts):
        try:
            return client.messages.create(**params)
        except anthropic.APIConnectionError:
            server_hint = None            # network blip: pure backoff
        except anthropic.APIStatusError as err:
            if err.status_code not in RETRYABLE_STATUS:
                raise                     # 400/401/404/413: retrying can't help
            server_hint = err.response.headers.get("retry-after")

        if attempt == max_attempts - 1:
            raise RuntimeError(f"model call failed after {max_attempts} attempts")

        if server_hint is not None:
            delay = float(server_hint)                     # server knows best
        else:
            cap = min(max_delay, base_delay * 2 ** attempt)
            delay = random.uniform(0, cap)                 # full jitter
        time.sleep(delay)
```

With a cap of 300 seconds and eight attempts, this layer rides out roughly a twenty-minute provider incident before giving up — long enough that overnight runs survive the routine turbulence that would otherwise kill them. The practical payoff is operational: teams can run near their provisioned rate-limit capacity and let backoff absorb the bursts, instead of self-throttling to a safety margin and leaving capacity idle.

One subtlety: retrying a *model call* is always safe, because generating a response has no side effects — the worst case is paying for tokens twice. Retrying a *step* is a different matter, because a step includes tool execution. That distinction drives the idempotency section below.

## Timeouts, layered

A retry policy is only as good as the timeout that triggers it. Harnesses need timeouts at three distinct layers, and each protects against a different failure.

**Request timeouts** bound a single model call. SDK defaults are generous — the Anthropic Python SDK defaults to ten minutes — because large-`max_tokens` completions genuinely take that long. The standard mitigation is streaming: a streamed response delivers tokens continuously, so the meaningful timeout becomes *time since last event* rather than *time for the whole response*, and a stalled stream is detectable in seconds instead of minutes. Beware the classic trap in hand-rolled HTTP: most client libraries' "read timeout" is per-chunk, resetting on every byte, so a connection trickling one heartbeat byte per second never times out. A wall-clock deadline must be enforced at your loop level (track a monotonic clock and abandon the call when the budget is gone), not delegated to the socket layer.

**Tool timeouts** bound each tool execution. A test suite that normally runs in ninety seconds but occasionally hangs on a network call will, without a timeout, silently freeze the whole session. Every `subprocess.run` in a tool implementation gets a `timeout=`; every container exec gets a deadline. Critically, a tool timeout should be reported *to the agent* as a structured error result — `"Error: command timed out after 120s"` — not raised as a harness exception. Models handle tool failure well when they can see it: as Anthropic's team observed about production agents, ["letting the agent know when a tool is failing and letting it adapt works surprisingly well"](https://www.anthropic.com/engineering/multi-agent-research-system). The agent retries with a narrower test selection or a different approach; the harness stays up.

**Session timeouts** bound the whole run: a max-steps cap (Chapter 3's stop conditions) plus a wall-clock ceiling. Without one, a confused agent loops forever, and "forever" is billed by the token. The right response to hitting a session ceiling is usually not a hard kill but a graceful wind-down: inject a message telling the agent its budget is nearly exhausted and ask it to write down its state — a handoff note in the sense of Chapter 6 — so the next session can continue rather than restart.

The layering matters because each timeout is meaningless without the one above it. A request timeout without a tool timeout leaves you hanging on `pytest`; both without a session ceiling leave you with an agent that makes perfectly timely calls in a perfectly useless circle.

## Resume from the transcript

Retries handle failures *within* a step. The harder problem is failure *of the harness itself* — the process dies mid-session, taking with it the in-memory `messages` list that constitutes the agent's working state. The fix is the oldest trick in databases: a write-ahead log. Persist every event to an append-only file *as it happens*, and the in-memory state becomes a disposable cache that can be rebuilt from disk at any time.

The near-universal format for this log is JSONL — one JSON object per line. It is append-only (a crash corrupts at most the final partial line, which a loader can skip), streamable (a `tail -f` is a live view of the session), and greppable. Claude Code, the most widely deployed agent harness, stores every session this way: a `.jsonl` file per session under `~/.claude/projects/`, one line per message, tool call, or metadata event, which is what makes its [`--resume` command](https://code.claude.com/docs/en/sessions) possible — the CLI rebuilds the full conversation from the file and continues where it stopped, whether the interruption was a Ctrl-C, a crash, or a reboot.

A minimal transcript-backed loop looks like this:

```python
import json
import os
from pathlib import Path

class Transcript:
    """Append-only JSONL session log. Survives crashes; rebuilds state."""

    def __init__(self, path: Path):
        self.path = path
        self._f = open(path, "a", encoding="utf-8")

    def append(self, record: dict) -> None:
        self._f.write(json.dumps(record, ensure_ascii=False) + "\n")
        self._f.flush()
        os.fsync(self._f.fileno())   # durable before we act on it

    @staticmethod
    def load_messages(path: Path) -> list[dict]:
        messages = []
        for line in path.read_text(encoding="utf-8").splitlines():
            try:
                record = json.loads(line)
            except json.JSONDecodeError:
                break                 # torn final write from a crash; stop here
            if record["type"] in ("user", "assistant"):
                messages.append({"role": record["type"],
                                 "content": record["content"]})
        return messages

def run_session(client, session_path: Path, task: str, max_steps: int = 100):
    if session_path.exists():
        messages = Transcript.load_messages(session_path)   # resume
        t = Transcript(session_path)
    else:
        messages = [{"role": "user", "content": task}]      # fresh start
        t = Transcript(session_path)
        t.append({"type": "user", "content": task})

    for _ in range(max_steps):
        response = call_model_with_retry(
            client, model="claude-sonnet-4-6",
            max_tokens=8192, messages=messages, tools=TOOLS,
        )
        content = [b.model_dump() for b in response.content]
        messages.append({"role": "assistant", "content": content})
        t.append({"type": "assistant", "content": content})

        if response.stop_reason != "tool_use":
            return response

        results = execute_tools(response)     # see idempotency, below
        messages.append({"role": "user", "content": results})
        t.append({"type": "user", "content": results})
```

Three details carry the weight here. The `fsync` before acting ensures the log never lags reality — an event is durable before its consequences exist, which is the write-ahead discipline. The loader stops at the first unparsable line rather than raising, converting a torn write into a clean truncation. And resumption is just "load messages, continue the loop": because the model API is stateless and the conversation *is* the state, there is nothing else to restore. The same mechanism that survives crashes also survives provider-side session death: if a long-lived CLI session aborts on repeated API errors, the harness can relaunch it against the same transcript with full context restored, rather than starting the task over.

There is one wrinkle: a crash can land *between* a logged assistant message containing tool calls and the logged tool results. On resume, the transcript ends with unanswered `tool_use` blocks. The harness must detect this and either re-execute the tools (safe only if they are idempotent — next section) or append synthetic error results ("execution was interrupted; re-run if needed") and let the agent decide. Both are correct; silently dropping the dangling tool calls is not, because most providers reject a conversation whose tool calls have no matching results.

Chapter 6 covers the broader state architecture — checkpoints, durable artifacts, the initializer-executor pattern. The reliability-specific point is narrower: whatever else you persist, the transcript is the recovery mechanism of last resort, and it costs almost nothing to maintain.

## Idempotent tools and partial-failure semantics

Re-executing a model call is free of side effects. Re-executing a tool is not. If the harness crashed after `git push` succeeded but before its result was logged, a resume that blindly re-runs the step pushes twice — harmless for `push`, catastrophic for "send the deployment webhook" or "post the PR comment." This is the classic exactly-once problem, and the classical answer applies: you cannot guarantee exactly-once *execution*, so you engineer for at-least-once execution plus idempotent *effects*.

Sort Ledgerbot's tools into three bins:

- **Naturally idempotent:** `read_file`, `grep`, `run_tests`, `list_failing_jobs`. Re-execution wastes a little time and nothing else. These can be retried freely, and on resume they can simply be re-run.
- **Idempotent with care:** `write_file` with full contents (last write wins), `apply_patch` if it checks whether the patch is already applied, `create_branch` if "already exists" is treated as success. The care is in the implementation: an `edit_file(old, new)` that fails when `old` is no longer present is *better* than idempotent — it detects its own duplicate execution and reports it.
- **Not idempotent:** `open_pull_request`, `send_slack_message`, `trigger_deploy`. Duplicates are user-visible harm.

For the third bin, the standard mechanism is the **idempotency key**, exactly as payment APIs use it: the caller attaches a unique key to the request, the server stores the result under that key, and a retry with the same key returns the stored result instead of re-executing. [Stripe's implementation](https://docs.stripe.com/api/idempotent_requests) is the canonical reference. In a harness, the natural key is derived from the transcript position — `f"{session_id}:{step_index}:{tool_call_id}"` — because the provider-assigned tool-call ID is already unique and already logged. The tool layer keeps a small table mapping keys to results; a re-executed step finds the key and returns the recorded result without side effects. This composes beautifully with the transcript: log the tool result *with its key* before returning it to the loop, and replay-after-crash becomes a pure cache lookup.

Where a downstream API offers no idempotency support, fall back to **check-then-act**: `open_pull_request` first queries for an existing open PR from the same branch and returns it if found. It is racy in theory and almost always sufficient in practice, since the race window is a single agent's own retry, not concurrent writers.

Partial failure inside a single step deserves the same explicitness. When a model requests three tool calls in parallel (Chapter 3) and the second one fails, do not throw away the step. Return all three results, with the failure as a structured error result (`is_error: true` in the Anthropic API's terms) — never silently drop a failed call, and never let one failure discard its siblings' successes. The model sees one success, one error, one success, and plans accordingly. The harness's job in partial failure is honest bookkeeping, not heroics; the agent's job is adaptation, and it is genuinely good at it when the bookkeeping is honest. This is also where error-message design from Chapter 4 pays reliability dividends: `"Error: patch failed to apply at hunk 2; file has changed since read"` prompts a re-read, while a bare stack trace prompts a random walk.

## Heartbeats: distinguishing slow from stuck

Long agent runs have long silences. A single model call at high reasoning effort can take minutes; a test suite takes ten; and from the outside, a harness doing deep useful work is indistinguishable from one that deadlocked twenty minutes ago. Operators respond to unexplained silence in exactly one way — they kill the run — and half the time they kill a healthy one.

Workflow engines solved this decades ago with **heartbeats**: a long-running activity periodically tells its supervisor "still alive, still progressing," and the supervisor treats a missed heartbeat, not mere elapsed time, as the failure signal. [Temporal's activity heartbeats](https://docs.temporal.io/activities) are the modern reference design, and the pattern transplants directly into a harness at two levels.

**Liveness heartbeats** prove the process is working. The cheapest implementation is a counter on the agent loop — every N tool calls, emit one line to stdout and to the trace: `[agent] 75 tool calls (112 messages), last activity 4s ago`. A watchdog thread adds coverage for the truly silent case:

```python
import threading
import time

class Heartbeat:
    def __init__(self, interval: float = 30.0):
        self.last_activity = time.monotonic()
        self.tool_calls = 0
        threading.Thread(target=self._beat, args=(interval,),
                         daemon=True).start()

    def touch(self, event: str) -> None:          # call on every loop event
        self.last_activity = time.monotonic()
        if event == "tool_call":
            self.tool_calls += 1

    def _beat(self, interval: float) -> None:
        while True:
            time.sleep(interval)
            idle = time.monotonic() - self.last_activity
            print(f"[heartbeat] {self.tool_calls} tool calls, "
                  f"idle {idle:.0f}s", flush=True)
```

Now "stuck" has an operational definition: idle time exceeding the largest legitimate single operation (say, the tool timeout plus the request timeout). A supervisor — human or script — can kill on *that* with confidence, instead of guessing.

**Progress signals** prove the work is going somewhere. A per-action line on stderr — `[step 37] → bash: pytest tests/billing -x` — costs one `print` per tool call and transforms operability: an operator tailing the log can see at a glance whether the agent is exploring, editing, or verifying, and whether it has been running the same failing command for forty minutes. This is observability's humblest form, and for a single-node harness it delivers most of the value of a tracing stack at none of the cost. The two are complements, not substitutes: heartbeats are for the moment of operation; traces, next, are for everything after.

## Structured tracing: JSONL transcripts and OpenTelemetry

The transcript answers "what did the agent say and do." Production operation asks more: how many tokens did step 40 consume, which sessions are burning cache misses, what is p95 step latency, which tool fails most often, and — across ten thousand sessions — where does the failure rate concentrate? These are telemetry questions, and the emerging standard answer is OpenTelemetry's **GenAI semantic conventions**: a shared vocabulary of span names and attributes so that a trace from any harness reads the same in any backend. The [conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/), developed by an OpenTelemetry special interest group since 2024, now cover LLM calls, agent invocations, tool execution, and token metrics, and are supported natively by the major observability vendors ([overview](https://opentelemetry.io/blog/2026/genai-observability/)).

The trace structure mirrors the agent loop: one root **`invoke_agent`** span for the session; a child **`chat`** span per model call carrying `gen_ai.request.model`, `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`, and `gen_ai.response.finish_reasons`; and an **`execute_tool`** span per tool invocation carrying `gen_ai.tool.name`. Prompt and completion *content* is deliberately not captured by default — content capture is an explicit opt-in, which is the right default for privacy and for storage sanity. Instrumented by hand, the loop looks like this:

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import (BatchSpanProcessor,
                                            ConsoleSpanExporter)

provider = TracerProvider()
provider.add_span_processor(BatchSpanProcessor(ConsoleSpanExporter()))
# in production, swap ConsoleSpanExporter for an OTLP exporter
trace.set_tracer_provider(provider)
tracer = trace.get_tracer("ledgerbot")

def run_session_traced(client, task: str):
    with tracer.start_as_current_span("invoke_agent ledgerbot") as session_span:
        session_span.set_attribute("gen_ai.operation.name", "invoke_agent")
        session_span.set_attribute("gen_ai.agent.name", "ledgerbot")

        for step, response in enumerate(agent_loop(client, task)):
            with tracer.start_as_current_span("chat") as llm_span:
                llm_span.set_attribute("gen_ai.operation.name", "chat")
                llm_span.set_attribute("gen_ai.request.model", response.model)
                llm_span.set_attribute("gen_ai.usage.input_tokens",
                                       response.usage.input_tokens)
                llm_span.set_attribute("gen_ai.usage.output_tokens",
                                       response.usage.output_tokens)
                llm_span.set_attribute("gen_ai.response.finish_reasons",
                                       [response.stop_reason])

            for call in tool_calls(response):
                with tracer.start_as_current_span(
                        f"execute_tool {call.name}") as tool_span:
                    tool_span.set_attribute("gen_ai.operation.name",
                                            "execute_tool")
                    tool_span.set_attribute("gen_ai.tool.name", call.name)
                    execute(call)
```

Frameworks like LangChain, CrewAI, and AutoGen emit these spans automatically via instrumentation packages, and vendors including [Datadog](https://www.datadoghq.com/blog/llm-otel-semantic-convention/) ingest them natively — but the conventions matter most to those of us building custom harnesses, because they mean your bespoke loop's traces are first-class citizens in standard tooling rather than a proprietary log format.

So which do you need — transcripts or traces? Both, because they answer different questions and fail differently. The JSONL transcript is *complete* (full content, exact bytes the model saw and produced) and *local* (a file on disk, readable with `less`, alive even when your telemetry pipeline is down); it is the debugging artifact. The OTel trace is *aggregable* (dashboards, alerts, cross-session queries, cost attribution) and *content-free by default*; it is the operations artifact. A useful third element joins them: log the trace ID into the transcript's metadata line and the session ID as a span attribute, so a spike on a dashboard resolves to the exact transcripts that caused it in one query.

One more logging habit distinguishes debuggable harnesses: **decision logging**. Beyond messages and spans, record why the *harness* did what it did — which stop condition fired, why a submission was rejected, which retry policy activated, what the compaction trigger saw. When a session ends early, the question is "who decided that, the model or the machinery?" and only the harness can answer for the machinery. One JSONL record per decision, `{"type": "harness_decision", "decision": "max_steps_reached", ...}`, interleaved in the transcript, settles it.

## Debugging from transcripts

Everything above exists so that when a run fails — and runs fail — you can find out why. The discipline that makes this tractable has a blunt name: **the transcript is the ground truth.** Not the summary the agent gave, not the dashboard, not your memory of what the prompt says. Anything not in the transcript did not happen; anything in the transcript happened exactly as recorded, because the transcript is what the model actually saw and actually emitted. Agents are, in Anthropic's phrase, ["non-deterministic between runs, even with identical prompts"](https://www.anthropic.com/engineering/multi-agent-research-system) — you usually cannot reproduce the failure, so the recording *is* the investigation. Their team's conclusion from production experience was to add full tracing precisely because agent errors can't be diagnosed by rerunning them.

Debugging a transcript is closer to reading a flight recorder than stepping through a debugger, and it rewards a repeatable procedure:

1. **Read the ending first.** The last few records tell you the failure *mode*: an exhausted step budget, a harness exception, a model that declared victory falsely, a tool error loop. Each mode has different usual suspects.
2. **Find the divergence step.** Walk backward to the last point where the run was clearly on track, then read forward to the first step where a competent engineer would have acted differently. Failures almost always trace to one identifiable step; everything after is downstream noise.
3. **Read the model's *inputs* at that step, not just its output.** The near-universal root cause of a bad decision is bad context: a truncated tool result, an error message that said nothing (Chapter 4), a compaction that dropped the load-bearing fact (Chapter 5), a stale file the agent had no way to know had changed. The model made a reasonable decision given what it saw; the harness fed it the wrong "what it saw." This is why the debugging stance from Chapter 1 — treat failures as environmental problems to fix permanently, not prompts to retry — is a *reliability* practice, not just a design slogan.
4. **Classify: model, tool, or harness.** Model failures (poor plan from good context) are prompt or model-choice problems. Tool failures (misleading result from correct call) are ACI problems. Harness failures (wrong context assembly, botched retry, mis-fired stop condition) are code bugs, and the decision log is where they show up.
5. **Turn the diagnosis into a permanent artifact.** A fixed tool description, a new structured error message, a regression case in the eval suite (Chapter 12) built from this very transcript. A transcript that taught you something and then got deleted taught you nothing.

Two practical enablers make the procedure fast. First, invest in a transcript *viewer* early — even a fifty-line script that renders JSONL as alternating colored turns with collapsed tool results changes how often people actually read transcripts, and read-frequency is the real bottleneck. Second, keep failed-run transcripts longer than successful ones; the failures are the curriculum, both for you and, in Chapter 12's eval flywheel, for the harness itself.

There is a final, quiet payoff to all of this machinery. Retries, resumable transcripts, idempotent tools, heartbeats, traces — each exists for the bad day. But together they change what you can attempt on the good days: a harness that survives rate limits, crashes, and restarts is a harness you can point at a task that takes all night, and trust that whatever you find in the morning, the record will be complete. Chapter 12 takes up the question that record makes answerable: how do you know whether any of it is getting better?

## Further reading

- Anthropic Engineering — [How we built our multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system): production reliability for stateful agents; resume-from-error, tracing, rainbow deployments.
- OpenTelemetry — [GenAI observability](https://opentelemetry.io/blog/2026/genai-observability/) and the [GenAI semantic conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/): `invoke_agent` / `chat` / `execute_tool` spans and token-usage attributes.
- Claude Code — [Managing sessions](https://code.claude.com/docs/en/sessions): JSONL session storage and `--resume`.
- Marc Brooker, AWS Architecture Blog — [Exponential Backoff and Jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/): why full jitter wins.
- OpenAI Cookbook — [How to handle rate limits](https://cookbook.openai.com/examples/how_to_handle_rate_limits): 429 semantics and backoff patterns.
- Anthropic — [API rate limits](https://platform.claude.com/docs/en/api/rate-limits) and the [Python SDK](https://github.com/anthropics/anthropic-sdk-python) retry/timeout defaults.
- Stripe — [Idempotent requests](https://docs.stripe.com/api/idempotent_requests): the idempotency-key pattern.
- Temporal — [Activities and heartbeats](https://docs.temporal.io/activities): supervision of long-running work.
- Datadog — [LLM Observability supports OpenTelemetry GenAI semantic conventions](https://www.datadoghq.com/blog/llm-otel-semantic-convention/): vendor adoption of the conventions.

---

[← Verification and Feedback Loops](ch10-verification-feedback.md) · [Evaluating the Harness →](ch12-evaluating-harnesses.md)
