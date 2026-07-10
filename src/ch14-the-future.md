# Chapter 14 — The Disappearing Harness

*Every component in a harness is a claim about what the model cannot do, and every new model generation puts those claims on trial. This closing chapter takes that observation seriously as an engineering discipline. We connect it to Sutton's bitter lesson — the seventy-year pattern of built-in human knowledge winning short-term and losing long-term to general methods that scale — and examine the evidence that the pattern now runs through harness engineering itself: an agent-computer interface dissolved into a bash prompt within a year; sprint decomposition deleted in a single model upgrade. Then we draw the line that keeps the discipline from being self-liquidating: capability compensations erode, but trust machinery — sandboxing, permissions, verification, audit — endures, because it solves problems that better models do not solve and in some ways sharpen. The chapter closes with a practical protocol for re-auditing a harness at each model generation, and with the long-term job description of the harness engineer, whose work does not disappear so much as migrate to a permanently moving frontier.*

## Every component is a claim about the model

Strip any harness down to its parts and each part turns out to be a sentence about a model weakness, with a date on it.

Consider Ledgerbot, this book's running example — the agent that triages and fixes failing CI builds in a Python monorepo. By Chapter 12 its harness had accumulated: a compaction pass that summarizes history at 70% context occupancy (claim: *the model degrades in long contexts*); a task decomposer that splits a migration into per-module chunks (claim: *the model loses coherence on multi-hour work*); a lint gate inside the edit tool (claim: *the model writes syntax errors and spirals on the traceback*); a search tool that caps results at 50 (claim: *the model floods its own window*); a rule in the system prompt to re-read the failing test before every fix attempt (claim: *the model forgets the goal*); a fresh-context reviewer that grades diffs (claim: *the model overpraises its own work*); a Docker sandbox with no network (claim: —).

Stop at the last one. That claim is not about the model at all. Whether the model is weak or superhuman, Ledgerbot runs code from a repository whose contents Ledgerbot's operators do not fully control, and the blast radius of a malicious or confused action must be bounded by something the action cannot argue with. The sandbox encodes a fact about *trust*, not a fact about *capability*. Hold that distinction; the whole chapter turns on it.

The capability claims, by contrast, are all dated. Each was true of some model, measured (if we were disciplined) against some evaluation suite, at some moment. Anthropic's engineering team, writing about their long-running app-development harness (the one Chapter 13 tears down), put it in one sentence that deserves to be printed above every harness repository: ["every component in a harness encodes an assumption about what the model can't do on its own, and those assumptions are worth stress testing, both because they may be incorrect, and because they can quickly go stale as models improve"](https://www.anthropic.com/engineering/harness-design-long-running-apps).

Stale scaffolding is not merely dead weight. A compensation designed for a weaker model actively constrains a stronger one: pagination hides context the model could now hold; forced decomposition interrupts plans the model could now sustain; a mandatory re-read ritual burns tokens the model no longer needs to spend. The cost curve of a harness component is a parabola — negative value before the model needs it (you built too early), positive through its useful life, negative again after the model outgrows it. Harness engineering, done over years, is the practice of tracking a fleet of components across that parabola.

## The bitter lesson comes for the harness

In 2019 Richard Sutton condensed the history of AI into a short essay whose opening line has become the field's most-quoted sentence: ["The biggest lesson that can be read from 70 years of AI research is that general methods that leverage computation are ultimately the most effective, and by a large margin."](http://www.incompleteideas.net/IncIdeas/BitterLesson.html) The pattern he describes repeats across computer chess, Go, speech recognition, and vision: researchers build human knowledge of the domain into their systems; this "always helps in the short term," but "in the long run it plateaus and even inhibits further progress," and "breakthrough progress eventually arrives by an opposing approach based on scaling computation by search and learning" — the two methods, Sutton notes, "that seem to scale arbitrarily."

Read with 2019 eyes, the essay is about feature engineering and hand-coded heuristics. Read with today's eyes, a harness is exactly the kind of artifact Sutton is describing. A task decomposer is our theory of how software problems should be broken down. A curated tool interface is our theory of how a repository should be navigated. A compaction prompt is our theory of what matters in a transcript. All of it is human knowledge about the domain, built into the system because the learned component wasn't yet strong enough to supply it — and all of it, on Sutton's account, is a short-term winner and a long-term liability.

The uncomfortable part for the working engineer is that both halves of Sutton's claim are true *simultaneously*. Scaffolding wins now. If your model, today, cannot sustain a two-hour refactor, a sprint structure is not a philosophical error — it is the difference between shipping and not shipping, and refusing to build it out of bitter-lesson piety just means losing to a team that built it. Anthropic's own agent-design guidance points the same direction from the other side: ["find the simplest solution possible, and only increasing complexity when needed"](https://www.anthropic.com/engineering/building-effective-agents) — complexity being justified exactly when it "demonstrably improves outcomes." The bitter lesson does not say scaffolding never pays; it says the payoff has an expiration date, and that teams reliably overestimate how far away that date is because the scaffolding is *their* knowledge, lovingly encoded.

There is also a real disanalogy worth keeping in view, because it prevents the lazy conclusion that all harness work is doomed. Sutton's examples are systems where the built-in knowledge *competed* with learning for the same job — chess evaluation functions versus search, phonemes versus end-to-end speech models. Much of a harness does not compete with the model for a job; it does jobs the model cannot do in principle, at any capability level, because they are not reasoning jobs. No amount of intelligence lets a process grant itself kernel-enforced filesystem permissions, produce ground truth about whether its own code passes the tests, or serve as an audit trail admissible to a security team. The bitter lesson erodes the harness's *cognitive* components. It does not touch the *institutional* ones. The rest of this chapter is about learning to tell them apart before the model does it for you.

## The evidence: watching scaffolding dissolve in public

This would be comfortable armchair theory if the field had not, unusually, run the experiment in public — twice, with results documented by the people whose scaffolding dissolved.

**SWE-agent to mini-SWE-agent.** Chapter 13 told this story as a case study; here it is as data. In 2024, the SWE-agent paper [demonstrated](https://arxiv.org/abs/2405.15793) that a carefully designed agent-computer interface — a 100-line file viewer, search capped at 50 results, an edit command that rejected syntactically invalid changes — beat a plain Linux shell decisively, reaching 12.5% on SWE-bench. The ACI was the contribution; interface design moved the number. One year later the same team released [mini-SWE-agent](https://github.com/SWE-agent/mini-swe-agent), asking "What if our agent was 100x simpler, and still worked nearly as well?" — roughly 100 lines of Python, no tools beyond bash, not even the tool-calling interface, every command a stateless `subprocess.run`, history a plain append-only list. It scores above 74% on SWE-bench Verified. The models had improved enough that, in the team's words, the simple design "puts the language model (rather than the agent scaffold) in the middle of our attention." Every 2024 guardrail encoded a 2024 weakness; by 2025 the weaknesses had shrunk below the rent the guardrails charged.

**The three-agent harness, self-pruned.** Anthropic's [long-running app-development harness](https://www.anthropic.com/engineering/harness-design-long-running-apps) offers the same arc compressed into one model transition. Under Claude Opus 4.5, the harness needed sprint decomposition — granular work chunks with per-sprint evaluation — partly to manage "context anxiety," the model's tendency to prematurely wrap up work as its window filled. When Opus 4.6 arrived, the author stripped the sprint structure and watched the generator run "coherently for over two hours without the sprint decomposition that Opus 4.5 had needed"; per-sprint evaluation collapsed into a single end-of-build check. Two load-bearing components, deleted in one generation. The post draws the general conclusion explicitly: "As models continue to improve, we can roughly expect them to be capable of working for longer, and on more complex tasks. In some cases, that will mean the scaffold surrounding the model matters less over time."

Notice, in both cases, what did *not* dissolve. mini-SWE-agent kept a loop, a transcript, seam-of-a-line sandboxability, and SWE-bench's test oracles — it deleted the interface, not the verification or the isolation. The pruned app-development harness kept its evaluator agent for tasks "at the edge of model capability," kept the sprint *contract* idea as a verification target, and kept Playwright-driven behavioral grading. The erosion is selective, and the selection is exactly the capability/trust line drawn above.

## What erodes

The perishable components share a signature: each substitutes the harness engineer's judgment for a cognitive act the model was too weak to perform. As the model strengthens, the substitution flips from prosthetic to straitjacket. The recurring species:

**Compaction hacks.** Chapter 5's aggressive summarize-at-70% pass, the carefully tuned "preserve file lists and test commands" compaction prompts, the heuristics for what to evict — all compensations for short effective context and lost-in-the-middle degradation. Longer windows, better in-context retention, and models that take their own notes (the filesystem-as-context pattern) each eat a slice of this machinery. It will not vanish — context is priced per token, so *economic* compaction survives even when *cognitive* compaction doesn't — but the elaborate hackery around what the model "can't keep track of" is among the fastest-decaying assets in any harness.

**Rigid decomposition.** Pre-chopping work into model-bite-sized pieces — sprints, phases, fixed checklists, mandatory plan formats — encodes a ceiling on coherent-work duration. It is precisely the component the Opus 4.6 transition deleted. Decomposition as a *contract between agents* (Chapter 9's handoff artifacts) survives, because contracts serve verification; decomposition as a *cognitive crutch* does not.

**Micro-designed tool interfaces.** Paginated viewers, result caps, format-enforcing wrappers around things a shell already does. This is the SWE-agent lesson: the general interface (bash, code execution) plus a capable model beats the bespoke interface plus last year's model. Chapter 4's deeper principles — errors that teach, poka-yoke against irreversible mistakes, honest documentation — outlive any particular interface, because they are about communication, not compensation.

**Prompt-level behavior management.** The "IMPORTANT: re-read the failing test before editing" incantations, the few-shot examples teaching output format, the ritual re-statements of goals every N turns. These are the first to go stale and the least likely to be cleaned up, because nobody remembers which sentence in a 400-line system prompt is load-bearing. (Chapter 12's regression suite for prompts exists precisely so someone can find out.)

**Second-guessing layers.** Retry-with-rephrasing wrappers, output-fixing parsers that repair almost-JSON, chains that ask the model to critique and revise its own draft by default. Structured-output enforcement moved into the model APIs; self-revision is increasingly wasted spend on strong models for easy tasks. What survives from this family is *external* evaluation — Chapter 10's fresh-context graders — which is not a capability patch at all, as the next section argues.

A useful heuristic for classifying a component you're unsure about: ask *who is protected if it stays, and who is limited if it stays?* If the answer to both is "the model," it is a capability compensation and it is perishable. If removing it would be fine with a perfect model but catastrophic with a *malicious* one, it is trust machinery. Keep it.

## What endures: trust problems, not capability problems

The durable core of a harness is the set of components that would remain necessary if the model were arbitrarily capable — because they answer questions that capability does not address. Three families dominate, and it is no accident that they correspond to the chapters of this book least about the model.

**Sandboxing and isolation (Chapter 7).** An agent's actions must run somewhere, and the somewhere must bound the blast radius of actions taken for bad reasons — a prompt injection in a README, a poisoned dependency, an honest misunderstanding of scope. Model improvement makes this *more* pressing, not less: a stronger model executes a hijacked objective more competently, and greater autonomy means fewer human eyes per action. That is why the trend line points toward more isolation, not less — Codex CLI reaching for kernel mechanisms (Seatbelt, Landlock/seccomp), Claude Code adding OS-level sandboxing alongside its permission prompts, agent platforms moving execution into managed sandboxes. Anthropic's [managed-agents architecture](https://www.anthropic.com/engineering/managed-agents) is instructive on what stability looks like here: the brain/hands/session decomposition and the durable session log are fixed interfaces — "We're opinionated about the shape of these interfaces, not about what runs behind them" — exactly so implementations can churn as models improve while the trust boundary stays put.

**Permissions and human authority (Chapter 8).** What an agent *may* do is a policy question, and policy is decided by institutions, not inferred by intelligence. A model that is smarter than every employee still does not get to decide whether it can push to production, spend money, or read the HR share. If anything, capability raises the stakes of the lethal trifecta — private data, untrusted content, exfiltration channel — because more capable agents are given more of all three. Permission systems, allowlists, approval policies, and the audit logs that make them enforceable in retrospect are load-bearing forever. Their *form* will change (fewer per-action prompts, more classifier-mediated and policy-as-code approvals), but the function is permanent.

**Verification and ground truth (Chapters 10 and 12).** An oracle is valuable precisely because it is *independent* of the intelligence being checked — a test suite's authority comes from running the code, not from being smarter than the author. Self-evaluation bias may shrink with model quality, but the epistemology doesn't change: a claim of success from the system that produced the work is evidence, while a green test run is ground truth, and organizations ship on ground truth. Moreover, verification is how you *detect* every other kind of erosion. The eval harness is the instrument that tells you the compaction pass stopped paying rent; give it up and you are back to vibes. The lineage of Chapter 2 is reassuring here — test harnesses did not disappear when compilers, type systems, and programmers all improved; they became more central.

Two more quiet survivors: **observability** (Chapter 11's transcript-is-ground-truth discipline — debugging, compliance, and incident response need the append-only record regardless of model quality) and **economics**. The research-system finding that [token usage explains 80% of performance variance](https://www.anthropic.com/engineering/built-multi-agent-research-system) and multi-agent configurations cost ~15× a chat session is a reminder that even in a world of very strong models, someone must decide what a task is worth and route effort accordingly. Cost engineering is a resource-allocation problem, and resource allocation does not go stale.

## The generational re-audit

If components decay on model-release boundaries, then model releases should trigger a specific, scheduled engineering activity — not a vibes-based "let's try the new model" but an audit. Anthropic's harness post states the practice: "When a new model lands, it is generally good practice to re-examine a harness, stripping away pieces that are no longer load-bearing to performance and adding new pieces to achieve greater capability that may not have been possible before." Here is that sentence expanded into a protocol.

**1. Keep an assumption ledger.** Every capability compensation gets a record at the moment it is built: the weakness it compensates, the model generation it was observed on, and the eval that demonstrated the win. This is cheap at write time and priceless at audit time — without it, nobody on the team can say why the 50-result search cap exists, and fear of the unknown preserves it forever.

```yaml
# harness/assumptions.yaml — one entry per capability compensation
- component: compaction_pass
  claim: "model output quality degrades beyond ~60% context occupancy"
  observed_on: model-gen-3
  evidence: evals/long_context_regression  # +14% solve rate when added
  class: capability            # capability | trust
- component: network_isolation
  claim: "repo contents are untrusted input; exfiltration must be impossible"
  observed_on: n/a
  evidence: threat-model §2
  class: trust                 # never strip-tested; reviewed, not audited
```

**2. Strip and measure.** For each `capability`-class entry, run the evaluation suite (Chapter 12) with the component disabled, on the new model. This is an ablation study, and your minimal baseline harness — the mini-SWE-agent lesson — is the limiting case: the whole harness ablated at once. In skeleton form:

```python
async def generation_audit(new_model: str, suite: EvalSuite) -> list[Verdict]:
    baseline = await suite.run(harness=full_harness, model=new_model)
    verdicts = []
    for comp in ledger.capability_components():
        ablated = await suite.run(
            harness=full_harness.without(comp.name), model=new_model
        )
        verdicts.append(Verdict(
            component=comp.name,
            keep=baseline.solve_rate - ablated.solve_rate > suite.noise_floor,
            delta_cost=ablated.cost - baseline.cost,
        ))
    return verdicts
```

The `noise_floor` term matters: agent evals are high-variance, and Chapter 12's warnings about reading signal from a handful of runs apply doubly when the conclusion is "delete a component someone is proud of."

**3. Strip what fails to pay, and record the deletion.** A component whose ablation costs nothing is now pure tax — tokens, latency, constraint, and maintenance surface. Delete it in a revertable commit with the eval results attached. If politics makes deletion hard, make it a config flag defaulting to off; dead-but-available is worse than gone, but better than mandatory.

**4. Re-review — don't strip-test — the trust components.** Trust machinery is audited against the threat model, not against the solve rate. The question for the sandbox at each generation is not "does the model still need this?" but "did the model's new capabilities open a hole in it?" A model that learns to use a new class of network API, or to write more convincing approval-request justifications, changes the *attack* surface, not the *capability* ledger. This review usually *adds* controls at exactly the moment the capability audit is removing them; a generational audit that only ever deletes is being done with one eye closed.

**5. Then look up, and extend.** The second half of the Anthropic sentence — "adding new pieces to achieve greater capability that may not have been possible before" — is the half teams forget. When Ledgerbot's model generation lets it hold a whole subsystem in context, the compaction pass goes — and suddenly whole-subsystem refactors, previously impossible, become a harness feature to build: new verification (behavioral tests across module boundaries), new permissions (multi-repo write access, with its own trust review), new orchestration (a fleet of Ledgerbots negotiating a lock on shared modules). The audit is not just a pruning exercise; it is how the harness's ambition tracks the model's.

Run honestly, the audit produces a harness that gets *simpler at the bottom and more ambitious at the top* with every generation — thinner cognitive prosthetics, thicker trust infrastructure, and a task envelope that keeps moving outward.

## The long-term job description

So does the harness disappear? The provocative version of the thesis says yes: models eat the scaffolding layer by layer until an agent is a model, a shell, and a prayer. The evidence reviewed here supports something more precise and more interesting. The harness does not disappear; it *bifurcates*. Everything in it that was secretly a workaround for model weakness evaporates on a two-or-three-generation half-life. Everything in it that was secretly an institution — a boundary, an oracle, a record, a budget — hardens into permanent infrastructure, the way test harnesses and CI hardened into permanent infrastructure long after the tools they originally compensated for improved beyond recognition.

That bifurcation rewrites the harness engineer's job description without eliminating the job. The parts of the role that will feel dated in five years are the ones this book spent Chapter 5 teaching you to do well anyway, because you need them *now*: hand-tuning compaction, chopping tasks to fit attention spans, wrapping tools in bumpers. The parts that compound are:

- **Environment design.** Someone must decide what world the agent inhabits: what it can touch, what it can see, what happens when it acts. This is Chapter 7's work, and demand for it scales with autonomy.
- **Verification design.** Someone must build the oracles — the tests, judges, contracts, and ground-truth signals that let anyone believe an agent's output. This is Chapter 10's work, and it is the binding constraint on how much work can be delegated at all: the frontier of agent deployment is not what models can do, it is what organizations can *check*.
- **Trust engineering.** Someone must own the permission model, the audit trail, the injection defenses, the boundaries between agents. This is Chapter 8's work, and every capability gain makes it larger.
- **The audit itself.** Someone must run the generational re-examination — the strip-and-measure loop, the ledger, the judgment about what a new model makes newly possible. This is a role that exists *because* the ground moves; it is the one job the moving ground cannot erode.

And there is the other direction, easy to miss from inside the pruning work. The same post that documented its own scaffolding dissolving also observed: ["the better the models get, the more space there is to develop harnesses that can achieve complex tasks beyond what the model can do at baseline."](https://www.anthropic.com/engineering/harness-design-long-running-apps) The multi-agent research system is the existence proof — an orchestrated harness outperforming its own very strong lead model by 90.2% on breadth-shaped work, an architecture that only became *worth building* once subagent-quality models existed. Every generation deletes last year's harness and makes next year's buildable. The harness engineer's permanent post is at that boundary: standing between what the model can do at baseline and what the task actually requires, building the difference — and then, when the model catches up, having the discipline to tear the difference down and move the post.

Sutton called his lesson bitter because researchers keep resisting it, preferring systems shaped like their own knowledge. Harness engineers get a sweeter deal, but only if they hold the deal's terms: build the scaffolding the current model needs, measure it, date it, and delete it without sentiment when its model outgrows it — while treating the sandbox, the oracle, and the audit log as what they are: not scaffolding at all, but the building.

## Further reading

- Richard S. Sutton, "The Bitter Lesson" (2019). http://www.incompleteideas.net/IncIdeas/BitterLesson.html
- Anthropic, "Harness design for long-running application development" — three-agent harness, context anxiety, and components going stale across the Opus 4.5 → 4.6 transition. https://www.anthropic.com/engineering/harness-design-long-running-apps
- Anthropic, "Building Effective Agents" — simplicity first; add complexity only when it demonstrably improves outcomes. https://www.anthropic.com/engineering/building-effective-agents
- Anthropic, "Scaling Managed Agents: decoupling the brain from the hands" — stable interfaces over changing implementations. https://www.anthropic.com/engineering/managed-agents
- Anthropic, "How we built our multi-agent research system" — token economics of orchestration; harnesses exceeding baseline model capability. https://www.anthropic.com/engineering/built-multi-agent-research-system
- Yang, Jimenez, et al., "SWE-agent: Agent-Computer Interfaces Enable Automated Software Engineering." https://arxiv.org/abs/2405.15793
- SWE-agent team, mini-SWE-agent — the 100-line control experiment. https://github.com/SWE-agent/mini-swe-agent
- OpenAI, "Agent approvals & security" — kernel-enforced sandboxing as durable trust machinery. https://developers.openai.com/codex/agent-approvals-security
- Anthropic, "Claude Code best practices" — verification as the first-class workflow primitive. https://code.claude.com/docs/en/best-practices
- Drew Breunig, "Does the Bitter Lesson Have Limits?" — on where the lesson's domain ends. https://www.dbreunig.com/2025/08/01/does-the-bitter-lesson-have-limits.html
- Simon Willison's writing on agents and prompt injection — why untrusted input keeps trust machinery permanent. https://simonwillison.net/

---

[← Case Studies](ch13-case-studies.md) · [Glossary →](glossary.md)
