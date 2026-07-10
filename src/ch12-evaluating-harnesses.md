# Chapter 12 — Evaluating the Harness

*Every chapter so far has told you to change something — a tool description, a compaction policy, a retry budget, a judge prompt. This chapter is about knowing whether any of those changes worked. Evaluating an agentic system is harder than evaluating a model: the unit under test is a loop, the output is an end state rather than a string, and the variance is enormous. We build up the measurement stack from first principles: pass@k and task-completion metrics with working code; the public benchmark harnesses (SWE-bench, Terminal-Bench, GAIA) and what they actually measure; the twin pitfalls of contamination and harness overfitting; how to A/B a harness change without fooling yourself; regression suites for prompts and tools; and cost engineering — caching, model routing, and effort tiers — with real token arithmetic at verified public prices. The through-line is the eval flywheel: every production failure becomes a test case, and the suite becomes the memory of everything that has ever gone wrong.*

## You can't improve what you don't measure

Harness engineering without evaluation degenerates into vibes-driven development: someone tweaks the system prompt, runs two tasks, feels good about it, and ships. Three weeks later a regression surfaces that the tweak caused, nobody connects the two, and another tweak papers over it. The cycle repeats because nothing in it produces knowledge.

The eval-harness tradition — the third lineage from Chapter 2, running from EleutherAI's [lm-evaluation-harness](https://github.com/EleutherAI/lm-evaluation-harness) through [SWE-bench](https://www.swebench.com/) — exists to break that cycle, and its core insight predates language models: an evaluation is an *experiment*, with all the apparatus experiments require. Anthropic's paper [Adding Error Bars to Evals](https://arxiv.org/abs/2411.00640) states the uncomfortable observation directly: evaluations are fundamentally experiments, yet the eval literature has largely ignored what every other experimental science knows about analysis and planning. Scores get reported without confidence intervals, models get compared on unpaired runs, and single-digit "improvements" get shipped that are statistically indistinguishable from noise.

For agentic systems the experimental discipline is more necessary, not less, because two properties amplify noise. First, **nondeterminism compounds**: a model that varies slightly per call varies enormously per hundred-call session, so per-task success is closer to a coin with an unknown bias than a fixed property. Second, **the harness is a confound**: the same model scores wildly differently under different scaffolding, so a benchmark number is always a claim about a *model-harness pair*, never about a model. Both properties dictate the same responses — repeated trials, paired comparisons, and metrics that treat success probabilistically. That is where pass@k comes in.

## Metrics: pass@k, pass^k, and task completion

The foundational metric for generate-and-verify tasks is **pass@k**: the probability that at least one of k independent attempts solves the task. It entered the field with OpenAI's HumanEval paper, [Evaluating Large Language Models Trained on Code](https://github.com/openai/human-eval) (Chen et al., 2021), which also contributed the estimator everyone now uses. The naive approach — run exactly k attempts, check if any passed — is an unbiased but high-variance estimate. The better approach: run n ≥ k attempts, count the c successes, and compute the probability that a random size-k subset contains at least one success:

pass@k = 1 − C(n−c, k) / C(n, k)

Computing binomial coefficients directly overflows for realistic n, so the reference implementation uses a numerically stable product form — this is the actual code from the [human-eval repository](https://github.com/openai/human-eval), reproduced because you will reimplement it otherwise, and probably wrongly:

```python
import numpy as np

def pass_at_k(n: int, c: int, k: int) -> float:
    """Unbiased estimator of pass@k.

    n: total attempts run,  c: attempts that passed,  k: budget being scored.
    """
    if n - c < k:
        return 1.0
    return 1.0 - np.prod(1.0 - k / np.arange(n - c + 1, n + 1))
```

Aggregate by averaging over tasks: run n = 10 attempts per task, score pass@1 and pass@5 from the same data. Note what pass@1 means under this estimator — c/n, the *expected* single-attempt success rate — which is a far better number than one coin flip per task.

Pass@k rewards getting lucky once, which matches some deployments (a human reviews the k candidate patches and picks one) and badly mismatches others (the agent acts autonomously; every attempt reaches production). For the autonomous case the complementary metric is **pass^k** — the probability that *all* k attempts succeed, estimated as (c/n)^k — introduced by the [τ-bench](https://arxiv.org/abs/2406.12045) agent benchmark to measure reliability rather than capability. The two metrics diverge dramatically: a harness with a 60% per-attempt success rate has pass@5 ≈ 0.99 and pass^5 ≈ 0.08. Deciding which of those two numbers is your product is a prerequisite to optimizing anything.

For a full agentic harness, the score that feeds these estimators is **task completion**, and defining it is most of the work of building an eval. The rule that generalizes: **judge the end state, not the transcript.** For Ledgerbot — this book's running example, an agent that fixes failing CI builds in a Python monorepo — completion means the previously failing test suite now passes *and* the originally passing tests still pass, checked by running them in a fresh checkout with the agent's patch applied. This is Chapter 10's oracle discipline applied to evaluation: the verifier must be independent of the agent (fresh environment, no agent-writable state — the same trust boundary that prevents reward hacking) and must check outcomes a rubric can't fake. Anthropic's guidance from building their research agent points the same direction: for stateful, multi-path tasks, evaluate whether the agent achieved the correct final state rather than validating each intermediate step, and start small — [about 20 queries representing real usage patterns](https://www.anthropic.com/engineering/multi-agent-research-system) catch most regressions long before a formal benchmark exists.

Alongside the binary outcome, record the economics of every run: wall-clock duration, steps used, tokens in and out, and dollars. The derived metric that ties the chapter together is **cost per solve** — total spend divided by tasks completed — because it is the number that moves when you change models, caching, or effort, and it is the number your CFO experiences.

## The public benchmark harnesses

Three public benchmarks dominate agent evaluation, and each is a complete *harness* in this book's sense — task definitions, execution environment, and oracle — not merely a dataset. Studying how they are built is as instructive as running them.

**SWE-bench** ([swebench.com](https://www.swebench.com/)) evaluates software agents on real GitHub issues from popular Python repositories: given the repo at the pre-fix commit and the issue text, produce a patch; the oracle applies the patch and runs the repository's own tests, including the "fail-to-pass" tests from the human fix. Because the full set proved noisy — underspecified issues, broken environments — OpenAI and the SWE-bench authors released **SWE-bench Verified**, a 500-problem subset screened by human annotators for solvability, which became the de facto standard for coding-agent claims. Its dominance is also its weakness, as the next section details.

**Terminal-Bench** ([tbench.ai](https://www.tbench.ai/)) evaluates agents on hard, realistic command-line tasks — compiling code, configuring servers, training small models, sysadmin debugging. Each task is an instruction, a Docker image, a set of verification tests, a reference solution, and a time limit; the agent drives a real shell inside the sandbox, and success is decided by pytest-style checks against the *end state* of the environment, never by reading the agent's transcript ([paper](https://arxiv.org/abs/2601.11868)). Version 2.0 is a deliberately curated hard set of 89 tasks, run through the Harbor harness, which can drive Claude Code, Codex CLI, OpenHands, mini-SWE-agent, and others against the same tasks — making it one of the few places you can compare *harnesses* holding tasks constant, exactly the comparison Chapter 13's case studies care about.

**GAIA** ([Mialon et al., 2023](https://arxiv.org/abs/2311.12983)) targets general assistants rather than coders: 466 human-curated questions (300 held out as a private test set) requiring web browsing, multi-modality, and tool use, organized into three levels — Level 1 solvable in a few steps with minimal tooling, Level 3 requiring arbitrarily long tool-using sequences. Its design philosophy inverts the benchmark arms race: instead of tasks ever harder for humans, GAIA uses tasks that are "conceptually simple for humans yet challenging for most advanced AIs" — at release, human respondents scored 92% while GPT-4 with plugins scored 15%. Answers are short, factual, and unambiguous, so scoring is exact-match — an oracle so cheap it never becomes the bottleneck.

Three design lessons recur across all three. The oracle is programmatic and end-state-based. The environment is containerized and reset per attempt — Chapter 7's isolation argument, applied to measurement. And the private-test-set decision (GAIA's 300 held-out answers) is the only structural defense against the failure mode we turn to now.

## Contamination and harness overfitting

Benchmark numbers decay through two distinct mechanisms, and diagnosing which one you are looking at matters because the remedies differ.

**Contamination** is the model having seen the test. SWE-bench Verified's tasks are drawn from public GitHub repositories, so both the issues and their human fixes are in every recent model's training data. The evidence that this inflates scores is now direct. [The SWE-Bench Illusion](https://arxiv.org/abs/2506.12286) found that state-of-the-art models can name the buggy file path from the issue text *alone* — no repository access — up to 76% of the time on SWE-bench tasks, with accuracy dropping to 53% on out-of-distribution repositories, and found substantially higher verbatim n-gram overlap with model outputs on SWE-bench than on comparable benchmarks: a memorization signature, not a reasoning one. Independent audits of the benchmark itself compound the concern: the [SWE-Bench+](https://arxiv.org/abs/2410.06992) study found that roughly a third of passing patches benefited from *solution leakage* — fix code or fragments quoted directly in the issue or its comments — and about 31% passed only because the repository's test suite was too weak to catch an incorrect patch. By 2025 OpenAI had publicly [retired SWE-bench Verified from its frontier evaluations](https://openai.com/index/why-we-no-longer-evaluate-swe-bench-verified/), citing saturation and contamination, and community efforts like [SWE-rebench](https://arxiv.org/abs/2505.20411) now continuously harvest *fresh* post-cutoff issues precisely so there is something uncontaminated left to measure.

**Harness overfitting** is subtler: the *scaffolding* having been tuned to the test. Because a benchmark score belongs to a model-harness pair, the harness is a free parameter that teams optimize — retry-until-pass loops, prompt phrasing mirroring the benchmark's issue format, tool result formats tuned on the dev split. Analyses of SWE-bench leaderboards find the same model scoring up to ~12 points apart under different harnesses ([overview of the contamination debate](https://www.codesota.com/news/swe-bench-contamination-debate)), and controlled studies such as the [Holistic Agent Leaderboard](https://arxiv.org/abs/2510.11977) show benchmark-tuned scaffolds systematically overestimating real-world capability. This is Goodhart's law with a budget line: the number goes up, and the thing the number was a proxy for does not. Nothing about it is dishonest — every improvement was real *on the benchmark* — which is exactly what makes it insidious, and exactly the trap your internal eval suite falls into the tenth time someone "fixes the harness" against the same 30 tasks.

The defenses are the same for both mechanisms, applied at different layers:

- **Temporal splits.** Evaluate on tasks created after the model's training cutoff (fresh CI failures for Ledgerbot, fresh GitHub issues for coding agents). This is the single strongest contamination control available to a practitioner.
- **A held-out set the harness never sees.** Iterate freely against the dev split; touch the held-out split only to validate a release. If dev and held-out scores diverge over time, you have measured your own overfitting — that divergence is a feature of the design, not a failure.
- **Rotate and mutate.** Perturbed variants of known tasks (renamed modules, reworded issues) cheaply separate memorization from capability; a model that solves the original but not the trivially-mutated variant has told you which one it was doing.
- **Report the pair.** State the harness and its version alongside the model in every number you publish internally. A score without its scaffolding identified is not reproducible, and Chapter 14 gives the deeper reason to keep them separable: the harness assumptions go stale at each model generation.

## A/B-ing harness changes

The everyday evaluation act is not running a leaderboard; it is answering "did my change help?" — a new tool description, a different compaction threshold, a reworded judge rubric. Getting this right is mostly about respecting variance.

The cardinal sin is the **unpaired comparison**: run suite A on Monday, suite B on Wednesday, compare means. With per-task nondeterminism as high as agentic runs exhibit, a 30-task suite's mean moves several points between *identical* configurations. The remedy, per [Adding Error Bars to Evals](https://arxiv.org/abs/2411.00640), is to analyze **paired differences**: run both configurations on the same tasks (ideally multiple trials per task), difference the per-task scores, and ask whether the mean difference is distinguishable from zero. Pairing cancels the dominant variance component — task difficulty — and typically shrinks the sample size needed to detect a given effect by a large factor. The same paper's other prescriptions transfer directly: use a power analysis to size the suite before trusting it to detect the effects you care about, and use multiple attempts per task to tame per-question sampling noise.

A paired bootstrap is fifteen lines and catches most self-deception:

```python
import random

def paired_bootstrap_ci(scores_a: list[float], scores_b: list[float],
                        iters: int = 10_000, alpha: float = 0.05):
    """95% CI for mean(B - A) over the same tasks. Lists are per-task
    success rates (c/n from repeated trials), aligned by task."""
    assert len(scores_a) == len(scores_b)
    diffs = [b - a for a, b in zip(scores_a, scores_b)]
    n = len(diffs)
    means = sorted(
        sum(random.choices(diffs, k=n)) / n for _ in range(iters)
    )
    lo = means[int(alpha / 2 * iters)]
    hi = means[int((1 - alpha / 2) * iters)]
    return sum(diffs) / n, (lo, hi)

delta, (lo, hi) = paired_bootstrap_ci(baseline, candidate)
print(f"Δ pass-rate = {delta:+.3f}, 95% CI [{lo:+.3f}, {hi:+.3f}]")
if lo <= 0 <= hi:
    print("not distinguishable from zero — do not ship a conclusion")
```

Three further disciplines keep the experiment honest. **Pin everything but the treatment**: model version, effort setting, container image, task set — a model alias silently upgrading mid-experiment is a confound you will never spot afterward (pin dated snapshots or record the served model from response metadata). **Change one thing**: bundled changes ("new prompt + new tool + new judge") produce unattributable results, and unattributable results produce cargo-cult harnesses. **Read the transcripts of the flips**: for tasks that changed outcome between arms, Chapter 11's transcript-debugging procedure tells you *why* the delta happened — which regularly reveals that a "win" is two unrelated effects, one good and one alarming, netting positive.

## Regression suites and the eval flywheel

Benchmarks measure capability; regression suites preserve it. The distinction is the same as in conventional software: benchmarks are load tests you run occasionally, the regression suite is CI you run on every change — to the system prompt, to any tool description, to the harness code itself. Prompts and tool schemas are code and deserve the same protection, because they break the same way: silently, at a distance, three commits after the cause.

The suite's growth mechanism is the **eval flywheel**: every diagnosed production failure becomes a permanent test case. The Chapter 11 transcript that revealed Ledgerbot mis-parsing a pytest failure summary yields a task — that repo state, that failing test — with an end-state oracle; the fix is validated against it; and the case stands guard forever after. Over months this produces the most valuable eval asset a team owns: a suite that is, by construction, a distribution of *your* failures rather than the internet's. The flywheel also runs on near-misses — the run that succeeded but took 400 steps is a latency regression case; the tool-selection failure caught in review (Chapter 4's tool-testing loop) becomes a single-step assertion.

Structurally, a regression suite trades breadth for speed and layers accordingly:

- **Tier 0 — static checks** (seconds, every commit): prompts render; tool schemas validate as JSON Schema; token budgets are within bounds; every tool description still contains its load-bearing sections.
- **Tier 1 — single-step behavioral checks** (a minute, every commit): given a canned context, the model selects the right tool with the right arguments; given a canned failing-test output, the model's next action is to read the failing file, not to rewrite the config. One model call per case, assertions on the tool call — cheap enough to run dozens on every prompt edit.
- **Tier 2 — end-to-end tasks** (minutes to hours, on merge and nightly): a fixed set of full sessions with end-state oracles, scored as pass@1 with n ≥ 3 trials, with cost and step-count budgets asserted alongside correctness so that economic regressions gate merges too.

The familiar test-suite pathologies all reappear and the familiar treatments all apply: flaky cases (raise n or fix the oracle), suites too slow to run (push cases down a tier), and — the agentic special — overfitting to your own regression set, which is the harness-overfitting problem in miniature and gets the same defense: a slice of Tier 2 held out from day-to-day iteration.

## Cost engineering

An eval stack that measures only correctness optimizes for a product nobody can afford. Cost per solve belongs on the same dashboard as pass rate, and it responds to engineering just as readily. The arithmetic below uses publicly listed prices as of mid-2026 — [Anthropic](https://www.anthropic.com/pricing) lists Claude Opus-tier at $5 input / $25 output per million tokens, Sonnet-tier at $3 / $15, and Haiku-tier at $1 / $5; [OpenAI](https://developers.openai.com/api/docs/pricing) lists its flagship at $5 / $30 with cached input at $0.50, mid-tier at $2.50 / $15, and small models down to $0.20 / $1.25. Prices change; the *structure* of the arithmetic doesn't.

Start with the shape of agent spend. A Ledgerbot session might run 60 steps with an average of 40,000 input tokens per step (the context grows as the transcript accumulates — Chapter 5's token math) and 500 output tokens per step. Naively, on Sonnet-tier pricing:

- Input: 60 × 40,000 = 2.4M tokens × $3/M = **$7.20**
- Output: 60 × 500 = 30,000 tokens × $15/M = **$0.45**

Two observations fall out immediately. First, **agent cost is input-dominated** — 94% of this bill is re-reading the growing transcript — which is the opposite of chat workloads and the reason the next lever dominates. Second, at $7.65 per session and a 65% solve rate, cost per solve is ~$11.77 — the number to actually optimize.

**Prompt caching** attacks the input term directly. Providers price cached prefix tokens at roughly a tenth of the base input rate — Anthropic charges ~0.1× for cache reads, with a 1.25× write premium (5-minute TTL; 2× for 1-hour); OpenAI's cached input is a flat 90% discount with no write premium. An agent loop is the ideal caching workload because each step's context is the previous step's context plus a little more: with correct breakpoint placement, nearly all of step N's input is a cache read of step N−1. Rerun the arithmetic with 90% of input tokens hitting cache at 0.1×:

- Cached: 2.16M × $0.30/M = $0.65; uncached: 0.24M × $3/M = $0.72; plus write premiums ≈ $0.15
- Input total ≈ **$1.52** versus $7.20 — a ~5× reduction on the dominant term, from a configuration change

The catch is that caching is a *prefix* mechanism, so anything that mutates the front of the context — a timestamp in the system prompt, a mid-session tool-list change — silently zeroes the benefit. Cache hit rate is therefore a first-class eval metric: assert it in Tier 2, because a prompt refactor that costs 20% accuracy would never pass review, while one that quietly quadruples spend routinely does.

**Model routing** exploits the price spread — 5× between tiers within each provider's lineup — by matching model to step difficulty. The orchestrator-workers pattern from Chapter 9 gives natural routing seams: frontier-tier for planning and diagnosis, mid-tier for the main loop, small-tier for high-volume auxiliary calls (summarizing tool output, judging duplicates, formatting reports). A worked example: if 20 of Ledgerbot's 60 steps are mechanical (log summarization, commit-message drafting) and route to a $1/$5 small model, those steps cost roughly one-third of their mid-tier price, trimming ~20% off the session — *if* the small model's error rate on those steps doesn't trigger extra recovery steps. That conditional is why routing decisions are eval decisions: the experiment is run as a paired A/B on cost per solve, not on price per token. One caution from Chapter 5's caching discussion applies: caches are model-scoped, so switching models mid-conversation forfeits the cached prefix — route at *sub-agent* boundaries, not mid-loop.

**Effort tiers** are the newest lever: both major providers now expose a knob trading reasoning depth for tokens and latency on the same model — Anthropic's `effort` parameter (`low` through `max`) and OpenAI's `reasoning_effort`. Effort behaves like a price multiplier that also moves quality, so it slots into the same experimental frame: sweep effort levels over the Tier 2 suite, plot pass rate against cost per solve, and pick per-route operating points. The recurring empirical shape is that high effort on a *planning* step reduces total session cost — better plans mean fewer wasted steps — while high effort on mechanical steps is pure waste. Spending more per token to spend fewer tokens overall is the kind of result you only find if cost per solve, not cost per call, is the metric.

Two cheaper levers round out the kit. **Batch APIs** halve prices (both providers discount asynchronous batch traffic 50%) and fit evaluation itself perfectly — nightly Tier 2 runs have no latency requirement, so the eval suite should be the first workload you move to batch. And **context discipline is cost discipline**: every Chapter 5 technique — tool-result truncation, compaction, just-in-time retrieval — reappears here with a dollar sign attached, because in an input-dominated cost model, a token saved from the context is saved sixty times.

The flywheel closes where it started. The eval suite measures the harness; the cost instrumentation measures the eval suite; and together they turn the question that opened this chapter — *did my change help?* — into something a CI job can answer while you sleep. What no CI job can answer is how long any of these carefully tuned components will stay load-bearing as models improve. That question — which parts of the harness are durable and which are scaffolding around temporary weakness — is where Chapter 14 ends the book.

## Further reading

- Chen et al. — [Evaluating Large Language Models Trained on Code](https://arxiv.org/abs/2107.03374) and the [human-eval repository](https://github.com/openai/human-eval): pass@k and its unbiased estimator.
- Yao et al. — [τ-bench](https://arxiv.org/abs/2406.12045): the pass^k reliability metric for agents.
- Miller (Anthropic) — [Adding Error Bars to Evals](https://arxiv.org/abs/2411.00640): paired differences, power analysis, and reporting uncertainty.
- [SWE-bench](https://www.swebench.com/); OpenAI — [Why we no longer evaluate SWE-bench Verified](https://openai.com/index/why-we-no-longer-evaluate-swe-bench-verified/).
- [The SWE-Bench Illusion](https://arxiv.org/abs/2506.12286): memorization evidence; [SWE-Bench+](https://arxiv.org/abs/2410.06992): solution leakage and weak tests; [SWE-rebench](https://arxiv.org/abs/2505.20411): decontaminated continuous evaluation.
- [Terminal-Bench](https://www.tbench.ai/) and the [Terminal-Bench paper](https://arxiv.org/abs/2601.11868): end-state verification in Docker sandboxes; the Harbor harness.
- Mialon et al. — [GAIA: A Benchmark for General AI Assistants](https://arxiv.org/abs/2311.12983): levels, private test set, human-vs-model gap.
- [Holistic Agent Leaderboard](https://arxiv.org/abs/2510.11977): harness effects on agent benchmark scores.
- EleutherAI — [lm-evaluation-harness](https://github.com/EleutherAI/lm-evaluation-harness): the eval-harness lineage.
- Anthropic Engineering — [How we built our multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system): end-state evaluation, LLM-as-judge, starting with ~20 queries.
- Pricing references: [Anthropic pricing](https://www.anthropic.com/pricing), [OpenAI API pricing](https://developers.openai.com/api/docs/pricing).

---

[← Reliability and Observability](ch11-reliability-observability.md) · [Case Studies →](ch13-case-studies.md)
