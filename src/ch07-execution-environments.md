# Chapter 7 — Execution Environments and Sandboxing

*Every action an agent takes has to run somewhere, and the choice of where is one of the most consequential decisions a harness engineer makes. This chapter maps the isolation spectrum — from raw local execution through OS-level sandboxes, containers, microVMs, and remote sandbox services — and shows how to configure each defensibly. We build a hardened container specification line by line, examine why Firecracker and gVisor exist and what they cost, look at snapshot/restore as a harness primitive, and work through dev-container patterns for team use. The organizing principle throughout: isolation is the safety model. Permission systems, which Chapter 8 covers, are a policy layer stacked on top of it; they decide what the agent should do, while the execution environment bounds what the agent can do when policy fails.*

## The environment is part of the harness

A model produces text. The harness turns that text into consequences: files written, processes spawned, packets sent. Everything up to the moment of execution — context assembly, tool schemas, the agent loop — is reversible. Execution is where the harness stops simulating and starts acting, and it is therefore where a harness engineer's mistakes stop being bugs and start being incidents.

Consider Ledgerbot, this book's running example: an agent that triages and fixes failing CI builds in a mid-size Python monorepo. To do its job, Ledgerbot must read build logs, edit source files, install dependencies, and run the test suite. Every one of those capabilities is also an attack capability. `pip install` executes arbitrary code from `setup.py` hooks. Running the test suite executes whatever the repository contains — including code Ledgerbot itself just wrote, possibly under the influence of a malicious string it read from a CI log. An agent that can run `pytest` can run anything.

This is the central asymmetry of agent execution: you cannot enumerate the safe commands in advance, because the agent's usefulness comes precisely from its ability to compose commands you didn't anticipate. Two failure sources make this concrete. The first is model error — the agent misunderstands, hallucinates a path, runs `rm -rf` against the wrong directory, force-pushes over a branch. The second is adversarial steering: prompt injection, where untrusted content the agent reads (a log line, an issue comment, a web page) carries instructions the model follows as if they came from its operator. Chapter 8 treats injection in depth; what matters here is the design conclusion both failure sources point to. You cannot make the model reliably safe, so you make the environment safe to be unreliable in. The question shifts from "will the agent do something bad?" to "what is the blast radius when it does?"

That question has a well-established engineering answer — least privilege, applied to compute. The agent's execution environment should grant exactly the filesystem, network, and resource access the task requires, and nothing else. The rest of this chapter is about the mechanisms available for enforcing that, and their trade-offs.

## The isolation spectrum

Execution environments for agents fall along a spectrum of isolation strength, cost, and fidelity. From weakest to strongest:

1. **Local execution.** The agent's commands run as your user, on your machine, with your credentials and your home directory. Zero setup cost, full fidelity — and no boundary at all. Everything you can do, the agent can do, including reading `~/.ssh` and `~/.aws/credentials`.
2. **OS-level sandboxing.** The agent still runs on your machine, but kernel primitives restrict which paths and network destinations its processes can touch. Cheap, fast, no image builds — but the boundary is a policy on a shared kernel, not a separate system.
3. **Containers.** Processes run in their own namespaces with their own filesystem image, resource limits, and (optionally) no network. This is the workhorse tier for agent harnesses: strong-enough isolation, mature tooling, reproducible images.
4. **MicroVMs and user-space kernels.** Firecracker gives each workload its own guest kernel under hardware virtualization; gVisor interposes a user-space kernel between the workload and the host. Both exist because containers share the host kernel, and the kernel is a large attack surface.
5. **Remote sandbox services.** The environment isn't on your machine at all — a provider (or your own fleet) runs it, and your harness talks to it over an API. Isolation becomes someone else's operational problem, and scale becomes an API call.

No tier is "correct." Ledgerbot running against your own monorepo on your own laptop, with you watching, is a different threat model than Ledgerbot running unattended overnight against fifty repositories, which is different again from a multi-tenant service running agent code on behalf of strangers. The spectrum exists so you can match blast radius to trust.

### OS-level sandboxing: cheap and shallow

The lightest mechanism worth using restricts an agent's processes without virtualizing anything. Anthropic's [sandboxing for Claude Code](https://www.anthropic.com/engineering/claude-code-sandboxing) is a good public example of the pattern: filesystem isolation via [bubblewrap](https://github.com/containers/bubblewrap) on Linux and Seatbelt (`sandbox-exec`) on macOS confines the agent's shell commands to declared directories, while network isolation routes traffic through a proxy running *outside* the sandbox that enforces a domain allowlist and prompts the user for new destinations. Anthropic released the underlying mechanism as an open-source research preview, [sandbox-runtime](https://github.com/anthropic-experimental/sandbox-runtime), pitched at teams that want directory- and domain-scoped enforcement "without the overhead of spinning up and managing a container."

Two design points in that architecture generalize. First, the enforcement point for network policy lives outside the sandbox — a proxy the sandboxed process cannot modify. Any security control that lives inside the boundary it polices is a control the agent (or an injected instruction) can eventually turn off. Second, sandboxing and permission prompts trade off against each other: with hard OS-level boundaries in place, the harness can stop asking the human about every command inside those boundaries. Anthropic reports that sandboxing "safely reduces permission prompts by 84%" in internal testing — a number worth remembering when Chapter 8 discusses approval fatigue, because it quantifies how much of the human-in-the-loop burden exists only to compensate for a missing boundary.

The limits are equally instructive. OS-level sandboxes share the host kernel and see the real machine; a kernel exploit, a policy gap, or a mis-scoped rule exposes the host directly. They isolate *access*, not *environment*: the agent still runs against your actual filesystem state, so a run is not reproducible and a mess is not disposable. For anything long-running, unattended, or adversarial-facing, you want at least a container.

## Containers: the workhorse tier

Containers hit the sweet spot for most agent harnesses: near-instant startup, a reproducible filesystem image, kernel-enforced resource limits, and one-flag network removal. The evaluation harnesses for [SWE-bench](https://www.swebench.com/) run each task instance in its own Docker container for exactly these reasons — isolation plus reproducibility — and coding-agent platforms like [OpenHands](https://github.com/OpenHands/OpenHands) execute agent actions in a sandboxed Docker runtime by default.

But a default `docker run` is not a sandbox. Out of the box the container has full outbound network access, unbounded memory and process counts, and a root user inside the container. Hardening it is a matter of a dozen deliberate flags. Here is a specification suitable for an agent like Ledgerbot running its build-and-test loop, followed by what each line buys you.

```dockerfile
# Dockerfile — agent execution image
FROM python:3.12-slim

# Toolchain the agent needs, installed at build time (the container
# will have no network at runtime, so everything must be baked in).
RUN apt-get update && apt-get install -y --no-install-recommends \
        git build-essential \
    && rm -rf /var/lib/apt/lists/*

# Non-root user. Everything the agent does runs as this identity.
RUN useradd --create-home --uid 1000 agent
WORKDIR /workspace
USER agent
```

```bash
docker run --rm \
  --name ledgerbot-run-042 \
  --network none \
  --memory 2g --memory-swap 2g \
  --cpus 2 \
  --pids-limit 256 \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=256m \
  --cap-drop ALL \
  --security-opt no-new-privileges \
  --user 1000:1000 \
  -v "$PWD/checkout:/workspace:rw" \
  -v "$PWD/deps-cache:/home/agent/.cache/pip:ro" \
  ledgerbot-exec:latest \
  sleep infinity
```

The harness then drives the container with `docker exec` (or the Docker SDK), one command per agent step, and destroys it when the session ends. Walking through the flags:

- **`--network none`** removes the network namespace entirely. No DNS, no sockets, no exfiltration channel. This single flag eliminates one leg of the "lethal trifecta" Chapter 8 describes — an agent that cannot transmit data cannot leak it, no matter what an injected instruction asks for. It also forces discipline at image-build time: dependencies must be baked into the image or mounted as a cache, which incidentally makes runs reproducible.
- **`--memory 2g --memory-swap 2g`** caps RAM, and setting swap equal to memory disables swap so the limit is real. Agents genuinely hit this: a test that loads a corpus into memory, a runaway build. Without a cap, the agent's mistake becomes the host's out-of-memory event.
- **`--cpus 2`** and **`--pids-limit 256`** bound compute and process count. The pids limit is the one people forget; it is what turns a fork bomb — which agents can and do produce by accident, for instance a test spawning subprocesses in a retry loop — from a host outage into a contained `fork: retry` error.
- **`--read-only`** mounts the container's root filesystem read-only, so the agent can modify only the volumes you explicitly grant (`/workspace`) and the tmpfs scratch space. The agent cannot trojan the image's own tools — no editing `/usr/bin/git` to persist across steps.
- **`--tmpfs /tmp:rw,noexec,nosuid,size=256m`** gives a bounded scratch directory. `noexec` means a payload dropped in `/tmp` cannot be executed from there; it is a speed bump rather than a wall (an interpreter can still `python /tmp/x.py`), but layered defenses are the point.
- **`--cap-drop ALL`** removes every Linux capability. The default Docker set includes capabilities like `NET_RAW` and `MKNOD` that an agent build-and-test loop never needs. Drop everything; add back individually if something breaks.
- **`--security-opt no-new-privileges`** blocks privilege escalation via setuid binaries — even if one exists in the image, `execve` cannot gain privileges through it.
- **`--user 1000:1000`** runs as the unprivileged user even if the image's `USER` directive is somehow wrong. Root inside a container is not root on the host, but it is much closer than you want: combined with a kernel bug or a mis-mounted socket, it is the standard first step in container escapes.
- **The volume lines** are the least-privilege statement for data: the repository checkout is the only writable mount, and it is a *copy* the harness made for this run — never the canonical working tree. The dependency cache is mounted read-only.

One deliberate omission deserves a note: there is no `-v /var/run/docker.sock` line, and there never should be. Mounting the Docker socket into an agent's container hands it control of the Docker daemon — equivalent to root on the host. If the agent's task requires building or running containers, use a nested or remote builder, or move up an isolation tier.

Docker's own [security documentation](https://docs.docker.com/engine/security/) and the [runtime options reference](https://docs.docker.com/engine/containers/run/) cover each of these controls; the composition above is just least privilege applied systematically. In harness code, the pattern looks like this fragment:

```python
import subprocess

def exec_in_sandbox(container: str, command: str, timeout: int = 300) -> tuple[int, str]:
    """Run one agent-issued command inside the sandbox container."""
    proc = subprocess.run(
        ["docker", "exec", "--user", "1000:1000", container,
         "bash", "-lc", command],
        capture_output=True, text=True, timeout=timeout,
    )
    output = (proc.stdout + proc.stderr)[-20_000:]  # cap tool-result size
    return proc.returncode, output
```

The timeout and the output cap are harness concerns as much as safety concerns: Chapter 4 discusses token budgets for tool results, and Chapter 11 covers timeout design. The sandbox is where those policies get enforced.

### What containers do not give you

Every container on a host shares that host's kernel. The isolation boundary is kernel code — namespaces, cgroups, seccomp filters — and the kernel exposes hundreds of syscalls of attack surface to the workload. For an agent running code you wrote and reviewed, that is fine. For an agent running code that untrusted parties can influence — and an agent that reads the internet, processes user uploads, or serves multiple tenants is exactly that — a kernel privilege-escalation bug is a container escape. This is not hypothetical; it is why the next tier exists, and why every serious multi-tenant code-execution service runs on something stronger than namespaces.

## MicroVMs and user-space kernels

Two architectures address the shared-kernel problem from opposite directions.

**Firecracker** gives each workload its own kernel. It is a virtual machine monitor, open-sourced by AWS, that runs "microVMs" — stripped-down virtual machines with a minimal device model (no PCI, no BIOS, a handful of virtio devices) — on top of KVM. Because there is almost no virtual hardware to initialize, a microVM boots to a running guest kernel in as little as 125 milliseconds, with roughly 5 MiB of VMM overhead per instance, per the [project's documentation](https://github.com/firecracker-microvm/firecracker). Firecracker was built to run AWS Lambda and Fargate — millions of short-lived, mutually distrusting workloads packed densely onto shared hardware — which is almost exactly the agent-sandbox problem statement, and it is the foundation most agent-sandbox providers build on ([Fly.io's explainer](https://fly.io/learn/firecracker-vm/) is a good introduction). The security claim is structural: two microVMs share no kernel code paths, so a guest-kernel vulnerability in one cannot propagate to another; an escape requires breaking the hardware virtualization boundary itself. Firecracker also ships a `jailer` that wraps the VMM process in its own cgroup, chroot, and seccomp policy — defense in depth for the component that touches KVM.

**gVisor** takes the other route: keep containers, but stop letting them talk to the host kernel. Its core component, the Sentry, is an application kernel written in Go that implements the Linux syscall interface itself — syscalls, memory management, signal delivery, a network stack — and intercepts every syscall the sandboxed application makes, per the [gVisor architecture docs](https://gvisor.dev/docs/). The application's syscalls never reach the host kernel directly; the Sentry services them, itself running under a tight seccomp filter, with filesystem access mediated by a separate Gofer process. The host kernel's exposed surface shrinks from "all of Linux" to the small set of syscalls the Sentry needs, and the kernel-reimplementation is in a memory-safe language. Google runs it as [GKE Sandbox](https://cloud.google.com/kubernetes-engine/docs/concepts/sandbox-pods); operationally it is a drop-in container runtime (`runsc`), which makes it the lowest-friction way to strengthen an existing Docker-based harness.

The trade-offs are symmetrical. Firecracker costs you container ergonomics — you manage kernels, rootfs images, and a VM lifecycle instead of `docker run` — and requires bare-metal or nested virtualization. gVisor keeps the container workflow but taxes every syscall (interception plus reimplementation), so syscall-heavy workloads — which build-and-test loops are — pay a real performance penalty, and compatibility is "most of Linux" rather than all of it. The practical guidance: if your harness already speaks Docker and you need stronger isolation tomorrow, gVisor. If you are building a sandbox *service* — many concurrent, untrusted, snapshot-heavy workloads — Firecracker.

## Remote sandbox services

At some point the right answer is to stop running agent code on your own machine entirely. A remote sandbox service exposes ephemeral execution environments over an API: create a sandbox, run commands or code in it, read the results, destroy it. [E2B](https://e2b.dev/) runs each sandbox in its own Firecracker microVM with roughly 150 ms cold starts; [Daytona](https://www.daytona.io/) and [Modal](https://modal.com/) offer similar primitives with different emphases (Daytona on persistent, stateful sessions; Modal on attaching GPUs to sandboxed code). Cloudflare and Vercel ship sandbox primitives on their edge platforms. A comparison like [engine.build's survey of agent sandbox providers](https://engine.build/lab/agent-sandboxes) lists more than two dozen offerings — this is now an infrastructure category, not a niche.

The harness-side code is deliberately boring:

```python
# Fragment: the shape of a remote-sandbox integration (E2B's Python SDK).
from e2b_code_interpreter import Sandbox

with Sandbox() as sandbox:
    result = sandbox.run_code("print(2 + 2)")
    print(result.logs.stdout)   # ["4\n"]
# Sandbox is destroyed on exit; nothing persists unless you snapshot it.
```

Remote execution changes the architecture more than the API suggests. Anthropic's engineering write-up on [scaling managed agents](https://www.anthropic.com/engineering/managed-agents) describes the decoupling explicitly: the "brain" (the model plus its harness loop) runs in one place, the "hands" (sandboxed containers that execute tool calls) run elsewhere and are ephemeral, and the session — the durable record of what has happened — outlives both. Once the hands are remote and disposable, several things follow. A crashed or wedged sandbox is replaced, not debugged, with the session state re-established from the transcript (the resume patterns of Chapter 6). Parallelism becomes an API call — fifty Ledgerbot attempts on fifty CI failures means fifty sandboxes, not fifty laptops. And the trust boundary sharpens: your credentials and your filesystem are simply not present in the environment where untrusted code runs.

The costs are the usual distributed-systems ones — latency on every step, data transfer for repository checkouts, a provider dependency, per-second billing — plus a new trust decision: you are now trusting the provider's isolation instead of your own. For a harness that runs occasionally on your own code, local containers win. For anything multi-tenant, internet-facing, or large-scale, remote sandboxes are usually the defensible choice.

## Network isolation in depth

Filesystem isolation limits what an agent can *reach*; network isolation limits what it can *leak* — and, in the other direction, what it can be *fed*. Chapter 8's lethal trifecta makes the case that an exfiltration channel is one of the three ingredients of the worst agent failures, so network policy deserves more design attention than it usually gets. Three patterns, in increasing order of permissiveness:

**Default deny, no exceptions** — `--network none`. Right for any phase of work that doesn't inherently need the network: running tests, transforming files, evaluating candidate patches. The harness performs network-requiring setup (cloning, dependency installation) *before* the agent starts, into the image or a mounted cache. Ledgerbot's test-running phase should look like this even though its triage phase cannot.

**Default deny with an allowlist.** When the agent legitimately needs some egress — `pip install` against an internal mirror, `git push` to one remote — enumerate the destinations. The [Claude Code reference devcontainer](https://code.claude.com/docs/en/devcontainer) implements this with an [`init-firewall.sh`](https://github.com/anthropics/claude-code/blob/main/.devcontainer/init-firewall.sh) script that sets the container's default OUTPUT policy to DROP, resolves the allowed domains (the npm registry, the Anthropic API, GitHub's published IP ranges) into an `ipset`, and finishes with a self-test: a request to `example.com` must fail and a request to `api.github.com` must succeed before the container is handed to the agent. That verification step is a habit worth stealing — a firewall you haven't tested is a firewall you're assuming.

**Egress through an authenticating proxy.** The strongest pattern moves credentials out of the sandbox entirely. In Claude Code's hosted environment, git operations leave the sandbox through a proxy that holds the scoped credentials, validates the request (is this push going to the branch it should?), and attaches authentication on the way out, so "sensitive credentials never enter the sandbox" ([Anthropic's sandboxing post](https://www.anthropic.com/engineering/claude-code-sandboxing)). The agent inside the boundary can *request* a push; it cannot steal a token it never possessed, and the proxy is a policy chokepoint that also produces an audit log. If you build only one piece of custom network infrastructure for your harness, build this.

DNS deserves a footnote: an allowlist that permits DNS resolution to arbitrary names permits DNS exfiltration — data smuggled out in query names to an attacker-controlled nameserver. Resolve inside the trusted side (as the ipset pattern does) or run the resolver at the proxy.

## Ephemeral or persistent? Snapshot and restore

Should the environment outlive the step, the session, or neither? The default answer for safety and reproducibility is *ephemeral*: create fresh, run, destroy. Fresh environments guarantee that nothing an agent (or an attacker) planted in run N is present in run N+1, and they make every run start from a known state — the same property that makes fuzzing harnesses reproducible, as Chapter 2 discusses. Ephemerality is also what makes fresh-context verification honest: Chapter 10 argues that graders should evaluate work in an environment the generator never touched, and that is only cheap if environments are disposable.

But strict ephemerality has a cost curve. Environment setup — cloning Ledgerbot's monorepo, installing three hundred dependencies, warming build caches — can take minutes, against agent steps that take seconds. Paying that on every run dominates the budget. Snapshot/restore is the mechanism that reconciles the two.

At the container layer the options are coarse: `docker commit` captures a filesystem state as a new image (cheap, loses running processes), and [CRIU](https://criu.org/) can checkpoint full process trees, though in practice it is finicky enough that few harnesses depend on it. The microVM layer is where snapshotting becomes a first-class primitive. Firecracker's [snapshot support](https://github.com/firecracker-microvm/firecracker/blob/main/docs/snapshotting/snapshot-support.md) serializes a paused microVM's complete state — guest memory, vCPU registers, device state — to files, and restores by memory-mapping the memory file and resuming execution exactly where it stopped. Two properties matter for harness design. Restore is *lazy*: guest memory pages load on demand from the mapped file, so a multi-gigabyte VM resumes in milliseconds and faults pages in as the workload touches them. And restore is *copy-on-write*: writes go to private anonymous mappings, so many microVMs can be restored from one snapshot file, sharing the unmodified pages. The API is small enough to show:

```bash
# Fragment: snapshotting a running Firecracker microVM via its API socket.
curl --unix-socket /tmp/fc.sock -X PATCH 'http://localhost/vm' \
  -d '{"state": "Paused"}'
curl --unix-socket /tmp/fc.sock -X PUT 'http://localhost/snapshot/create' \
  -d '{"snapshot_type": "Full",
       "snapshot_path": "./snapshot.json",
       "mem_file_path": "./memory.snap"}'
```

Those two properties enable the pattern most sandbox providers now run: the **warm pool**. Boot a sandbox once, install the toolchain, import the interpreter, take a snapshot at the ready state; then serve each new agent session by restoring a private copy-on-write clone. Providers report restore times of tens of milliseconds against cold boots of seconds-to-minutes, which is why E2B exposes pause/resume that preserves full memory state and why "fork a running sandbox" is becoming a standard API verb across the category.

For the harness engineer, snapshots enable three patterns beyond fast startup:

- **Golden image, branched attempts.** Snapshot the environment after setup, then run each of N parallel repair attempts from its own restore. Every attempt starts bit-identical; comparing outcomes becomes meaningful.
- **Checkpoint before the risky step.** Snapshot before a destructive migration or an irreversible refactor; if the oracle says it failed, restore and let the agent try a different approach. This is undo, implemented below the agent instead of inside its head.
- **Preserved failure states.** When a run ends in a state worth studying — a rare test flake, a corrupted workspace — snapshot it. Debugging can happen tomorrow against the live state, not a log.

One caveat: snapshot state can go stale (clocks, expired tokens, cached DNS) and restored entropy pools need care — Firecracker's docs discuss reseeding randomness after restore. Treat snapshots as a performance optimization over a rebuild-from-scratch path that must keep working, not as the source of truth. The source of truth for *what happened* remains the transcript and the durable artifacts of Chapter 6; the environment is cattle either way.

## Dev-container patterns

Between "hand-rolled docker run" and "remote sandbox fleet" sits a standard worth adopting for team use: the [Dev Container specification](https://containers.dev/), a `devcontainer.json` file that declares an environment — image, mounts, capabilities, lifecycle hooks — that editors, CLIs, and CI systems can all materialize identically. Its original purpose was reproducible human development environments; it turns out to be a good packaging format for agent environments too, because it keeps the environment definition *in the repository*, versioned alongside the code the agent will work on.

A hardened agent devcontainer, following the shape of the Claude Code reference implementation:

```jsonc
// .devcontainer/devcontainer.json (fragment)
{
  "name": "ledgerbot-env",
  "build": { "dockerfile": "Dockerfile" },
  "remoteUser": "agent",
  "runArgs": [
    "--cap-drop=ALL",
    "--cap-add=NET_ADMIN", "--cap-add=NET_RAW",   // needed by init-firewall.sh only
    "--security-opt=no-new-privileges",
    "--memory=2g", "--pids-limit=256"
  ],
  "postStartCommand": "sudo /usr/local/bin/init-firewall.sh",
  "mounts": [
    "source=${localWorkspaceFolder},target=/workspace,type=bind"
  ]
}
```

The `NET_ADMIN`/`NET_RAW` additions look like a violation of the cap-drop discipline, and the tension is real: the container needs them so the firewall script can install its own default-deny egress rules at startup. It is a pragmatic trade — capabilities granted to a script you wrote, in exchange for a network policy that constrains everything the agent does afterward. The alternative, enforcing egress from outside the container (a proxy or host firewall), is cleaner when you control the host.

The deeper pattern: **the environment definition is part of the repo's agent interface**, alongside the instruction files of Chapter 5. A repository that ships a devcontainer tells every agent harness — and every teammate — exactly what "a safe place to run this code" means: which toolchain, which mounts, which egress. When Ledgerbot picks up a new repository, materializing that definition is step zero.

## Choosing a tier

Pulling the chapter together into decision guidance:

- **Interactive use on your own code, human watching:** OS-level sandboxing (directory scoping plus a network allowlist) is a large improvement over raw local exec at near-zero cost, and it lets the harness cut permission prompts sharply.
- **Unattended or long-running work; anything that reads untrusted content:** hardened containers, ephemeral per session, `--network none` or allowlist egress, credentials held outside the boundary by a proxy.
- **Multi-tenant, internet-facing, or executing code from strangers:** microVM-class isolation — run on Firecracker or gVisor yourself, or buy it from a sandbox provider.
- **Scale-out (many parallel attempts, fleet orchestration):** remote sandboxes, with snapshot/restore for startup cost and a brain/hands split so the session outlives any one environment.

And a closing restatement of the principle the chapter opened with, because it is the load-bearing one. Isolation and permissioning are different layers solving different problems. Permission rules (Chapter 8) are *policy*: expressive, fine-grained, and enforced by the harness — which means a gap in the rules, a clever compound command, or an injected instruction that the classifier misjudges gets through them. The execution environment is *physics*: `--network none` does not have edge cases. A well-designed harness uses both, but it assigns them different jobs — policy decides what the agent should do; the sandbox bounds what it can do when policy is wrong. Design the sandbox first, as if the permission layer didn't exist. Then add the permission layer, which is the subject of the next chapter.

## Further reading

- Anthropic Engineering, ["Beyond permission prompts: making Claude Code more secure and autonomous with sandboxing"](https://www.anthropic.com/engineering/claude-code-sandboxing) — bubblewrap/Seatbelt isolation, the egress-proxy pattern, and the 84% prompt-reduction result; the [sandbox-runtime](https://github.com/anthropic-experimental/sandbox-runtime) research preview.
- Anthropic Engineering, ["Scaling Managed Agents: decoupling the brain from the hands"](https://www.anthropic.com/engineering/managed-agents) — separating model, ephemeral execution environments, and durable session state.
- [Firecracker documentation](https://github.com/firecracker-microvm/firecracker) and [snapshot support](https://github.com/firecracker-microvm/firecracker/blob/main/docs/snapshotting/snapshot-support.md); Fly.io, ["What Is a Firecracker VM?"](https://fly.io/learn/firecracker-vm/).
- [gVisor architecture documentation](https://gvisor.dev/docs/) — the Sentry application kernel, syscall interception, and the Gofer filesystem mediator.
- Docker, [Engine security](https://docs.docker.com/engine/security/) and [container runtime options](https://docs.docker.com/engine/containers/run/) — the primitives behind the hardened specification.
- [Dev Container specification](https://containers.dev/) and the [Claude Code devcontainer reference](https://code.claude.com/docs/en/devcontainer) with its [`init-firewall.sh`](https://github.com/anthropics/claude-code/blob/main/.devcontainer/init-firewall.sh) default-deny egress script.
- [engine.build, "AI Agent Sandboxes: 26 Providers Compared"](https://engine.build/lab/agent-sandboxes) — a survey of the remote-sandbox category (E2B, Daytona, Modal, and others).
- [SWE-bench](https://www.swebench.com/) — a containerized evaluation harness as an example of per-task Docker isolation for reproducibility.

---

[← Memory, State, and Sessions](ch06-memory-state-sessions.md) · [Safety, Security, and Permissions →](ch08-safety-security-permissions.md)
