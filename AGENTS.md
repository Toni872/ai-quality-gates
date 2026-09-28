# AGENTS.md

This file is loaded by agents at session start. It is the highest-leverage file in the repository and it is loaded in full, on every session, whether or not the work at hand is relevant to it. That has two consequences you must design for. Noise degrades it: every line that says nothing actionable is context your actual task has to compete with. Lies poison everything: a single invented command, a false claim about what a tool does, or a rule that contradicts the real codebase propagates into every task in every session and is acted upon with full confidence. When you are tempted to add a line that feels reasonable but is not verifiable against this repository, leave it out.

---

## Project facts

> **Required before first use.** An agent reads this section as ground truth. Every field must describe a command that actually exists in this repository, right now, and produces a truthful exit code.

| Fact | Value |
| --- | --- |
| Language | `REPLACE: primary language, e.g. TypeScript 5.x` |
| Package manager | `REPLACE: pnpm / npm / yarn / poetry / pip / go modules / cargo` |
| **Test command** | `REPLACE: e.g. pnpm test` |
| **Lint command** | `REPLACE: real command, or exactly: none configured — do not invent one` |
| **Typecheck command** | `REPLACE: real command, or exactly: none configured — do not invent one` |
| **Build command** | `REPLACE: real command, or exactly: none configured — do not invent one` |
| Primary test framework | `REPLACE: e.g. vitest / jest / pytest / go test` |
| Test location convention | `REPLACE: e.g. src/**/__tests__/*.test.ts` |
| Source location | `REPLACE: e.g. src/` |
| Generated code location | `REPLACE or: none` |

**Never let an agent invent a test or lint command that does not exist.** If the field says `none configured — do not invent one`, that is the complete answer and the agent must not substitute `npm run lint`, `pnpm lint`, `cargo test`, or any other plausible-looking default. A hallucinated lint command produces green CI that lints nothing. A hallucinated test command is worse: it produces green CI and a signed receipt certifying that a change was verified, when no test ever executed. That false evidence is what allows a real defect to reach production behind a process that everyone believes was working. The same rule applies to `openspec/config.yaml`: a `test_command` of `true`, `echo ok`, or `exit 0` is a false-evidence generator and must never be used to make a gate pass.

If a command in this table fails, report the failure. Do not swap in a different command to make it pass, and do not narrow the command's scope to get a green result.

---

## Generated files: do not hand-edit

> **Required.** List this repository's generated artifacts. Hand edits here are silently overwritten on the next generation run, and the discrepancy between the committed file and the file the tool produces is invisible in review because the review diff looks like a normal edit.

| Path | Produced by | Regenerate with |
| --- | --- | --- |
| `REPLACE: e.g. src/generated/` | `REPLACE: e.g. prisma generate` | `REPLACE: e.g. pnpm prisma generate` |
| `REPLACE: e.g. src/api/schema.ts` | `REPLACE: e.g. openapi-typescript` | `REPLACE: e.g. pnpm gen:api` |
| `REPLACE: e.g. pnpm-lock.yaml` | `REPLACE: package manager` | `REPLACE: pnpm install` |

Rules for generated paths:

- Never hand-edit a generated file. Change the input, then regenerate, then commit both the input change and the regenerated output in the same commit.
- If a diff contains changes only to generated files, the correct action is to regenerate and verify the output is reproducible, not to review it line by line. Two runs producing different bytes for the same input is a real defect in the generator, and it is the finding that matters.
- Never edit a lockfile by hand. Run the package manager.
- If a generated file is hand-edited in a diff, treat it as a Tier 1 finding at minimum: the edit will be lost, and any behavior that depended on it is now undocumented.

---

## Risk tiering rule (condensed)

Full specification in [`docs/risk-tiering.md`](docs/risk-tiering.md). The rule in short:

- **Tier 0** - docs, comments, formatting, dependency bumps with no behavior change, mechanical renames. Verification is a structural readback of the diff. **0 reviewers.**
- **Tier 1** - internal refactors, UI copy, non-critical code paths, changes fully covered by existing tests. **1 focused reviewer** with a narrow brief.
- **Tier 2** - authentication, authorization, billing, PII, data migrations, external API contracts, cryptography, concurrency, and anything with a real rollback cost. **Full four-lens independent review, plus human line-by-line review, plus a written rollback plan produced before deploy.**

**Why tiering exists.** Review fatigue kills a process faster than bad code does. A reviewer who is asked to give the same scrutiny to a comment reformat as to a permission check stops doing either, and the process becomes a checkbox that trains the team to trust it when it does not matter. If everything is Tier 2, nothing is. That is the failure mode this rule exists to prevent, and it is strictly worse than having no review process, because it consumes the budget and produces false assurance.

When you are unsure of the tier, take the higher tier. Escalation triggers are enumerated in `docs/risk-tiering.md` and they override the examples here.

---

## Independence rule

**The verifier must not share context with the author.** An independent review by definition has a different information state from the thing it is reviewing. Shared context is not a weaker review; it is not a review. It is the author reading their own reasoning back and confirming it, which is the one thing the layer exists to prevent.

**Wrong patterns, all of which are common:**

- Reviewing code you just wrote in the same session, in the same context window, where you still hold the intent and the reasoning that produced it.
- The same model reviewing its own output. A model does not lose its own blind spots when prompted to critique itself; it re-expresses the same assumptions in a different register.
- Reading the PR description, commit message, or task ticket while reviewing the diff. The description is the author's account. Reviewing against the author's account confirms the description, not the code. Review the diff against the spec and the code as it is.
- Reviewing with the author's diff and your own mental model of what the code "was supposed to do" filling the gaps, instead of the frozen candidate bytes.

**Correct patterns:**

- Reviewers receive the frozen candidate and the spec, and nothing else. Not the conversation, not the reasoning trail, not the PR narrative.
- Multiple reviewers with genuinely different briefs, so that one blind spot does not become the verdict. The Tier 2 four lenses are deliberately different questions, not four samples of the same question.
- The human reviewer reads the diff themselves. An automated receipt is an input to that decision and never a substitute for it.
- Where a reviewer cannot obtain an independent view, that is a reason to escalate the human obligation, not a reason to accept the review as complete.

---

## Dependency hallucination rule

**Before trusting any code that imports a package, verify that the package is actually installed.** This one check catches the single most common failure mode of AI coding assistants: an import for a library that does not exist in the project's dependency graph. The failure is not a compile error, because the author frequently writes the test too, and the test asserts the invented behavior. The result is internally consistent, green, and wrong.

Exact commands per ecosystem:

```bash
# Node — pnpm
pnpm ls <pkg>
pnpm why <pkg>

# Node — npm
npm ls <pkg>
npm explain <pkg>

# Node — yarn
yarn why <pkg>

# Python (pip)
pip show <pkg>
python -c "import <module>; print(<module>.__file__)"

# Python (poetry)
poetry show <pkg>

# Python (uv)
uv pip show <pkg>

# Go
go list -m all
go list -m -f '{{.Path}} {{.Version}}' all | grep <module>

# Rust
cargo tree -i <crate>
cargo tree -p <crate>

# .NET
dotnet list package
```

Additional requirements:

- A transitive dependency is not a direct dependency. If new code imports a package directly, it must be added directly to the manifest, not merely present transitively through another package. A transitive import works until the intermediate package changes its own dependencies, and then it fails in a build nobody can reproduce.
- Verify the version, not only the presence. An import satisfied by `^1.2.0` resolving to a different major than the code was written against is a latent break.
- If the dependency is legitimately missing, adding it is a separate, declared change with its own version and license review. Do not add a package mid-refactor to make a diff compile.
- Check the license of every new dependency against this project's distribution constraints before it lands. A copyleft or source-available dependency in a client-delivered product is a contractual issue, not a lint issue.

---

## Known tooling pitfalls

These are verified gotchas. Treat this list as a pre-flight checklist, not as background reading.

- **`gentle-ai install` with no `--component` flags installs a broad default set.** That default set includes a `permissions` component that can overwrite hand-tuned security rules in your agent configuration. **Always run `gentle-ai install --agent <name> --dry-run` first** and read the diff it reports before applying anything. When you only need part of the stack, scope the install with `--component` rather than installing the default set and undoing the parts you did not want.
- **Agent prompt assets drift out of sync with the binary.** If `~/.gentle-ai/state.json` contains `installed_agents: null`, `gentle-ai sync` cannot repair anything, because it has no inventory to sync from. Register the agents first, then sync. A null inventory is a registration failure, not a sync failure, and re-running sync will not fix it.
- **A stale local agent prompt can reference commands that do not exist in the installed binary version.** After any version bump, grep the agent configuration for command names and verify each one against `gentle-ai --help`. A prompt that instructs an agent to call a removed subcommand produces confident, well-formatted invocations of nothing.
- **`gentle-ai doctor` is the health check. Treat a red line as a real defect, not noise.** A failing check is the tool telling you the system is not in the state it claims to be in. Fix it or escalate it; do not filter it out of the output you report.
- **Tool version provenance.** A tool reporting a `dev` version, which is common when installed from a `--depth 1` clone where the installer stamps the version from `git describe --tags`, means you cannot tell which version you are running. Behavior differs between releases. Pin real released versions and record the version in your lockfile or in this file, so that "it worked yesterday" is a diagnosable statement.

---

## Pre-commit and pre-merge checklist

Run these before you claim a change is ready. Each line is a command with a specific failure it catches.

**Pre-commit**

```bash
# 1. Working tree state: nothing uncommitted or unstaged slips into the commit under review
git status --porcelain

# 2. The diff under review is what you think it is
git diff --name-status
git diff --stat

# 3. Project gates — use the exact commands from "Project facts" above, not substitutes
<test_command>
<lint_command>          # skip only if the field says "none configured"
<typecheck_command>     # skip only if the field says "none configured"

# 4. No secrets committed. Tracked and staged
git ls-files | xargs rg -n --no-heading -e 'AKIA[0-9A-Z]{16}' -e '-----BEGIN [A-Z ]*PRIVATE KEY-----' -e 'sk-[A-Za-z0-9]{20,}' -e 'ghp_[A-Za-z0-9]{36}' 2>/dev/null
git diff --cached | rg -n --no-heading -e 'password\s*[:=]' -e 'secret\s*[:=]' -e 'token\s*[:=]'
```

**Pre-merge**

```bash
# 1. Tests pass on the exact commit being merged, not on a previous one
git rev-parse HEAD
<test_command>

# 2. Every new import resolves to a real installed dependency
#    (see "Dependency hallucination rule" for the per-ecosystem command)

# 3. The branch contains no generated-file hand edits
git diff --name-status <base>...HEAD
#    For each changed path listed as generated: confirm the generator reproduces it

# 4. Risk tier is declared in the PR. An undeclared PR is treated as Tier 1
# 5. If Tier 2: human line-by-line review done, and a written rollback plan
#    exists in the PR before deploy (see docs/release-gate.md)
# 6. No change to CI or gate configuration rides along with a feature change.
#    A PR that weakens a gate must be reviewed as its own Tier 2 change
```

**Rules that apply to the whole chain:**

- Never disable or weaken a check to make a change pass. If a check is wrong, fix the check in a separate, declared change.
- A PR that modifies `.github/workflows/ci.yml`, `.gga`, `openspec/config.yaml`, or this file is a Tier 2 change on its own merit, because it changes what the gates are.
- Never mark a check as passing that you did not run, or that you ran and that failed. Report failures as failures.
