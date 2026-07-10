# Chapter 6 — Memory, State, and Sessions

*Chapter 5 established that the context window is working memory: fast, expensive, and wiped whenever a session ends. This chapter builds everything on the other side of that boundary — the machinery that lets an agent's work survive its own context window. We separate three kinds of memory that get conflated under one word, show why structured state files beat prose for anything a machine must act on, and work a complete checkpoint/resume example: a JSON state schema plus the Python code that reads it, writes it atomically, and re-enters a task idempotently. We then develop the initializer-executor pattern for multi-session work and close with the discipline that makes all of it debuggable: the transcript as ground truth.*

## Three kinds of memory

The word "memory" hides three different engineering problems, with different lifetimes, different storage, and different failure modes. The taxonomy — used, among others, by Firecrawl's overview of agent harnesses ([What Is an Agent Harness?](https://www.firecrawl.dev/blog/what-is-an-agent-harness)) — is worth fixing precisely:

- **Working context** is the content of the context window on the current model call: the system prompt, history, and tool results Chapter 5 budgeted. Lifetime: one step to one session. Storage: none — it exists only in the request.
- **Session state** is what a specific task has accomplished and what remains: the plan, the checklist, the current hypothesis, the files touched. Lifetime: one task, potentially spanning many sessions. Storage: files (or a database) the harness owns.
- **Long-term memory** is knowledge that outlives any task: project conventions, past decisions, lessons learned, user preferences. Lifetime: indefinite. Storage: durable files or a memory store, curated deliberately.

A **session**, in this book's vocabulary, is one continuous run of the agent loop; it may be resumed. The relationships among the three memories define the harness's job: working context is *assembled from* session state and long-term memory at the start of each session, and session state and long-term memory are *updated from* what happens in working context before the session ends. Firecrawl compresses the design consequence into a slogan: "memory isn't a plugin, it's the harness." You cannot bolt persistence onto an agent as an afterthought, because every other component — the loop's stop conditions, the tools, the context assembly, the resume path — depends on where state lives and who is allowed to write it.

The failure mode when the taxonomy is ignored is familiar to anyone who has run a coding agent overnight. The agent does four hours of good work, the session dies — crash, rate limit, window exhaustion — and the next session starts with no memory of what came before. Anthropic's engineering write-up on long-running agents opens with exactly this scenario and an apt analogy: engineers working shifts with no handoff notes, each shift rediscovering the project from scratch ([Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)). The rest of this chapter is, in effect, the design of the handoff note.

## Durable artifacts as inter-session memory

The simplest persistence mechanism is also the best one: ordinary files that both the agent and the harness can read and write. Plans, todo lists, progress journals, decision logs — these are not documentation *about* the work; they are the medium through which one session communicates with the next. Ledgerbot, this book's running example — the agent that triages and fixes failing CI builds in a mid-size Python monorepo — keeps a `plan.md` for humans, a `state.json` for machines (schema below), and a `journal.md` where each session appends a few lines about what it did and why. A fresh session reads all three in its first thousand tokens and is oriented.

Anthropic's memory tool for the Claude API productizes the pattern. The tool is client-side: the model issues file operations — `view`, `create`, `str_replace`, `insert`, `delete`, `rename` — against a `/memories` directory, and *your* handler executes them against storage you control ([memory tool docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool)). Two details of its design are instructive independent of the specific API. First, the system prompt the API injects when the tool is present tells the model to check memory before doing anything else and to record progress as it works, ending with a line every harness designer should tape to the monitor: "ASSUME INTERRUPTION: Your context window might be reset at any moment, so you risk losing any progress that is not recorded in your memory directory." Interruption is not an edge case for agents; it is the steady state, and memory design that assumes otherwise fails on contact. Second, because the handler is client-side, the security burden is the harness's: every path must be validated against traversal (`/memories/../../secrets.env` is the canonical attack), which previews Chapter 8's rule that agent-facing storage is a trust boundary like any other.

Instruction files are the long-term end of the same spectrum. A `CLAUDE.md` checked into the repository root — documenting build commands, code style, testing conventions, known gotchas — is long-term memory that every session loads at startup ([Claude Code best practices](https://www.anthropic.com/engineering/claude-code-best-practices)). It is also *versioned* memory: it travels with the code, gets reviewed in pull requests, and improves monotonically as the team appends lessons. When Ledgerbot repeatedly wastes a session rediscovering that the payments service's tests need a local Redis, the durable fix is one line in the instruction file, not a bigger context window.

## Why JSON beats prose for state

Session state could live in Markdown — it is legible, and models write it fluently. Production experience says not to. Anthropic's long-running-agent harness stores its feature list as JSON specifically because "the model is less likely to inappropriately change or overwrite JSON files compared to Markdown files" ([Effective harnesses](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)); Firecrawl reports the same choice for the same reason — models treat prose as editable and structure as load-bearing. A model asked to update a Markdown checklist will occasionally "improve" it: rewording items, merging duplicates, deleting entries it judges obsolete. Each improvement is a small act of state corruption.

Prose has three further defects as a state medium. It cannot be validated — there is no schema check that catches a missing field in a paragraph. It cannot be reliably diffed — you cannot write a monitor that alerts when `attempts > 3` if attempts is a narrative. And it invites summarization drift — each rewrite is a lossy re-encoding, the same compounding loss that makes repeated compaction dangerous in Chapter 5.

So the rule: **prose for humans and for the model's narrative memory; JSON for anything the harness must act on.** The journal is Markdown. The checklist is JSON. And the JSON earns its keep by following a few schema disciplines: carry a `schema_version` so old state files are detected rather than misread; make every status an enum, not free text; record `attempts` and timestamps so the harness can enforce give-up policies; and keep a `verified` flag distinct from "done" — the gap between *claimed* done and *verified* done is where agents lie to themselves, as Chapter 10 examines at length.

## Checkpoint and resume: a worked example

Here is the pattern end to end for Ledgerbot. The task: CI run #4812 has five failing tests; fix them. The state file is the checkpoint — the single document from which any future session can reconstruct where the task stands.

The schema, as a valid instance:

```json
{
  "schema_version": 2,
  "task_id": "ci-fix-4812",
  "goal": "Make CI green on branch fix/ci-4812 (run #4812, 5 failures)",
  "created_at": "2026-07-10T09:14:03Z",
  "updated_at": "2026-07-10T11:42:51Z",
  "branch": "fix/ci-4812",
  "status": "in_progress",
  "failures": [
    {
      "id": "tests/payments/test_refunds.py::test_partial_refund",
      "status": "fixed",
      "verified": true,
      "attempts": 1,
      "hypothesis": "Rounding: Decimal quantize missing on split refunds",
      "fix_commit": "a1c9f04",
      "notes": "Fixed by quantizing to cents in RefundSplitter.split()"
    },
    {
      "id": "tests/payments/test_invoices.py::test_overdue_flag",
      "status": "in_progress",
      "verified": false,
      "attempts": 2,
      "hypothesis": "Timezone: test assumes UTC, CI runs in UTC but code uses local now()",
      "fix_commit": null,
      "notes": "First attempt (freeze_time) broke two other tests; reverted."
    },
    {
      "id": "tests/billing/test_proration.py::test_leap_year",
      "status": "pending",
      "verified": false,
      "attempts": 0,
      "hypothesis": null,
      "fix_commit": null,
      "notes": null
    }
  ]
}
```

Every field is there for a resume-time reason. `status` plus `verified` tells the next session what to trust. `attempts` lets the harness stop an agent from burning a fourth session on a test that has defeated three. `hypothesis` and `notes` carry the *reasoning* forward — without them, session N+1 re-derives session N's diagnosis at full token price. `fix_commit` ties the state to the other checkpoint system in play, git, of which more below.

The harness code — reading, atomic writing, and re-entry — fits in a page of standard-library Python:

```python
"""Ledgerbot checkpoint/resume: state file I/O and session re-entry."""
import json
import os
import tempfile
from datetime import datetime, timezone
from pathlib import Path

SCHEMA_VERSION = 2

def load_state(path: Path) -> dict:
    """Read and validate the checkpoint. Fail loudly on mismatch."""
    state = json.loads(path.read_text(encoding="utf-8"))
    if state.get("schema_version") != SCHEMA_VERSION:
        raise ValueError(
            f"{path}: schema v{state.get('schema_version')}, "
            f"expected v{SCHEMA_VERSION} — run the migration, don't guess."
        )
    return state

def save_state(path: Path, state: dict) -> None:
    """Atomic write: tmp file + fsync + rename. A crash mid-write must
    leave the previous checkpoint intact, never a truncated one."""
    state["updated_at"] = datetime.now(timezone.utc).isoformat()
    fd, tmp = tempfile.mkstemp(dir=path.parent, prefix=".state-")
    try:
        with os.fdopen(fd, "w", encoding="utf-8") as f:
            json.dump(state, f, indent=2, sort_keys=True)
            f.flush()
            os.fsync(f.fileno())
        os.replace(tmp, path)          # atomic on POSIX
    except BaseException:
        os.unlink(tmp)
        raise

def next_pending(state: dict, max_attempts: int = 3) -> dict | None:
    """Pick one unit of work: unverified, under the attempt cap."""
    for f in state["failures"]:
        if not f["verified"] and f["attempts"] < max_attempts:
            return f
    return None

def build_resume_prompt(state: dict, failure: dict) -> str:
    """First user message of a resumed session: orient, then narrow."""
    done = [f["id"] for f in state["failures"] if f["verified"]]
    return (
        f"You are resuming task {state['task_id']}: {state['goal']}\n"
        f"Branch: {state['branch']}. "
        f"Already fixed and verified: {done or 'none'}.\n\n"
        f"Work on exactly ONE failure this session:\n"
        f"  test: {failure['id']}\n"
        f"  prior attempts: {failure['attempts']}\n"
        f"  last hypothesis: {failure['hypothesis'] or 'none yet'}\n"
        f"  notes from last session: {failure['notes'] or 'none'}\n\n"
        f"Before changing anything: run this one test and confirm it "
        f"still fails as described. Trust the world over the notes."
    )

def run_one_session(state_path: Path, run_agent) -> bool:
    """One executor session. Returns True if any work remains."""
    state = load_state(state_path)
    failure = next_pending(state)
    if failure is None:
        state["status"] = "done" if all(
            f["verified"] for f in state["failures"]) else "needs_human"
        save_state(state_path, state)
        return False

    failure["attempts"] += 1
    failure["status"] = "in_progress"
    save_state(state_path, state)          # checkpoint BEFORE the attempt

    outcome = run_agent(build_resume_prompt(state, failure))  # Chapter 3's loop

    failure["hypothesis"] = outcome.hypothesis
    failure["notes"] = outcome.notes
    if outcome.test_passed and outcome.suite_passed:   # harness-run, not agent-claimed
        failure.update(status="fixed", verified=True,
                       fix_commit=outcome.commit_sha)
    save_state(state_path, state)          # checkpoint AFTER, win or lose
    return True
```

Three deliberate choices deserve narration. The write path is *atomic* — temp file, `fsync`, `os.replace` — because a checkpoint that can be corrupted by a crash is worse than no checkpoint: it converts "lost some progress" into "resumed from garbage." The checkpoint is written *before* the attempt as well as after, so the `attempts` counter survives even a hard kill mid-session — otherwise a crashing test could be retried forever. And the `verified` flag is set from the harness's own test run (`outcome.test_passed and outcome.suite_passed`), never from the model's claim that it fixed something; the state file records observations, not assertions.

## Idempotent re-entry

Resume logic has one governing requirement: running it twice must be safe. Sessions die at arbitrary points — after the code change but before the commit, after the commit but before the checkpoint write — so the resume path will inevitably re-enter work that is half done. **Idempotent re-entry** means the session converges on the correct state regardless of where its predecessor stopped.

The core rule is visible in `build_resume_prompt` above: *trust the world over the notes.* The state file says `test_overdue_flag` is in progress; the first action of the resumed session is to run that test and see. Maybe the previous session actually fixed it and died before checkpointing — then the test passes, the session marks it verified, and no work is duplicated. Maybe the working tree holds a half-applied edit — then the test fails differently than described, which is itself vital information. The state file is a *claim* about the world; the world is the authority. Anthropic's long-running-agent harness encodes the same principle as a standing session preamble: read the git log and progress notes, then run a basic health check — start the app, exercise a few core flows — *before* touching anything new, because "undocumented bugs" left by a dead session otherwise get built upon ([Effective harnesses](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)).

Git is the second half of idempotent re-entry, and it is no accident that every serious coding-agent harness leans on it. Commits are checkpoints of the *artifact* just as the state file is a checkpoint of the *task*: content-addressed, atomic, cheap, and reversible. The discipline that makes them useful is commit-per-unit-of-work with descriptive messages — one failing test fixed, one commit, referenced by SHA in the state file — so that a resumed session can correlate task state with artifact state, and a bad fix can be reverted without archaeology. A dirty working tree at session start is a red flag the resume prompt should surface explicitly: it means the last session died mid-edit, and the first decision is whether to keep, stash, or discard the residue.

The remaining rules of idempotent re-entry are all instances of one idea — make each session's unit of work small enough that redoing it is cheap. One failure per session for Ledgerbot. One feature per session in Anthropic's harness. If the unit of work is four hours long, a crash costs four hours; if it is twenty minutes, idempotency is almost free.

## The initializer-executor pattern

Everything so far assumed the state file exists. Someone has to create it — and the session that creates it has a fundamentally different job from the sessions that consume it. That observation, generalized, is the **initializer-executor pattern**, the standard architecture for multi-session agent work.

The **initializer** runs once, at task start, and does no substantive work on the task itself. Its entire job is to build the environment in which future sessions can work statelessly. Firecrawl's description is compact: the initializer "sets up the durable project environment: folder structure, feature list, init.sh, initial git commit," after which every executor session "reads from this environment, makes incremental progress on one feature, runs tests, commits, updates the progress file, and exits cleanly" ([What Is an Agent Harness?](https://www.firecrawl.dev/blog/what-is-an-agent-harness)). Anthropic's harness for building a full web application shows the pattern at scale: the initializer agent decomposed the goal into a feature list of *over two hundred* end-to-end requirements, each with a category, a human-readable description, numbered verification steps, and a boolean `passes` field initialized to `false`; wrote an `init.sh` that starts the dev environment and runs a smoke test; and made the baseline git commit ([Effective harnesses](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)).

The **executor** then runs as many times as needed, each session following the same script. Anthropic's version, condensed: orient (read git log and progress file), health-check (run `init.sh`, confirm fundamentals still work), select exactly one incomplete feature, implement it, verify it end to end, commit, update the progress file, exit cleanly. The one-feature rule "turned out to be critical" — it is the small-unit-of-work principle from the previous section, institutionalized. And the verification step is guarded by a hard rule in the feature list itself — "it is unacceptable to remove or edit tests" — because the observed failure mode was agents marking features complete without testing them, or quietly deleting the criteria they couldn't meet.

The pattern's deeper logic is a separation of concerns across *time*. The initializer is run when the context is cheapest and cleanest — nothing has happened yet, the whole window is available for understanding the goal and structuring the work. Executors then never need the full goal in context; they need the state file, one unit of work, and the tools. This is Chapter 5's reset strategy given an architecture: instead of one heroic session fighting context rot for six hours, a chain of short sessions, each starting fresh from durable state, each ending at a checkpoint. In harness code, the shape is a plain loop — deterministic orchestration around nondeterministic sessions, a theme Chapter 9 develops:

```python
def run_task(task_dir: Path, run_agent) -> None:
    state_path = task_dir / "state.json"
    if not state_path.exists():
        run_initializer(task_dir, run_agent)   # once: scaffold + state + baseline commit
    while run_one_session(state_path, run_agent):  # executor sessions until done
        pass
```

Note what is *not* in the loop: no summary handoff from session to session, no growing context, no compaction. Each `run_one_session` is independently restartable, and killing the process at any point loses at most one unit of work.

## Transcripts as ground truth

State files record where the task stands. They do not record *how it got there* — and when an agent does something inexplicable, "how" is the only question that matters. The record that answers it is the **transcript**: the complete, ordered log of every model call, every tool invocation, and every result, exactly as they occurred.

The discipline this book calls *transcript-as-ground-truth* has three clauses. The transcript is **append-only**: events are written as they happen and never edited, because an edited log is testimony, not evidence. It is **written through**: flushed and fsynced per event, so that a `kill -9` leaves a readable record up to the final action rather than an empty buffer. And it is **the arbiter**: when the state file, the agent's summary, and the transcript disagree, the transcript wins, because it is the only artifact that was not produced by summarization.

The implementation is deliberately boring — JSON Lines, one event per line, flushed:

```python
import json, os
from datetime import datetime, timezone
from pathlib import Path

def append_event(transcript: Path, event: dict) -> None:
    event["ts"] = datetime.now(timezone.utc).isoformat()
    with transcript.open("a", encoding="utf-8") as f:
        f.write(json.dumps(event, ensure_ascii=False) + "\n")
        f.flush()
        os.fsync(f.fileno())

# In the agent loop (Chapter 3), around each step:
# append_event(t, {"type": "model_response", "content": blocks, "usage": usage})
# append_event(t, {"type": "tool_result", "tool": name, "output": truncated})
```

JSONL is the format of choice for a reason: it is appendable without rewriting, each line is independently parseable (a truncated final line from a crash costs one event, not the file), and it greps. Claude Code stores every session exactly this way — a `.jsonl` file per session under `~/.claude/projects/<project>/`, one JSON object per message, tool use, or metadata event ([Manage sessions](https://code.claude.com/docs/en/sessions)) — and builds its resume features directly on top: `--continue` reopens the most recent session and `--resume` offers a picker, reconstructing working context from the transcript. That is the deeper payoff: a full transcript makes *the session itself* resumable. A crashed session can be relaunched with its conversation history rebuilt from the log — resume-from-transcript, which Chapter 11 develops as a reliability primitive alongside retries and backoff — and a confusing session can be replayed step by step to find the exact tool result that sent the agent sideways.

Framework-based harnesses reach the same destination with different machinery. LangGraph, for example, builds persistence in as *checkpointers*: the graph's state is snapshotted at every super-step into a *thread*, and any thread can be inspected, resumed from its latest checkpoint, or even forked from an earlier one — with a separate cross-thread store for long-term memory ([LangGraph persistence](https://langchain-ai.github.io/langgraph/concepts/persistence/)). The vocabulary differs — checkpoints and threads rather than transcripts and sessions — but the architecture is the one this chapter has been building by hand: durable, append-only history as the substrate, resume and time-travel as features on top of it.

One caution keeps the layers honest: the transcript is ground truth for *what happened*, not a substitute for session state. Rebuilding "which tests are fixed" by parsing ten megabytes of transcript is possible and miserable. The division of labor stands — transcript for evidence and resume, state file for decisions, journal for narrative — and each is derived independently from events, never from one another.

## Curating long-term memory

Session state has a natural end: the task completes and `state.json` is archived. Long-term memory has no such boundary, which makes it the memory tier most prone to rot. Two disciplines keep it useful.

**Write distilled lessons, not raw history.** The instruction file should say "payments tests require a local Redis; `make redis-dev` starts one," not narrate the four sessions it took to learn that. Long-term memory earns its context-window cost only if it is denser than what the agent could rediscover — every entry is a purchase measured in tokens-per-session forever after, which is why the highest-leverage memory write is often a *deletion*.

**Make memory auditable and correctable.** Anything the agent writes to its own long-term memory will eventually include a wrong conclusion, and a wrong "lesson" replays into every future session as confident fact. The mitigations are structural: keep memory in version control where humans review diffs; prefer per-project memory over global memory so blast radius stays contained; and never let secrets in — memory files are replayed into future contexts by design, which makes them an exfiltration channel of exactly the kind Chapter 8 dissects.

## Putting it together

Ledgerbot's memory architecture, assembled from the chapter:

1. **Working context** is rebuilt fresh each session from the pieces below, per Chapter 5's budget — never carried over.
2. **Session state** lives in `state.json`: versioned schema, enum statuses, attempt counters, a `verified` flag only the harness may set, written atomically at every transition.
3. **The initializer** creates the branch, the state file from the CI failure list, and the baseline commit; **executors** fix one verified failure per session and exit cleanly.
4. **Re-entry is idempotent**: every session re-runs its target test before editing, treats the state file as a claim and the world as the authority, and inherits at most one unit of lost work from a crash.
5. **Transcripts** stream to JSONL with per-event fsync; they are the evidence for every postmortem and the substrate for resume.
6. **Long-term memory** is a reviewed, versioned instruction file plus a small per-repo lessons file — distilled, auditable, secret-free.

The pattern to carry forward is the inversion it represents. A naive agent keeps its intelligence in the context window and treats disk as an afterthought; a well-harnessed agent keeps its knowledge on disk and treats the window as a cache. Sessions become cheap, disposable compute over durable state — and once sessions are disposable, everything else in this book gets easier: verification can grade artifacts instead of vibes (Chapter 10), orchestrators can parallelize executors (Chapter 9), and reliability becomes a matter of resuming from checkpoints rather than praying nothing crashes (Chapter 11).

## Further reading

- Anthropic Engineering, [Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents) — the initializer/coding-agent architecture, feature-list JSON, init.sh, git-based recovery.
- Anthropic Engineering, [Harness design for long-running application development](https://www.anthropic.com/engineering/harness-design-long-running-apps) — session boundaries, context resets, evaluator sessions.
- Firecrawl, [What Is an Agent Harness?](https://www.firecrawl.dev/blog/what-is-an-agent-harness) — the memory taxonomy, initializer-executor pattern, JSON state files.
- Anthropic, [Memory tool documentation](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool) — client-side memory operations, the "assume interruption" protocol, path-traversal protections.
- Anthropic Engineering, [Claude Code best practices](https://www.anthropic.com/engineering/claude-code-best-practices) — CLAUDE.md as versioned long-term memory.
- Claude Code documentation, [Manage sessions](https://code.claude.com/docs/en/sessions) — JSONL session transcripts, `--continue` and `--resume`.
- LangGraph documentation, [Persistence](https://langchain-ai.github.io/langgraph/concepts/persistence/) — checkpointers, threads, state snapshots, and time travel.
- Anthropic Engineering, [Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) — structured note-taking and just-in-time retrieval, the context-side view of the same machinery.

---

[← Context Engineering](ch05-context-engineering.md) · [Execution Environments and Sandboxing →](ch07-execution-environments.md)
