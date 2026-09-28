# AI Quality Gates

This repository is a reusable template that bootstraps an AI-assisted delivery quality stack for teams shipping client-facing software with AI coding assistants. It contains no application code. It contains the constitution, configuration, and operating rules that you copy into a real project so that an agent working in that project has a written specification layer, a mechanical pre-commit gate, an independent pre-merge review layer, a durable record of why decisions were made, and a single file that states the house rules the agent is required to read. The five layers are designed to be orthogonal, not redundant, because five gates that all answer the same question waste inference budget and destroy the team's trust in green checks.

## The five layers and the five questions they answer

| Layer | Question it answers | When it fires |
| --- | --- | --- |
| SDD (spec) | Are we building the right thing? | Planning, not review |
| gga (pre-commit hook) | Does this change respect house style? | Every commit; cheap and mechanical |
| RDD (receipt-driven development) | Is this change independently verified correct? | Before merge |
| Engram | Do we know why we are doing this? | Across sessions |
| AGENTS.md | Does the agent know the rules? | Always loaded |

### These are not five duplicate gates

Stacking checks that answer the same question is the most common way a quality stack becomes theater. Each layer here has a distinct failure mode it is the sole owner of, and the first two mechanical layers are explicitly orthogonal:

- **gga checks style against `AGENTS.md`.** It reads a rule file and diffs the working tree against it. It cannot judge whether a function is correct, and it never launches an independent reviewer.
- **RDD runs independent reviewers against a frozen candidate.** It has no notion of house style, formatting, or lint. It has no way to check a rename. It only asks whether the change is correct and safe.

Running a full independent review on a three-line documentation fix, or running a style hook on a permission-check change, is the same mistake in both directions. The gating rules below exist to keep each layer on its own question.

## Quick start

### Consume the template

1. GitHub: **Use this template** on this repository, which creates a new repository with this content and a clean history.
2. Or clone and copy:

```bash
git clone https://github.com/<owner>/ai-quality-gates.git
rsync -a --exclude '.git' ai-quality-gates/ /path/to/your-project/
```

3. Or copy individual files into an existing repository that is already in flight. All paths in this template are repository-root-relative and are safe to copy one at a time.

### Phase 0 bootstrap, in order

Run these steps in this order. Skipping step 2 and doing step 5 first is how teams end up reviewing work that was never specified.

1. **Copy the guardrails.** `AGENTS.md`, `.gga`, `openspec/config.yaml`, `.github/workflows/ci.yml`, `.github/pull_request_template.md`, and the `docs/` directory.

2. **Fill in `openspec/config.yaml`.** Set `test_command` to the command that actually runs your test suite. Leave `lint_command` and `typecheck_command` as `null` unless those commands genuinely exist. This is the single most consequential edit in the whole bootstrap; see the warning in `AGENTS.md` under "Never let an agent invent a command".

3. **Fill in the "Project facts" block in `AGENTS.md`.** Language, package manager, real test command, real lint command, real build command, real typecheck command. For any command that does not exist in your project, write the literal string `none configured — do not invent one`. Do not leave a plausible-looking placeholder in a command field.

4. **Verify CI runs your real test command on your real code.** Push a deliberately failing test on a scratch branch and confirm CI goes red, then revert. A CI pipeline that has never been observed failing is not evidence of anything.

5. **Install the gga pre-commit hook.**

```bash
gga init
gga install
```

`gga init` is required per repository. `gga install` is what actually registers the pre-commit hook. See `.gga` for the provider configuration, including the fully local options required for client work.

6. **Tune the gga rules before you trust them.** `RULES_FILE="AGENTS.md"` means gga enforces whatever rules `AGENTS.md` states. Add only the rules you are willing to enforce mechanically. A pre-commit hook that produces false positives gets disabled by the team within a week, and once a hook is disabled it never comes back.

7. **Declare the risk tiers in your pull request template.** `.github/pull_request_template.md` already contains the tier checkboxes. Reviewers calibrate effort by the declared tier; an undeclared PR is treated as Tier 1.

8. **Confirm RDD model selection is yours.** Receipt-driven development must never be enabled on a maintainer's behalf. Selection of model, provider, and profile is a user-owned decision made explicitly, and the review kill switch is a user-owned decision as well.

## Phased rollout

Do not enable everything on every repository in week one. Review fatigue kills a process faster than bad code does.

| When | What you enable | Why this order |
| --- | --- | --- |
| Week 1 | Repository constitution (`AGENTS.md`) plus the real test command wired into CI. Nothing else. | The constitution and a truthful CI signal are the only two things everything else depends on. There is no value in gating a repo whose baseline is unknown. |
| Week 2 | gga pre-commit hook on **one** repository. Tune rules until the false-positive rate is acceptable. | You cannot tune rules without observing them. One repo is enough signal, and a false-positive flood on the whole org is how gga gets banned. |
| Week 3 | RDD on **Tier 1 changes only.** | Tier 1 work is bounded, so a reviewer finding per change is finite and predictable. This is where you learn whether the receipt is worth its cost. |
| Week 4+ | RDD on Tier 2, with mandatory human line-by-line review and a written rollback plan. | Tier 2 work carries real rollback cost. Automated review is an input to a human decision, never a substitute for it. |

The rollout order is the same every time. Constitution before enforcement, baseline before gate, one repo before all repos, bounded tier before unbounded tier.

## What this stack does NOT protect you from

This section is the point of the template. Everything above is structural verification, and structural verification has a specific and narrow power.

**A wrong spec.** SDD produces a written, reviewable statement of intent. It does not validate the business decision inside that intent. If the requirement is wrong, the spec is a rigorous, well-formatted, independently reviewed document describing the wrong thing. Garbage in, verified garbage out. The spec layer makes a wrong decision *visible and arguable*; it does not make the decision right.

**Tests that encode the author's own hallucination.** An agent that invents an API also invents the test that asserts the invented API behaves as imagined. That test passes. It proves the hallucination is internally consistent. Your test suite cannot detect a misunderstanding that the test author shares; only a test written from a contract, a spec, or a real observed behavior can. This is the single largest gap in AI-assisted delivery, and no structural gate closes it.

**Runtime environment differences.** Local pass, CI pass, production fail, because the difference between the environments is not the code. Timezone defaults, locale-dependent collation, filesystem case sensitivity, CPU float behavior, connection pool sizes, a missing environment variable that only exists on one host, a library that behaves differently on Windows and Linux. None of the tools in this stack reason about deployment topology. Your staging environment, not your test suite, is what surfaces this.

**Race conditions and distributed-systems failure.** Data races, deadlock, split brain, partial writes, non-idempotent retries, eventual consistency windows, clock skew, out-of-order message delivery. These are not code-reading problems. A reviewer looking at a diff cannot see the interleaving that fails, and a green test suite for a race condition is a test that did not exercise the race. Only soak testing, fault injection, and load testing surface these, and none of them are in this stack.

**A human rubber-stamping a review.** The weakest link in the chain is still a person in a hurry, and the whole chain is only as strong as that link. A Tier 2 change that gets a perfunctory "LGTM" in a Slack thread has passed a process while receiving none of its benefit. `docs/review-protocol.md` requires the independence rule specifically because self-review is the default failure mode, not the exception.

**Legal or contractual obligations no linter knows about.** Data residency requirements, retention windows defined in a signed MSA, export controls, professional licensing constraints, accessibility obligations under the relevant regulation, a contractual uptime commitment with a defined remedy. These are real contractual exposure and no static analysis tool in this stack has any representation of your contract.

**Sending client code to an external AI provider without a Data Processing Agreement or an on-prem model.** This is a data-processing decision with legal consequence, not a tooling detail, and it is the most common way client work goes wrong. Source code, prompts, diffs, and tool logs may transit a provider outside your contractual relationship with the client. `docs/client-data-safety.md` is the checklist. `.gga` documents the local-provider configuration (`ollama:<model>`, `lmstudio`) that exists for exactly this case, and you should treat having that path available as a precondition for accepting client work at all.

## Requirements, assumptions, and open questions

**This template has no test command of its own, because it has no application code.** That is deliberate. `.github/workflows/ci.yml` detects the project type from marker files and exits successfully with a notice when it finds none, so this template's own CI is green and consumers get a real run the moment they add code.

**Consumers MUST set their real test command in `openspec/config.yaml`.** A wrong or placeholder test command produces false evidence, and false evidence is worse than no evidence because it converts a missing gate into an apparently satisfied one. A command that exits 0 without running anything, such as `echo "no tests"`, will make every downstream layer certify work that was never exercised.

**Assumed:** a Git repository with a `main` branch, GitHub as the hosting platform for CI and pull request templates, and an agent runtime that reads repository-root `AGENTS.md` at session start. If your agent loads rules from a different path, keep `AGENTS.md` at the root as the canonical copy and point your runtime at it.

**Assumed:** the project uses a supported ecosystem for the CI matrix (Node, Python, or Go) or you will edit the matrix. The Rust and Java toolchains are not in the detection matrix in this template; add them rather than deleting the layer.

**Open question for the consuming project:** what is the real review cost of Tier 2 work in your context, measured in engineer-hours per change. Set that number before enabling Tier 2 automated review, because the review budget is finite and the allocation rule in `docs/risk-tiering.md` only works if you know the ceiling.

**Open question for the consuming project:** which data may transit which AI provider. Write that down before the first client task, not during an incident. See the closing section of `docs/client-data-safety.md`.

## Documentation

- [`docs/risk-tiering.md`](docs/risk-tiering.md) - full Tier 0 / Tier 1 / Tier 2 matrix, escalation triggers, and worked examples showing why diff size is not the risk axis.
- [`docs/review-protocol.md`](docs/review-protocol.md) - what receipt-driven development actually enforces, the frozen-candidate rule, bounded correction, and the exact read-only command surface.
- [`docs/client-data-safety.md`](docs/client-data-safety.md) - provider data handling, migration reversibility, secrets, and logging rules for client work.
- [`docs/release-gate.md`](docs/release-gate.md) - rollback planning, staged rollout with abort criteria, backward-compatible migrations, and observability before traffic.

The constitution itself is [`AGENTS.md`](AGENTS.md). Read that one first; it is the file your agent loads.
