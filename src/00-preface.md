# Preface

In 2023 the industry believed the interesting engineering problem was the prompt.
In 2024 it decided the problem was the context. By 2026 a quieter consensus had
formed among the people actually shipping agents: the problem is *everything else*
— the loop that drives the model, the tools it is handed, the sandbox its actions
run in, the checkpoints that survive a crash, the evaluator that refuses to take
its word for it, and the logs you read at 2 a.m. when a six-hour run went sideways
at hour four. The industry borrowed an old word for that everything-else: the
**harness**. The discipline of designing it well is **harness engineering**.

The word is borrowed deliberately. Test engineers have built harnesses — drivers,
stubs, oracles, fixtures — for half a century. Fuzzing engineers write harnesses
that feed hostile bytes into parsers millions of times per second and know that a
bad harness silently finds nothing. Evaluation researchers wrap models in
harnesses so that a benchmark number means the same thing twice. Every one of
those traditions learned the same lesson the agent builders are now relearning:
**the apparatus around the system under test determines what the system can
actually do, and whether you can trust what it did.** This book takes that lineage
seriously, because it contains most of the answers.

## What this book argues

Three claims run through every chapter.

**First: an agent is a model plus a harness, and the harness is the part you
control.** You cannot retrain the model this afternoon. You can, this afternoon,
fix the tool whose error message taught the model nothing, add the verification
step that would have caught the regression, or move the one instruction that was
being ignored from the middle of the context to a place the model attends to.
Harness engineering is the highest-leverage work available to most teams building
on frontier models — precisely because it is ordinary software engineering, done
on an extraordinary component.

**Second: agent failures are environmental problems to be fixed permanently, not
prompts to be retried.** When an agent fails, the tempting response is to rerun it
and hope. The engineering response is to ask what the environment allowed to go
wrong — a tool that accepted an ambiguous argument, a context that had silently
lost the plan, a grader that believed the agent's self-assessment — and change the
environment so that the failure class, not the failure instance, is gone. This is
the mindset shift that separates prompt tinkering from harness engineering.

**Third: every component of a harness encodes an assumption about what the model
cannot do — and those assumptions expire.** Harnesses built for one model
generation calcify into overhead for the next. The final chapter argues that the
durable parts of this discipline are the ones that address *trust* (sandboxing,
permissions, verification) rather than *capability* (decomposition scaffolds,
context workarounds), and that a good harness engineer spends as much time
removing machinery as adding it.

## Who this book is for

Software engineers and AI engineers who build, operate, or evaluate agentic
systems — or are about to. It assumes fluency in Python, HTTP APIs, containers,
and CI, and assumes nothing about prior agent-building experience. Researchers
will find the evaluation and verification chapters useful; engineering leaders
will find the case studies and the final chapter a map of where this field is
going and how much of today's stack to bet on.

## How it was written

This book synthesizes the public record of the field as of mid-2026: vendor
engineering blogs, open-source harness codebases, benchmark documentation,
academic papers, and practitioner writing. Sources are cited inline where claims
are made and collected at the end of each chapter. Where the field disagrees with
itself — and it often does — the disagreement is presented rather than smoothed
over. All code examples target publicly documented APIs.

A note on shelf life: harness engineering moves fast. The *principles* here —
oracles, isolation, least privilege, feedback loops, measurement — are stable,
because they were stable for decades before language models existed. The *product
details* (specific CLIs, specific pricing, specific benchmark numbers) are
snapshots, and are dated in the text so you can tell which is which.

## Acknowledgments

This book stands on the public writing of the teams at Anthropic, OpenAI,
Princeton NLP, EleutherAI, All Hands AI, Google's OSS-Fuzz project, and the many
individual practitioners — Simon Willison prominent among them — who documented
what actually happens when you let a language model touch a computer. The failures
described in these pages are composites; anyone who has run an agent for more than
an hour will recognize them anyway.

---

[What Is Harness Engineering? →](ch01-what-is-harness-engineering.md)
