# Chapter 2 — Lineage: Test, Fuzz, and Eval Harnesses

*The word "harness" did not enter software vocabulary with the agent boom; it has been doing steady engineering work for roughly fifty years. This chapter traces three traditions that shaped it. Test harnesses — drivers, stubs, and the xUnit family — taught the field how to build oracles and isolate the thing under test. Fuzzing harnesses — libFuzzer targets and the OSS-Fuzz fleet — taught it that the harness is an experiment apparatus whose determinism and speed decide what can be discovered at all. Evaluation harnesses — lm-evaluation-harness and SWE-bench — taught it that the harness is part of the measurement, inseparable from the score. Each tradition solved, decades early, a problem agent builders now face daily; this chapter extracts those lessons, with a runnable xUnit-style example and a real, compilable libFuzzer target along the way.*

## A word with fifty years of history

Before software borrowed it, a harness was tackle for a draft animal: an arrangement of straps that takes a source of raw power — strong, useful, and not inclined to pull in a straight line on its own — and couples it to productive work, in a controlled direction, with the operator holding the reins. Electrical engineering borrowed the word next: a *wiring harness* is the bundled, routed, connectorized assembly that turns loose wires into a maintainable system. Both senses survive intact in the software usage. A harness channels power; a harness organizes connections.

Software's version emerged from a specific, recurring problem: how do you exercise a piece of code that was never designed to run by itself? A module deep inside a larger system has no `main` function, no user interface, and dependencies that may not exist yet. To test it in isolation you must build scaffolding around it — something to call it, something to stand in for what it calls, something to judge the result, and something to report what happened. That scaffolding, collectively, became the *test harness*, and by the time the term settled it had a reasonably crisp anatomy, which the [standard definitions](https://en.wikipedia.org/wiki/Test_harness) still reflect: a test execution engine, a repository of test cases, and the paired supporting players known as drivers and stubs.

Keep that anatomy in mind throughout this chapter, because it maps with almost embarrassing directness onto the agent harness defined in Chapter 1. Something to invoke the unit of intelligence, something to simulate or sandbox its surroundings, something to judge outcomes, something to record results: the agent loop, the execution environment, the oracle, the transcript. The field did not invent this shape in 2025. It inherited it.

## Test harnesses: drivers, stubs, and the birth of the oracle

The intellectual foundation was laid in 1979, when Glenford Myers published *The Art of Software Testing* — one of the earliest rigorous treatments of testing as an engineering discipline rather than an afterthought. Myers advocated testing modules in isolation, and named the two artifacts that make isolation possible. A **driver** is throwaway code that invokes the module under test: it constructs inputs, makes the call, and captures the result — a stand-in for the module's eventual callers. A **stub** is the mirror image: a fake, minimal implementation of something the module under test *depends on*, standing in for callees that are unavailable, slow, expensive, or nondeterministic. Drivers fake the world above; stubs fake the world below. Between them, the module under test runs in a bubble whose every input and output the tester controls. The [terminology survives essentially unchanged](https://en.wikipedia.org/wiki/Test_harness), and mocking libraries in every modern language are stubs with better ergonomics.

What Myers-era harnesses lacked was infrastructure. Drivers were bespoke; results were read by eye. The next contribution came from an unglamorous direction: the Perl interpreter's own test suite. When Larry Wall released Perl 1.0 in December 1987, its source shipped with a test harness — a script called `t/TEST` — whose test programs printed results in a line-based format of almost comical simplicity: `ok 1`, `ok 2`, `not ok 3`. The harness script consumed that stream, counted successes, and reported the total. Tim Bunce and Andreas König refined the format and built the `Test::Harness` module on top of it so that every Perl module author could reuse the same machinery, and the format was eventually formalized under the name it still carries: the [Test Anything Protocol, or TAP](https://testanything.org/history.html), by its own history [one of the oldest test protocols still in production use](https://en.wikipedia.org/wiki/Test_Anything_Protocol). The design insight was the *decoupling*: test programs and the harness that runs them agree only on a trivial text protocol, so either side can be replaced independently. Any program that prints `ok`/`not ok` lines is a test; anything that parses them is a harness. The same era produced kindred systems elsewhere — the GNU project's DejaGnu framework wrapped interactive programs in Expect scripts to the same end — but TAP's protocol-first minimalism is the ancestral pattern that recurs everywhere from CI job protocols to, as we will see, the structured transcripts of agent harnesses.

The third contribution gave the harness its modern in-process shape. In the late 1980s Kent Beck wrote [SUnit](https://en.wikipedia.org/wiki/SUnit), a testing framework for Smalltalk, first described in his paper "Simple Smalltalk Testing: With Patterns." SUnit's pattern language — the *test case* as an object, the *fixture* that establishes a known state before each test and tears it down after, *assertions* as the pass/fail judgment, and a *runner* that discovers, executes, and reports — became the template for what is now called the xUnit family. In 1997, on a flight from Zurich to the OOPSLA conference in Atlanta, Beck and Erich Gamma ported the design to Java as JUnit; the port became, per [Martin Fowler](https://martinfowler.com/bliki/Xunit.html), perhaps the most influential few hundred lines in modern software practice — his famous line is that never in the field of software development have so many owed so much to so few lines of code. Every mainstream language now has an xUnit descendant, and Python's `unittest` is a direct one.

Here is the pattern in its modern form — self-contained and runnable with `python3 -m unittest example.py`. The code under test is a fragment of Ledgerbot, this book's running example (introduced in Chapter 1): the function that classifies a CI failure log before the agent decides what to do about it.

```python
import unittest

# --- Code under test --------------------------------------------------
def classify_failure(log: str) -> str:
    """Classify a CI failure log as 'test', 'lint', 'infra', or 'unknown'."""
    if "FAILED" in log and "::" in log:
        return "test"
    if "would reformat" in log or "E501" in log:
        return "lint"
    if "Connection reset" in log or "timed out" in log:
        return "infra"
    return "unknown"

def triage(ci_client, build_id: str) -> str:
    """Fetch a build's log from CI and classify it."""
    return classify_failure(ci_client.fetch_log(build_id))

# --- The harness -------------------------------------------------------
class FakeCIClient:
    """A stub: stands in for the real CI service, so tests need no network."""
    def fetch_log(self, build_id: str) -> str:
        return "FAILED tests/test_billing.py::test_rounding - AssertionError"

class ClassifyFailureTest(unittest.TestCase):
    def setUp(self):
        # The fixture: a fresh, known state before every single test.
        self.infra_log = "requests.exceptions.ConnectionError: timed out"

    def test_pytest_failure_is_classified_as_test(self):
        log = "FAILED tests/test_billing.py::test_rounding - AssertionError"
        self.assertEqual(classify_failure(log), "test")

    def test_network_failure_is_classified_as_infra(self):
        self.assertEqual(classify_failure(self.infra_log), "infra")

    def test_empty_log_is_unknown(self):
        self.assertEqual(classify_failure(""), "unknown")

class TriageTest(unittest.TestCase):
    def test_triage_classifies_the_fetched_log(self):
        # This test method is the driver; FakeCIClient is the stub.
        self.assertEqual(triage(FakeCIClient(), "build-42"), "test")

if __name__ == "__main__":
    unittest.main()  # the runner: discovers tests, executes, reports
```

Every structural element of the fifty-year tradition is visible in thirty lines. The test methods are drivers: they invoke code that has no `main` of its own. `FakeCIClient` is a stub: the real CI service is slow, remote, and nondeterministic, so the harness replaces it with a fake whose behavior is exactly specified. `setUp` is the fixture: each test begins from a known state, so tests cannot contaminate one another and any test can be run alone, first, last, or a thousand times. The `assertEqual` calls are the **oracle** — the mechanism that tells the harness whether the action worked — reduced to its purest form: an executable statement of expectation that requires no human judgment to evaluate. And the runner is the loop: enumerate, execute, aggregate, report, exit nonzero on failure so that machines above it (CI, and one day Ledgerbot itself) can consume the verdict.

Three lessons from this tradition transfer directly to agent harnesses. First, *oracles must be explicit and executable*. A test without an assertion is not a test, and an agent step without a checkable outcome is not verifiable work — Chapter 10 builds on exactly this point. Second, *isolation is what makes results meaningful*. Fixtures exist because state leaking between tests produces verdicts you cannot trust; the agent-harness analogues are fresh sandboxes per session and fresh contexts per evaluation, for the same reason. Third, *a boring reporting protocol is a superpower*. TAP's `ok`/`not ok` is why thousands of incompatible test suites could share one harness; the structured transcript formats of Chapter 11 buy agent systems the same interchangeability.

## Fuzzing harnesses: the harness as experiment apparatus

The second tradition begins with a storm. In 1988, Barton Miller at the University of Wisconsin was dialed into a Unix system over a phone line during a thunderstorm; line noise kept injecting garbage characters into his commands, and the utilities kept crashing. The observation became a class project, then a landmark 1990 paper: feeding purely random input to standard Unix utilities crashed or hung [roughly a quarter to a third of them](https://pages.cs.wisc.edu/~bart/fuzz/). Miller called the technique *fuzz testing*, and its scandalous finding was not any single bug but the ratio: the most heavily used programs in the world fell over when confronted with input their authors had not imagined. Randomness, applied at scale, was a bug-finding instrument.

Modern fuzzing is enormously more sophisticated — coverage-guided engines mutate inputs preferentially toward those that exercise new code paths — but its central artifact is a piece of harness code so important that fuzzing practitioners simply call it "the harness" or the **fuzz target**. In [LLVM's libFuzzer](https://llvm.org/docs/LibFuzzer.html), the in-process coverage-guided engine that ships with Clang, a fuzz target is one function with a fixed signature: it accepts a byte array from the engine, feeds it to the API under test, and returns. The engine calls it millions of times, evolving the input corpus toward coverage, while sanitizers watch for the crash.

Here is a real, compilable fuzz target in C, harnessing a small date parser. Everything below the first divider is the code under test (in real use it would live in the library you link against); everything below the second divider is the harness.

```c
/* fuzz_date.c — a libFuzzer harness for a small date parser.
 *
 * Build:  clang -g -O1 -fsanitize=fuzzer,address fuzz_date.c -o fuzz_date
 * Run:    mkdir -p corpus && ./fuzz_date corpus/
 */
#include <stdint.h>
#include <stddef.h>
#include <stdlib.h>
#include <string.h>

/* --- Code under test (normally the library you link against) -------- */
struct date { int year, month, day; };

int parse_date(const char *s, struct date *out) {
    /* Accepts "YYYY-MM-DD"; returns 0 on success, -1 on bad input. */
    if (strlen(s) != 10 || s[4] != '-' || s[7] != '-')
        return -1;
    for (int i = 0; i < 10; i++) {
        if (i == 4 || i == 7) continue;
        if (s[i] < '0' || s[i] > '9') return -1;
    }
    out->year  = atoi(s);
    out->month = atoi(s + 5);
    out->day   = atoi(s + 8);
    if (out->month < 1 || out->month > 12) return -1;
    if (out->day   < 1 || out->day   > 31) return -1;
    return 0;
}

/* --- The fuzz target: the harness ----------------------------------- */
int LLVMFuzzerTestOneInput(const uint8_t *data, size_t size) {
    if (size > 64)                 /* bound the input; dates are short   */
        return -1;                 /* -1: don't add this to the corpus   */
    char *buf = malloc(size + 1);  /* the parser expects a C string, so  */
    if (!buf) return 0;            /* the harness must null-terminate —  */
    memcpy(buf, data, size);       /* adapting raw bytes to the API's    */
    buf[size] = '\0';              /* contract is the harness's job      */
    struct date d;
    parse_date(buf, &d);           /* exercise the API under test        */
    free(buf);
    return 0;                      /* 0: input accepted                  */
}
```

The signature is fixed by contract: `int LLVMFuzzerTestOneInput(const uint8_t *Data, size_t Size)` (in C++ it must be declared `extern "C"`; in C, as here, no annotation is needed). Compiling with `-fsanitize=fuzzer,address` links the fuzzing engine — which supplies `main` — and AddressSanitizer. Run the binary with a corpus directory and it will execute the target thousands of times per second, mutating inputs, saving any input that reaches new coverage. If `parse_date` ever reads or writes out of bounds, AddressSanitizer aborts the process with a full report, and libFuzzer writes the offending input to a `crash-<hash>` file. Re-running the binary with that file as its argument replays the crash deterministically. The output of the whole apparatus is not prose; it is a file of bytes that reproduces the failure on demand.

The [libFuzzer documentation](https://llvm.org/docs/LibFuzzer.html) is strict about what this function must be, and the discipline it imposes is the tradition's core teaching. The target "must be as deterministic as possible": the same input must produce the same behavior, or the engine's coverage feedback becomes noise and reported crashes stop reproducing. It "must not `exit()` on any input" and must "tolerate any kind of input (empty, huge, malformed)": the harness is invoked millions of times in one process and must survive all of them. It should not modify global state, because state leaking between executions makes runs order-dependent — the fuzzing version of the test-fixture rule. And it "must be fast": Google's guide to [writing good fuzz targets](https://github.com/google/fuzzing/blob/master/docs/good-fuzz-target.md) sets the bar at roughly 1,000 executions per second per core and memory consumption under about 1.5 GB — because throughput is experiment count, and experiment count is what discovers bugs. The same guide teaches corpus craft: seed with small, representative inputs covering as much of the API as possible, and reject pathological sizes in the target itself, as the size check above does.

Notice what the fuzzing tradition adds beyond the testing tradition. A unit test encodes *one* hypothesis with a hand-written oracle. A fuzz target encodes a *universal* oracle — "this code never corrupts memory, never trips an assertion, never crashes, on any input whatsoever" — and delegates hypothesis generation to the machine. That universal oracle is made checkable by *instrumenting the environment*: sanitizers turn silent memory corruption, which might otherwise go unnoticed for a million executions, into an immediate, loud, attributable failure. And the whole apparatus runs unattended at industrial scale: Google's [OSS-Fuzz](https://google.github.io/oss-fuzz/), launched in 2016 in the aftermath of the Heartbleed vulnerability (itself the kind of bug a well-harnessed fuzzer finds quickly), runs fuzz targets continuously for open-source projects; as of August 2023 it had helped find and fix over 10,000 vulnerabilities and 36,000 bugs across 1,000 projects.

The lessons for agent harnesses are one-to-one, and Chapter 1's Ledgerbot makes them concrete. *Determinism and reproducibility*: a crash file that replays is the fuzzing tradition's transcript — Ledgerbot's harness likewise records every session such that a failure can be replayed and studied, not shrugged at (Chapter 11's transcript-is-ground-truth discipline). *The harness bounds what can be discovered*: a fuzz target that only ever null-terminates and calls one function will never find bugs in the functions it doesn't call, no matter how clever the engine — exactly as an agent whose tool layer cannot express an action will never take it, however capable the model (Chapter 4). *Make silent failures loud*: sanitizers are the model for lint gates, type checkers, and schema validation on tool results — environmental instrumentation that converts "subtly wrong" into "immediately visible" (Chapter 10). *Throughput is discovery*: fuzzing's exec/s obsession reappears as the cost-and-latency engineering of Chapter 12 — an evaluation you can afford to run a hundred times teaches you more than one you can afford once. And the *corpus* — the accumulated set of interesting inputs, carried between runs — is memory across sessions, Chapter 6's concern, in its oldest working form.

## Eval harnesses: measuring the unmeasurable

The third tradition is the youngest, born when language models themselves became the thing under test — and it is where the word "harness" entered the LLM world years before agents did.

When large language models began multiplying around 2020, every lab evaluated them with its own scripts, its own prompt phrasings, its own answer-extraction heuristics — and the numbers were quietly incomparable. A model's score on a benchmark could shift by points based on how the harness formatted few-shot examples or parsed the model's output, and papers rarely published those details. EleutherAI's response was the [lm-evaluation-harness](https://github.com/EleutherAI/lm-evaluation-harness): "a unified framework to test generative language models on a large number of different evaluation tasks," now spanning over 60 standard academic benchmarks with hundreds of subtasks and variants. Its stated design principle goes to the heart of the problem: evaluation with *publicly available prompts* ensures reproducibility and comparability between papers. Tasks are defined declaratively — YAML configurations with templated prompts, few-shot policy, and answer-extraction rules — so an evaluation is a versioned artifact, not a folk practice. The project became infrastructure: it served as the backend of Hugging Face's Open LLM Leaderboard and has been used across hundreds of papers and dozens of organizations, which means a leaderboard number is, precisely, *the model's score under this harness, at this commit* — and everyone can run the same one.

Then agents made evaluation harder, because an agent's output is not a string to compare against a reference; it is a *behavior* in an environment. The benchmark that defined this era is [SWE-bench](https://www.swebench.com/) (Jimenez et al., ICLR 2024): [2,294 task instances](https://arxiv.org/abs/2310.06770) drawn from real GitHub issues and their resolving pull requests across 12 popular Python repositories. The system under test receives a repository snapshot and an issue description and must produce a patch. And here the harness *is* the oracle, in a design worth studying closely. For each task, the benchmark carries two sets of tests: **FAIL_TO_PASS** — tests that fail before the reference fix and pass after it, encoding "you actually fixed the reported problem" — and **PASS_TO_PASS** — tests that pass both before and after, encoding "and you broke nothing else." The evaluation harness applies the candidate patch to the pristine repository, runs both sets, and issues a verdict no human needs to adjudicate. It is Myers's driver-and-oracle pattern operating at the scale of an entire repository, and it is exactly the oracle Ledgerbot's own verification step implements for every fix it proposes: *the failing thing now passes, and everything that passed still does.*

SWE-bench's operational history then re-taught, in public and in fast-forward, every lesson of the older traditions. Early on, results varied with the machine that ran the evaluation — environmental nondeterminism, the fuzzing tradition's cardinal sin — so in June 2024 the maintainers shipped a fully containerized evaluation harness using Docker, giving each task a reproducible environment: isolation, rediscovered. Scrutiny of the tasks themselves revealed that a meaningful fraction were underspecified or had unreliable tests — the oracle itself was buggy — leading to [SWE-bench Verified](https://www.swebench.com/), a 500-instance subset in which human engineers confirmed each task is fairly solvable and its tests actually check the fix: oracle validation, rediscovered. And the community learned to read leaderboards with a caveat the eval tradition makes precise: a submission's score reflects the model *and* the scaffold it ran in — the tools, the retries, the prompts, the loop. Two systems using the identical model routinely post different numbers. There is no such thing as evaluating a model on an agentic task; there is only evaluating an *agent* — a model plus a harness — which is Chapter 1's equation restated as a measurement problem, and the reason Chapter 12 insists that when you A/B a harness change you hold the model constant, and vice versa.

This tradition's distinctive lesson, then, is *measurement validity*: the harness is not neutral plumbing around an experiment — it is part of the experimental design, and unshared harness details are unshared science. Its corollaries — pin the environment, version the harness, validate the oracle before trusting what it tells you, and beware benchmarks the system may have memorized — are the working content of Chapter 12.

## What the traditions teach

Fifty years of harness-building compress into four principles, each inherited by agent harness engineering with its serial numbers barely filed off.

**Oracles are designed artifacts.** Every tradition lives or dies by its answer to "how do we know it worked?" — the xUnit assertion, the sanitizer-instrumented crash, the FAIL_TO_PASS/PASS_TO_PASS pair. None of these oracles occurs naturally; each was engineered, and the hardest engineering in each tradition went into making the oracle cheap, objective, and machine-checkable. Agent harnesses inherit the full problem: an agent that cannot see whether its action worked is a unit test without an assertion. Chapter 10 treats oracle design as the highest-leverage work in the discipline, including what to do when no perfect oracle exists and an LLM judge must approximate one — a move the eval tradition's rubric-based scoring anticipated.

**Isolation makes verdicts trustworthy.** Fixtures reset state between tests; fuzz targets forswear global state; SWE-bench moved into containers. The shared insight is that a verdict is only as good as the guarantee that nothing else could have produced it. For agents this becomes: fresh sandboxes per session (Chapter 7), and — because an agent under evaluation has agency inside the apparatus in a way a date parser does not — verification in contexts the agent being verified cannot reach or influence. That separation, of generator from grader across a trust boundary, is Chapter 10's answer to reward hacking, and it is the fixture pattern with adversarial stakes.

**Reproducibility is the debugging substrate.** The crash file that replays, the pinned corpus, the versioned YAML task, the Docker image: each tradition converged on the same rule — an unreproducible failure is an anecdote. The agent-harness form of the rule is the transcript (Chapter 11): every session recorded completely enough that a surprising run can be replayed, diffed, and understood. Nondeterminism in the model makes bit-exact replay impossible, which raises rather than lowers the standard for determinism everywhere else in the apparatus.

**The harness is the experiment apparatus.** This is the deepest inheritance and the mindset shift this chapter exists to deliver. In every tradition, practitioners eventually stopped seeing the harness as scaffolding around the interesting part and recognized it as the instrument that determines what can be observed at all — the fuzz target bounds discoverable bugs; the eval harness is inseparable from the score. Agent harness engineering begins from the same recognition: the model is the powerful, somewhat unruly source of capability, and the harness is the apparatus that decides what that capability can touch, what it can perceive, and what counts as success. Ledgerbot's monorepo, tools, sandbox, and test oracle are not accessories to the model; they are the experiment within which the model's intelligence becomes legible, checkable work.

There is one genuine discontinuity, and it is fair to name it. In every prior tradition, the thing inside the harness was *passive* — a parser does not read the fuzzer's documentation, and a module under test does not try to convince the driver it passed. An agent does both: it perceives its harness, adapts to it, and will happily satisfy a badly designed oracle by the letter rather than the spirit. The old traditions supply the load-bearing concepts; the new discipline must apply them to an occupant that reads the blueprints. How to build the loop that occupant runs in is Chapter 3.

## Further reading

- Glenford J. Myers, *The Art of Software Testing* (Wiley, 1979) — the foundational treatment of unit testing with drivers and stubs; see also the [test harness overview](https://en.wikipedia.org/wiki/Test_harness).
- [TAP History](https://testanything.org/history.html) and the [Test Anything Protocol](https://en.wikipedia.org/wiki/Test_Anything_Protocol) — Perl's 1987 `t/TEST` harness and the protocol it left behind.
- [SUnit](https://en.wikipedia.org/wiki/SUnit) and Martin Fowler, [bliki: Xunit](https://martinfowler.com/bliki/Xunit.html) — Kent Beck's Smalltalk framework and the JUnit port that spread the pattern everywhere.
- Barton Miller et al., [the Fuzz Testing project at UW–Madison](https://pages.cs.wisc.edu/~bart/fuzz/) — the 1990 study that founded fuzzing.
- [libFuzzer documentation](https://llvm.org/docs/LibFuzzer.html) — the fuzz-target contract: signature, determinism, speed, input tolerance.
- Google, ["What makes a good fuzz target"](https://github.com/google/fuzzing/blob/master/docs/good-fuzz-target.md) — exec/s and memory budgets, seed corpora, input-size discipline.
- [OSS-Fuzz](https://google.github.io/oss-fuzz/) — continuous fuzzing as a service; scale and impact figures.
- [EleutherAI lm-evaluation-harness](https://github.com/EleutherAI/lm-evaluation-harness) — standardized, reproducible LLM evaluation; backend of the Open LLM Leaderboard.
- Carlos E. Jimenez et al., ["SWE-bench: Can Language Models Resolve Real-World GitHub Issues?"](https://arxiv.org/abs/2310.06770) (ICLR 2024) and [swebench.com](https://www.swebench.com/) — the benchmark, its FAIL_TO_PASS/PASS_TO_PASS oracle, the containerized harness, and SWE-bench Verified.

---

[← What Is Harness Engineering?](ch01-what-is-harness-engineering.md) · [The Agent Loop →](ch03-the-agent-loop.md)
