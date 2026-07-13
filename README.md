# Harness Engineering

### Building the Machinery Around the Model

*How to turn a raw language model into a dependable agent — loops, tools, context,
sandboxes, verification, and orchestration.*

---

An agent is a model plus a harness. The model gets the headlines; the harness is
where most production failures — and most production wins — actually live. This
book treats harness engineering as a real engineering discipline: one with a
fifty-year lineage in test, fuzz, and evaluation harnesses, and a fast-moving
present in agentic AI systems.

**Read it here on GitHub** (every chapter link below), or **as a website**: the
repo is an [mdBook](https://rust-lang.github.io/mdBook/) project with a GitHub
Actions workflow that publishes to GitHub Pages on every push to `main`
(enable it under *Settings → Pages → Source: GitHub Actions*).

```bash
# or read locally with search, sidebar, and dark mode:
cargo install mdbook && mdbook serve --open
```

## Contents

**Front matter**
— [Preface](src/00-preface.md)

**Part I — Foundations**
1. [What Is Harness Engineering?](src/ch01-what-is-harness-engineering.md)
2. [Lineage: Test, Fuzz, and Eval Harnesses](src/ch02-lineage.md)
3. [The Agent Loop](src/ch03-the-agent-loop.md)

**Part II — The Interfaces**
4. [Tools and the Agent-Computer Interface](src/ch04-tools-and-the-aci.md)
5. [Context Engineering](src/ch05-context-engineering.md)
6. [Memory, State, and Sessions](src/ch06-memory-state-sessions.md)

**Part III — The Boundaries**
7. [Execution Environments and Sandboxing](src/ch07-execution-environments.md)
8. [Safety, Security, and Permissions](src/ch08-safety-security-permissions.md)

**Part IV — Scale and Trust**
9. [Orchestration and Multi-Agent Patterns](src/ch09-orchestration.md)
10. [Verification and Feedback Loops](src/ch10-verification-feedback.md)
11. [Reliability and Observability](src/ch11-reliability-observability.md)
12. [Evaluating the Harness](src/ch12-evaluating-harnesses.md)

**Part V — Practice and Prognosis**
13. [Case Studies](src/ch13-case-studies.md)
14. [The Disappearing Harness](src/ch14-the-future.md)

**Back matter**
— [Glossary](src/glossary.md)

## How to read this book

**Part I** establishes what a harness is, where the idea comes from, and the agent
loop at the center of every harness — Chapter 3 builds a complete working harness
from scratch, and everything after refines a piece of it. **Part II** covers the
three surfaces through which a model touches the world: tools (what it can do),
context (what it can see), and memory (what it can keep). **Part III** covers
where agent actions run and what they are allowed to touch. **Part IV** covers
composing agents into larger systems, verifying their work, keeping long runs
alive, and measuring whether any of it works. **Part V** tears down real public
harnesses and argues about which parts of the discipline will outlive the next
model generation.

If you build agents for a living, read Part I and then jump to whatever is
currently failing in production. It will be covered.

## The running example

Worked examples throughout use **Ledgerbot**, a fictional agent that triages and
fixes failing CI builds in a mid-size Python monorepo: it reads logs, edits code,
runs tests, and opens pull requests. It is deliberately mundane — mundane tasks
are where harness engineering earns its keep.

## Code and sources

All Python examples target Python 3.11+. Where a concrete model API is required,
examples use publicly documented SDK shapes and are cited to vendor docs. Sources
are linked inline at the point of claim and collected per chapter under *Further
reading*. Product details and pricing are snapshots as of mid-2026 and dated in
the text; the principles are meant to outlast them.

## License

Text and figures © 2026 Huzaifa Sidhpurwala, licensed under
[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) (see [LICENSE](LICENSE)).
Code samples are licensed under the [MIT License](LICENSE-CODE) — use them in
your own projects freely, attribution appreciated but not required.
