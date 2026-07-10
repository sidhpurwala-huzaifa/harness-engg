# Chapter 4 — Tools and the Agent-Computer Interface

*Tools are the only part of your system a model can actually touch, which makes tool design the user-experience discipline of harness engineering — except the user is a language model with no persistence, no ability to ask a colleague, and a hard budget on attention. This chapter treats the agent-computer interface (ACI) as a first-class design surface: the anatomy of a JSON-schema tool definition, the docstring-for-a-junior-developer rule, poka-yoke techniques that make misuse structurally impossible, error messages that teach, return formats sized to token budgets, and the consolidation-versus-proliferation tradeoff. It closes with the two systemic developments every tool designer must now understand — the Model Context Protocol as a standard for tool interoperability, and code execution as an alternative interface that routes data around the model entirely — and with a working method for testing and iterating on tools.*

## Tool design is UX for models

The term **agent-computer interface** comes from the SWE-agent paper, ["SWE-agent: Agent-Computer Interfaces Enable Automated Software Engineering"](https://arxiv.org/abs/2405.15793) (Yang et al., 2024), and the claim in its title is a research finding, not a slogan. The authors observed that "LM agents represent a new category of end users with their own needs and abilities, and would benefit from specially-built interfaces" — just as human developers benefit from IDEs rather than raw `ed`. They built a deliberately agent-shaped interface over a repository (search that returns bounded results, a file viewer windowed to what fits in attention, an editor with built-in lint feedback) and reached 12.5% pass@1 on SWE-bench, state of the art at the time, where prior systems handed the model human-shaped interfaces and did far worse. Same models, different interface, different outcome: the interface *was* the intervention.

Anthropic's [Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents) reports the same lesson from production work — on their own SWE-bench agent they spent "more time optimizing our tools than the overall prompt" — and recommends investing as much design effort in the ACI as teams put into human-computer interfaces. Their later, more detailed guide, [Writing effective tools for agents](https://www.anthropic.com/engineering/writing-tools-for-agents), grounds the analogy in a property they call affordances: "agents have distinct 'affordances' to traditional software — that is, they have different ways of perceiving the potential actions they can take." An API designed for deterministic callers can assume the caller read the docs once, holds identifiers in variables, and never gets confused between similar endpoints. An agent re-reads the docs (the tool definitions) on every step, holds identifiers only in a lossy attention window, and *will* confuse similar endpoints at some base rate you can measure. Tool design is the practice of driving that base rate down.

Why is this the highest-leverage layer? Recall the inversion from Chapter 3: in an agent, the model owns control flow and the harness owns the environment. Tools are the densest part of that environment — they are simultaneously the model's action space, a chunk of its permanent context, and the vocabulary in which it plans. A bad system prompt costs you one instruction; a bad tool costs you every step of every session that touches it.

Throughout this chapter the examples come from Ledgerbot, this book's running example: an agent that triages and fixes failing CI builds in a mid-size Python monorepo, introduced in Chapter 1.

## Anatomy of a tool definition

On the wire, a tool is three fields: a `name`, a natural-language `description`, and an `input_schema` in [JSON Schema](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview). The model returns calls as structured `tool_use` blocks validated against that schema; your harness executes and replies with `tool_result` blocks. (OpenAI's function-calling API has the same shape with different field names; everything in this chapter is provider-neutral in substance.)

```json
{
  "name": "run_tests",
  "description": "Run the repository test suite with pytest and return the exit code plus the last 50 lines of output. Pass test_path to run one file or one test node while iterating; run the full suite only for final verification.",
  "input_schema": {
    "type": "object",
    "properties": {
      "test_path": {
        "type": "string",
        "description": "Optional pytest target, e.g. 'tests/test_invoice.py' or 'tests/test_invoice.py::test_rounding'."
      }
    },
    "required": []
  }
}
```

Three properties of this format shape everything else in the chapter.

First, **the definition lives in context, on every step**. Tool definitions are serialized into the prompt for each model call, and enabling tools also injects a tool-use system prompt of a few hundred tokens ([Anthropic publishes the per-model counts](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview)). Twenty tools with lavish descriptions can cost several thousand tokens per step before the agent has done anything. Tool definitions are the most-read documents in your system and among the most expensive; they should be written like poetry and budgeted like ad space.

Second, **every string in the schema is prompt**. The `description` on the tool, the `description` on each parameter, the parameter names themselves, even the enum values — the model reads all of it, every step, and treats it as instruction. This is why Anthropic ranks prompt-engineering tool descriptions "among the most effective methods for improving tools," and reports that Claude Sonnet 3.5 reached state-of-the-art SWE-bench performance after "precise refinements to tool descriptions." Nobody gets to skip writing; the schema *is* the writing.

Third, **the schema is a contract the platform can enforce**. Providers validate tool calls against the schema, and strict modes (e.g. `strict: true` with `additionalProperties: false` on the Anthropic API) guarantee the arguments parse exactly. Everything the schema can express — types, enums, required fields — is a class of error the model can no longer make. Everything the schema cannot express must be caught by your handler and taught back through error messages. Push as much of the contract into the schema as it can hold.

## The docstring-for-a-junior-developer rule

The most reliable heuristic for tool descriptions: write them as if onboarding a smart junior developer on their first day — someone fully capable of doing the work, who knows nothing about your conventions, your data model, or which of your two search functions is the right one. Anthropic's guidance is exactly this register: descriptions should spell out implicit context — "specialized query formats, definitions of niche terminology, relationships between resources" — that veteran team members carry in their heads and that a model has no other way to acquire. Increasingly important on recent models: say *when* to call the tool, not only what it does, because models have grown more deliberate about reaching for tools and a trigger condition in the description measurably lifts the should-call rate.

Here is the rule violated and then followed, on Ledgerbot's ticket-search tool.

**Bad:**

```json
{
  "name": "search",
  "description": "Searches tickets.",
  "input_schema": {
    "type": "object",
    "properties": {
      "q": {"type": "string"},
      "user": {"type": "string"}
    },
    "required": ["q"]
  }
}
```

Everything a junior developer would ask is unanswered. Search *which* tickets — the CI failure tracker, or the customer-support queue? What query syntax — free text, or the tracker's `field:value` language? Is `user` a login, an email, a display name, a numeric ID? What comes back, and how much of it? The model will guess, differently on different steps, and the failures will look like model flakiness when they are documentation debt.

**Good:**

```json
{
  "name": "cifail_search_tickets",
  "description": "Search the CI-failure ticket tracker (not the customer-support system). Use this when you need to know whether a failing test already has an open ticket, or to find past tickets about the same test for prior diagnoses. Query syntax is free text plus optional filters, e.g. 'test_rounding status:open' or 'flaky component:billing'. Returns at most 20 tickets, newest first, each with id, title, status, and a one-line summary; fetch full details with cifail_get_ticket.",
  "input_schema": {
    "type": "object",
    "properties": {
      "query": {
        "type": "string",
        "description": "Free text plus optional filters. Supported filters: status:(open|closed), component:<name>, test:<test node id>."
      },
      "assignee_id": {
        "type": "string",
        "description": "Optional. Numeric user ID like '4213' — not an email or username. Get IDs from cifail_list_team."
      }
    },
    "required": ["query"]
  }
}
```

The rewrite answers when to use it (and when not to), which corpus it hits, what the query language is, what comes back, what the follow-up tool is, and precisely what format `assignee_id` takes — including where to obtain one, which prevents the model from fabricating a plausible-looking email. Note also the two name changes: the tool gained a namespace prefix (`cifail_`, distinguishing it from any other search in the toolset — Anthropic found that even the choice "between prefix- and suffix-based naming [has] non-trivial effects on tool-use evaluations"), and the parameter went from `user` to `assignee_id`, an example of their advice to prefer unambiguous names — "`user_id` instead of `user`" — because the parameter name is the documentation the model sees at the exact moment it fills in the value.

The cost of the rewrite is real — it is perhaps 130 tokens heavier, resent every step — and worth stating as the general tradeoff: spend description tokens exactly where ambiguity would otherwise cost retries. A tool with one obvious string parameter needs one sentence, not a treatise.

## Poka-yoke: make misuse impossible

Poka-yoke is the manufacturing term — borrowed by [Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents) — for designing a part so it cannot be assembled wrong: the plug that only fits one way. Applied to tools: wherever a mistake is *possible*, some fraction of calls will make it, so the strongest fix is not better documentation but a signature in which the mistake cannot be expressed. Anthropic's canonical example is file paths: their SWE-bench agent kept making relative-path errors once it had `cd`'d away from the repository root, and the fix was not a prompt exhortation but changing the tools to require absolute paths — after which the model used them flawlessly.

The poka-yoke toolbox, roughly in order of power:

- **Enums over free strings.** If a parameter has five legal values, say so in the schema. `"status": {"type": "string", "enum": ["open", "closed", "wontfix"]}` deletes the entire class of `"Open"`, `"OPEN"`, `"resolved"` errors that a bare string invites — the platform's validator rejects them before your handler ever runs.
- **Formats over conventions.** `{"type": "string", "format": "date"}` beats a description saying "use YYYY-MM-DD". IDs that must be numeric strings should say so with a `pattern`.
- **Remove degrees of freedom.** Absolute paths only. One timezone (UTC) in and out. Identifiers from a designated source tool, never composed by the model.
- **Make the destructive path explicit.** If a tool can overwrite or delete, require a parameter that names the destruction — `overwrite: {"type": "boolean"}` with default false — so the dangerous act requires a deliberate, visible token in the call, which the harness can also gate on (Chapter 8 builds permission systems on exactly this hook).

Here is the pattern applied to Ledgerbot's meeting-scheduling tool (it files "flaky test triage" sessions with code owners).

**Bad — every parameter is a trap:**

```json
{
  "name": "schedule_meeting",
  "description": "Schedule a meeting.",
  "input_schema": {
    "type": "object",
    "properties": {
      "attendees": {"type": "string", "description": "Comma-separated names"},
      "time": {"type": "string", "description": "When the meeting should be"},
      "length": {"type": "number"},
      "room": {"type": "string"}
    },
    "required": ["attendees", "time"]
  }
}
```

Comma-separated names in one string (which "Smith, Jr." breaks), an unconstrained time ("tomorrow-ish" validates), a `length` in unstated units, a free-text `room` the model will populate with rooms that don't exist.

**Good — the schema does the enforcing:**

```json
{
  "name": "calendar_schedule_meeting",
  "description": "Book a triage meeting on the shared engineering calendar. All times are UTC. Attendee IDs come from cifail_list_team; do not guess them.",
  "input_schema": {
    "type": "object",
    "properties": {
      "attendee_ids": {
        "type": "array",
        "items": {"type": "string", "pattern": "^[0-9]+$"},
        "description": "Numeric user IDs from cifail_list_team."
      },
      "start_time_utc": {
        "type": "string",
        "format": "date-time",
        "description": "ISO 8601 UTC start, e.g. '2026-07-14T15:00:00Z'."
      },
      "duration_minutes": {
        "type": "integer",
        "enum": [15, 30, 45, 60],
        "description": "Meeting length in minutes."
      }
    },
    "required": ["attendee_ids", "start_time_utc", "duration_minutes"],
    "additionalProperties": false
  }
}
```

The room is gone entirely — the booking system assigns one — which is the quietest poka-yoke of all: a parameter that no longer exists cannot be filled in wrong.

## Errors that teach

When schema validation can't catch a mistake, the tool's error message is the model's only teacher — and unlike a human developer, the model cannot go read the source or ask in Slack. It has exactly the text you return. Chapter 3 established the mechanism (failures come back as `tool_result` blocks flagged `is_error: true`, and the model adapts); this section is about what to put in them.

A good tool error answers three questions in one or two sentences: *what failed*, *why*, and *what to do instead*. Compare:

**Bad:**

```
Error: request failed (500)
```

The model learns nothing. Its most likely responses are to retry the identical call (waste), abandon a viable approach (worse), or hallucinate around the gap (worst).

**Good:**

```
File not found: 'src/invoice.py'. The repository root contains 'src/billing/'
and 'src/ledger/'. Closest match: 'src/billing/invoice.py'. Use list_files
with a directory path to see what exists before reading.
```

This costs the handler a `difflib.get_close_matches` call and one directory listing, and it typically converts a dead step into a recovered one — the model's next call is the corrected path. The same principle applies to validation your handler does beyond the schema: "start_time_utc is in the past; the current UTC time is 2026-07-10T09:14:00Z" beats "invalid time," because the first version hands the model the fact it was missing.

Three further rules. **Never let a raw stack trace be the message** — fifty frames of your harness's internals is token-expensive noise that also leaks implementation detail the model may then try to reason about. Summarize, and keep the one line that matters. **Distinguish the model's errors from the world's errors.** "No ticket matches that query" is a valid result, not an error — don't flag it `is_error`, or you teach the model that a legitimate empty result means it did something wrong. Reserve the error flag for calls that were malformed or genuinely failed. **Make retryability explicit.** "The test runner timed out after 300s; re-run with a narrower test_path" tells the model the action is safe to retry *differently* — which is also a nudge toward the harness-level idempotency concerns Chapter 11 takes up.

## Return-format design and token budgets

A tool result is not a return value; it is a deposit into the context window, resent on every subsequent step of the session, competing for the model's attention against everything else there. Anthropic's tool-writing guide frames the goal as returning "high signal information" and offers a memorable concrete swap: prefer `name`, `file_type`, human-readable fields over `uuid`, `mime_type`, and other machine-natural ones — because downstream, the *model* is the consumer, and a 128-bit hex identifier is 20-odd tokens of attention poison that the model may also mistranscribe. (When later tool calls genuinely require the opaque ID, return both — the readable field for reasoning, the ID for plumbing.)

The budget techniques, from the same guide:

- **Bound everything.** Every list-returning tool needs pagination or a hard cap, with "sensible default parameter values." Claude Code caps tool responses at 25,000 tokens by default; your harness should have a number too, and it should probably be much smaller for most tools.
- **Truncate with instructions, not ellipses.** When a result is cut, say what was cut and how to get the rest: pair truncation with guidance steering the agent toward "more token-efficient strategies, like making many small and targeted searches" rather than one giant dump. A truncation notice is a teaching moment, exactly like an error message.
- **Offer a verbosity knob.** The guide's pattern: a `response_format` enum parameter with values like `"concise"` and `"detailed"`, so the agent itself chooses to pay for detail only when it needs it. Ledgerbot's `run_tests` follows the cheaper version of the idea — exit code plus the last 50 lines by default, because the tail of a pytest run is where the signal lives.

A worked pair, on Ledgerbot's log-search tool:

**Bad — the firehose:**

```python
def search_logs(query: str) -> str:
    return "\n".join(line for line in open("ci.log") if query in line)
    # 11,000 matching lines? All of them. Every future step pays for this.
```

**Good — bounded, structured, self-describing:**

```python
def search_logs(query: str, max_results: int = 20) -> str:
    hits = [f"{n}: {line.rstrip()}" for n, line in enumerate(open("ci.log"), 1)
            if query in line]
    shown = hits[:max_results]
    header = f"{len(hits)} matching lines; showing first {len(shown)}."
    if len(hits) > max_results:
        header += (" Refine the query (e.g. add the test name or a timestamp) "
                   "rather than raising max_results.")
    return header + "\n" + "\n".join(shown)
```

The count-plus-sample shape deserves to be a habit: the model learns the *scale* of the result ("11,000 lines match" is itself diagnostic — the query is too broad) at the price of twenty.

## Consolidation versus proliferation

The instinct when building a tool layer is coverage: wrap every endpoint, expose every operation. Anthropic's guidance is blunt that "more tools don't guarantee better outcomes" — the recommendation is "a few thoughtful tools targeting specific high-impact workflows," and their worked example is the consolidation pattern: instead of shipping `list_users`, `list_events`, and `create_event` and making the agent choreograph the three (spending steps, tokens, and error surface on the plumbing), ship `schedule_event`, which does the lookup-check-create dance internally in deterministic code. The heuristic: when a multi-tool sequence is *always the same*, it is not agent work — it is a subroutine wearing three tool definitions, and it should be collapsed into one. Conversely, when the composition genuinely varies with the task, keep the primitives separate; over-consolidation produces Swiss-army tools whose descriptions balloon to cover every mode.

Two forces bound the toolset from opposite ends. From below, the bash counterexample: [mini-swe-agent](https://github.com/SWE-agent/mini-swe-agent) ships *no* tools except shell access — "the focus is fully on the LM utilizing the shell to its full potential" — and scores over 74% on SWE-bench Verified, because for expert-distribution work like software engineering, the shell is an interface the model already knows deeply. From above, the observability requirement: a bash-only agent gives the harness one opaque string per action — nothing to validate, gate, or render. The working synthesis for production harnesses: start broad (bash or equivalent) for capability, then *promote* an action to a dedicated tool when the harness needs a typed hook into it — to permission-gate it (Chapter 8), to enforce an invariant the shell can't (an edit tool that rejects writes to files the agent never read), to mark it parallel-safe for the scheduling concerns of Chapter 3, or to render it meaningfully in a UI. The toolset, in other words, is not a list of capabilities; it is a list of *contracts* the harness intends to enforce.

Whatever the size, organize it. Namespacing related tools under common prefixes (`cifail_search_tickets`, `cifail_get_ticket`, `calendar_schedule_meeting`) helps "delineate boundaries between lots of tools," reduces cross-service confusion, and gives the model a navigable taxonomy instead of a flat bag of verbs.

## The Model Context Protocol

Everything so far assumed you build the tools yourself. The [Model Context Protocol](https://modelcontextprotocol.io/) (MCP), released as an open standard by Anthropic in late 2024 and since adopted across the ecosystem — Claude, ChatGPT, VS Code, Cursor, and many other hosts — standardizes the other case: connecting an agent to tools that someone else maintains. The project's own analogy is "a USB-C port for AI applications": one connector specification, so a tool server built once works in every compliant host.

The [architecture](https://modelcontextprotocol.io/docs/learn/architecture) has three participants. An **MCP host** is the AI application — the harness, in this book's terms. The host creates one **MCP client** per connection, and each client maintains a dedicated connection to one **MCP server**, a program that provides context and capabilities. The protocol splits into a **data layer** — a [JSON-RPC 2.0](https://www.jsonrpc.org/) message protocol covering lifecycle and primitives — and a **transport layer** with two standard options: **stdio**, where the host launches the server as a local subprocess and speaks over its standard streams, and **streamable HTTP**, where the server is remote, messages go over HTTP POST with optional server-sent events for streaming, and authorization uses standard HTTP mechanisms (the spec recommends OAuth for obtaining tokens).

A connection begins with lifecycle negotiation: the client sends an `initialize` request carrying its protocol version (a date-stamped string such as `"2025-06-18"`) and its capabilities; the server replies with its own; incompatible versions terminate the connection. MCP is a stateful protocol, and this handshake is what lets both sides know exactly which features are on the table.

Servers can then expose three primitives, and the distinction matters when you design or evaluate a server:

- **Tools** — executable functions the agent invokes to act. Discovered with `tools/list`, which returns each tool's `name`, `description`, and `inputSchema` — the same three-part anatomy from earlier in this chapter, which is not a coincidence; MCP standardized the prevailing shape. Invoked with `tools/call`, which returns a `content` array of typed blocks (text, images, resources).
- **Resources** — data sources providing contextual information (file contents, database records) without side effects.
- **Prompts** — reusable interaction templates the host can offer its user.

Clients can expose primitives back: **sampling** (`sampling/createMessage`) lets a server request an LLM completion *through the host*, so server authors get model access without bundling a model SDK or key; **elicitation** lets a server ask the host's human for input or confirmation; **logging** sends diagnostics to the client. Finally, the protocol supports notifications — a server whose toolset changes at runtime declares `listChanged` capability and emits `notifications/tools/list_changed`, prompting the client to re-fetch `tools/list`; discovery is dynamic by design.

Two harness-engineering implications, both flowing from a single fact: MCP standardizes the *plumbing* of tool access, not the *quality* of it. First, everything in this chapter still applies to MCP servers you write — a server whose tools have one-line descriptions and free-string parameters is a bad ACI with excellent interoperability. Second, connecting servers you did *not* write imports their tool definitions into your context budget and their behavior into your trust boundary. A handful of servers can quietly add tens of thousands of tokens of definitions to every step — the tool-search/deferred-loading patterns some platforms offer exist precisely to manage this — and every third-party tool description is text you are injecting into your own model's instructions. Chapter 8 treats the supply-chain side of that in earnest.

## Code execution as a tool interface

The chapter so far assumed the default dataflow: every tool result passes through the model's context. For pipelines that move real data, that dataflow is the bottleneck, and Anthropic's engineering write-up [Code execution with MCP](https://www.anthropic.com/engineering/code-execution-with-mcp) quantifies it with a scenario: an agent copying a sales-meeting transcript from Google Drive into a Salesforce record. Called as direct tools, the transcript "flows through twice" — once as the Drive tool's result, once as the Salesforce tool's argument — and "for a 2-hour sales meeting, that could mean processing an additional 50,000 tokens" that the model never needed to *read*, only to *move*. Multiply by hundreds of connected tool definitions loaded up front and agents can be "process[ing] hundreds of thousands of tokens before reading a request."

The alternative interface: give the agent a code-execution environment and "present MCP servers as code APIs" — filesystem-organized modules the agent can discover, import, and compose. Instead of two tool round-trips through context, the agent writes a short script; the transcript flows from Drive to Salesforce *inside the sandbox*, and only the outcome ("record updated, 18,402 characters") returns to the model. On the same scenario, Anthropic reports the token usage dropping "from 150,000 tokens to 2,000 tokens — a time and cost saving of 98.7%." The agent also regains real control flow — loops, filters, retries — executed deterministically rather than one attention-mediated step at a time, and it loads only the tool definitions it actually needs by exploring the API surface on demand.

This is the same insight behind mini-swe-agent's bash-only design, arrived at from the enterprise-integration direction, and it genuinely reframes the tool layer: for data-heavy work, the best tool interface may be *one* tool — an interpreter — plus well-designed libraries. But note what it does not change. Someone still writes those code APIs, and every principle in this chapter transfers with the medium: function names are still the vocabulary, docstrings are still the most-read documentation, exceptions are still the errors that must teach, return values are still the token budget. And it raises the stakes for Chapter 7, because "the agent writes and runs arbitrary programs" is a sandboxing requirement, not a footnote.

## Testing and iterating on tools

Tools are software with an unusual property: their primary consumer is stochastic. You cannot unit-test your way to a good ACI, because the failure mode is not "the tool returns the wrong value" but "the model uses the tool wrongly, or not at all." The discipline that works, per Anthropic's tool-writing guide, is evaluation-driven:

**Build realistic multi-step evaluation tasks.** Not smoke tests — tasks drawn from real workloads that "require multiple tool calls — potentially dozens," with verifiable outcomes. For Ledgerbot: a repository fixture with a genuinely failing test, graded on whether the suite is green afterward. Weak, single-call eval prompts systematically overstate tool quality because the pathologies — identifier confusion, budget blowouts, retry loops — only emerge across steps.

**Read the transcripts.** The metrics tell you *that* a task failed; only the transcript tells you *why*, and the why is usually specific and fixable: the model passed an email where `assignee_id` wanted a number (rename the parameter, tighten the pattern); it called the full test suite eleven times (the description now says to prefer targeted runs); it hallucinated a `search_code` tool that doesn't exist (the gap in the toolset is itself a finding). This is the transcripts-as-ground-truth discipline of Chapter 11 applied at design time.

**Let the agent critique its own tools.** A distinctive move from the guide: after an evaluation run, feed the transcript back to the model and ask what about the tools confused it — and even let it propose rewrites of the descriptions. Anthropic reports iterating this way agent-assisted; the model that has to *use* the interface turns out to be a competent reviewer of it.

**Hold the improvements with regression evals.** Tool descriptions are prompts, and prompts regress silently — a description tuned for one model generation may over- or under-trigger on the next. Keep the evaluation suite runnable and re-run it on every tool change and every model upgrade. Chapter 12 builds out this flywheel; Chapter 14 explains why the tool layer is one of the harness components most worth re-auditing as models improve, because every workaround baked into a description encodes an assumption about model weakness that will eventually be false.

The through-line of this chapter is that none of this is exotic engineering. Schemas, docstrings, error messages, pagination, namespaces — every technique is familiar from ordinary API design. What changes is the addressee. You are writing an interface for a consumer that reads everything, remembers nothing between sessions, takes every word literally, and pays for attention by the token. Design for that consumer deliberately, and the model spends its capability on the task. Design for it carelessly, and the model spends its capability compensating for you.

## Further reading

- John Yang et al., "SWE-agent: Agent-Computer Interfaces Enable Automated Software Engineering" — [arxiv.org/abs/2405.15793](https://arxiv.org/abs/2405.15793)
- Anthropic Engineering, "Writing effective tools for agents" — [anthropic.com/engineering/writing-tools-for-agents](https://www.anthropic.com/engineering/writing-tools-for-agents)
- Anthropic Engineering, "Building Effective Agents" (Appendix 2, "Prompt engineering your tools") — [anthropic.com/engineering/building-effective-agents](https://www.anthropic.com/engineering/building-effective-agents)
- Anthropic Engineering, "Code execution with MCP" — [anthropic.com/engineering/code-execution-with-mcp](https://www.anthropic.com/engineering/code-execution-with-mcp)
- Model Context Protocol — introduction and architecture — [modelcontextprotocol.io](https://modelcontextprotocol.io/) and [modelcontextprotocol.io/docs/learn/architecture](https://modelcontextprotocol.io/docs/learn/architecture)
- Model Context Protocol — specification — [modelcontextprotocol.io/specification/latest](https://modelcontextprotocol.io/specification/latest)
- Anthropic, "Tool use with Claude" (API documentation) — [platform.claude.com/docs/en/agents-and-tools/tool-use/overview](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview)
- SWE-agent team, mini-swe-agent — [github.com/SWE-agent/mini-swe-agent](https://github.com/SWE-agent/mini-swe-agent)
- Firecrawl, "What Is an Agent Harness?" — [firecrawl.dev/blog/what-is-an-agent-harness](https://www.firecrawl.dev/blog/what-is-an-agent-harness)

---

[← The Agent Loop](ch03-the-agent-loop.md) · [Context Engineering →](ch05-context-engineering.md)
