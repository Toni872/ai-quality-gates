# Review protocol

This document describes what receipt-driven development actually enforces at delivery time, and, just as importantly, what it deliberately does not do.

## The candidate is frozen

A candidate is an immutable set of bytes. Once frozen, reviewers inspect those bytes and only those bytes, through read-only git operations against the frozen trees.

What this rules out, and why each exclusion is load-bearing:

- **Inspecting the live worktree.** The worktree keeps moving. A reviewer who reads the working tree is reading a different artifact from the one under review, and the finding no longer applies to what merges. Worse, a live worktree can be edited to make a reviewer's finding disappear, which turns the review into a negotiation instead of a check.
- **Reading the index or `HEAD` directly.** Those are the *result* of applying the candidate, not the candidate. Reading them means reviewing post-application state with no record of what the change actually did.
- **Passing candidate bytes through a temp file.** A copy is a snapshot that can drift from the original, and a review of a copy is not a review of the candidate. It also breaks provenance: you can no longer prove what was reviewed.
- **Piping candidate bytes through another command** and reviewing the output. A filter, formatter, or normalizer between the reviewer and the bytes can change what the reviewer sees. The reviewer must be the last thing to touch the content.

The practical consequence: if a finding requires a view of the change that frozen, read-only git access cannot produce, that is not a reason to fall back to the worktree. It is a reason to escalate to a human.

## The receipt is the artifact

A receipt is the recorded output of the review: the findings, the verdict, the model and provider used, and the identity of the frozen candidate that was reviewed.

The rules that make a receipt mean something:

- **Delivery gates discover and validate the same receipt. They never start reviewers and never create a budget.** A gate's job is to answer "is there a valid receipt for this candidate". If no receipt exists, the gate fails and a human decides what happens next. A gate that starts a review to produce its own verdict has silently become the review layer, which means its verdict is not independent, and it means the budget is being spent by a process that was supposed to be spending nothing.
- **A receipt is bound to the candidate it reviewed.** If the candidate changes, the receipt is stale and a new review is required. A receipt that survives a change to the bytes it reviewed is worse than no receipt, because it certifies something that no longer exists.
- **A gate must not accept a receipt it cannot validate.** If it cannot confirm the receipt exists, matches the candidate, and covers the required lenses for the declared risk tier, the result is a failure, not a warning.

## Model, provider, and profile selection is user-owned

**Never enable receipt-driven development on a maintainer's behalf.** Not as a convenience, not as part of an onboarding script, not because the rollout table says week 3.

This is a decision with cost, latency, and data-residency consequences attached to it, and those consequences belong to the person who owns the consequences. The choice of model determines what the review can catch and what it cannot. The choice of provider determines where client source code transits. The choice of profile determines what is spent. A maintainer who did not make that choice has not consented to it, and an agent that makes it on their behalf has exceeded its authority regardless of how well the outcome would have served them.

Same rule for the kill switch, in the section below.

## One bounded correction

There is **at most one scoped correction per candidate**.

What this means in practice: reviewers report findings, a human decides which findings are in scope for a correction, the correction is applied as a new candidate, that new candidate is reviewed once, and whatever the second review reports becomes follow-up work rather than a second correction.

There is no loop-until-clean behavior. A process that iterates review and correction until the reviewer has nothing left is not a review process, it is a search for a reviewer-pleasing output, and it will produce code that satisfies the reviewer rather than code that is correct. It is also unbounded in cost, which is exactly the property that makes a gate impossible to budget.

Later observations are follow-ups. Schedule them as their own candidates with their own tiers. A follow-up discovered at correction time is not evidence that the correction failed; it is evidence that the first review had a finite scope, which is what reviews have.

## The independence rule

The verifier must not share context with the author. Shared context does not produce a weaker review; it produces no review at all.

**Wrong:**

- Reviewing code you just wrote in the same session, with the intent and reasoning that produced it still in context. You will confirm your own reasoning, because that is what reading a diff you wrote does.
- The same model reviewing its own output. Models do not lose their blind spots when asked to critique themselves. They restate the same assumptions in a different register, and the restatement reads like an independent observation.
- Reading the PR description, the commit message, or the ticket while reviewing the diff. The description is the author's account of the change. Reviewing against the account confirms the account.
- Reviewing with the diff plus a mental model of what the code "should" do, filling gaps with intent rather than reading what is there.

**Right:**

- Reviewers receive the frozen candidate and the spec. Nothing else. No conversation, no reasoning trail, no PR narrative.
- Multiple reviewers with genuinely different briefs. The Tier 2 four lenses are four different questions, not four samples of one question. A single reviewer asked the same question four times converges on the first answer.
- The human reviewer reads the diff themselves. The automated receipt is an input to that decision, never a replacement for it.
- Where independence cannot be established, the human obligation goes up. Not down.

## Only candidate-caused severe findings block

A finding blocks the merge only if all three hold:

1. It is **severe** — a real defect with a real consequence, not a style preference or an improvement suggestion.
2. It is **caused by the candidate** — the change introduced it.
3. It is **visible in the frozen candidate** — a reviewer can point at the bytes.

Everything else becomes a follow-up:

- **Pre-existing issues** in the base are follow-ups, not blockers. The change did not cause them, and blocking on them turns every PR touching an old file into an impossible task. File a separate issue with the file and line.
- **Base-only issues** — the same, including issues the candidate happens to make easier to notice.
- **Warnings and suggestions** are informational. Record them, do not gate on them.

This rule exists because a gate that blocks on everything blocks on nothing. If a reviewer can block the merge for a pre-existing typo three files away, the next reviewer will stop reporting it, and the real severe finding comes along with it.

## A validator that could not inspect the trees produced no verdict

A correction is validated read-only against the frozen trees.

If the validator could not obtain read-only access to those trees — the tree is gone, the reference does not resolve, a permission failure, a timeout before inspection completed — then it produced **no verdict**. Not a pass. Not a fail. No verdict.

That is a **blocked human decision**, not a failed check. Concretely:

- Do not record it as a passing validation.
- Do not record it as a finding against the candidate. There is no evidence of anything.
- **Do not consume the correction attempt.** The bounded-correction budget is for corrections that were evaluated and found insufficient. A validator that never looked cannot have found the correction insufficient. Spending the single attempt on an infrastructure failure is a bug in the implementation of the process, and it silently removes the one retry the process allows.

Surface it as: validation could not run, here is what blocked it, a human needs to decide. The honest state of a system that could not evaluate something is "unknown", and collapsing unknown into either pass or fail is how a gate starts lying.

## The read-only command surface

These are the only operations a reviewer or validator may use:

```bash
git diff --name-status    # which paths changed, and how (A/M/D/R)
git diff --numstat        # lines added and deleted per path
git diff --stat           # a compact summary
git diff                  # the full patch content
git cat-file              # read object contents by hash, e.g. git cat-file -p <sha>
```

Explicitly forbidden:

- **`git diff --binary`.** It emits raw encoded blobs. A reviewer assessing it is not reading code, and the encoding is a mechanism for smuggling content past inspection.
- **Reading the live worktree, the index, or `HEAD`** in place of the candidate. The candidate is the frozen artifact; the other three are states that can differ from it.
- **Writing temp files** containing candidate content. A copy is a new artifact with no provenance, and a stale copy reviewed as the candidate is worse than no review.
- **Piping candidate bytes through another command** and reviewing the output. No normalization, no formatting, no filtering, no encoding round-trip. The reviewer reads the bytes.

If a required view cannot be produced with these operations, escalate to a human. Do not substitute a convenient approximation.

## Disabling the review mode

Receipt-driven development is disabled for a single repository with:

```bash
gentle-ai review mode disable --scope clone --cwd <repo>
```

`--cwd <repo>` selects which repository, and `--scope clone` scopes the change to that one clone.

**The critical detail: omitting `--scope` defaults to `global`, which disables review for every repository on this machine.** Not the current repository. Every one. A maintainer who runs the command without `--scope` to turn off review for a repo they are annoyed at has silently turned it off for eleven other projects, and will not find out until a review they depended on does not appear.

Always pass both flags. If review mode is behaving unexpectedly, check scope before changing anything else.

**A disabled kill switch is the user's decision.** If a maintainer disables review mode on a repository, that is the owner of the repository exercising control over their own process. Do not argue with it, do not re-enable it, do not work around it by running the review steps manually to produce the receipt the gate now wants, and do not quietly route around the gate. If you think the decision is wrong, say so once, plainly, and then respect it. An agent that overrides a user's explicit kill switch has converted a safety mechanism into an obstacle to be circumvented, which is strictly worse than having no mechanism.
