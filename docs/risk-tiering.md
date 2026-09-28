# Risk tiering

Every change gets reviewed. The question is not whether but how hard, and the answer is a function of blast radius, not of diff size.

## The matrix

| Tier | Examples | Reviewers | Human obligation | What "done" means |
| --- | --- | --- | --- | --- |
| **Tier 0** | Documentation edits. Comment and docstring changes. Formatting and whitespace. Generated artifacts regenerated from an unchanged input. Dependency version bumps with no behavior change. Mechanical renames that touch no logic and no public API. Adding a test for existing behavior. | **0 reviewers.** Structural readback of the diff by the author: confirm every hunk is what it claims and nothing is missing. | None beyond opening the PR. | Tests pass, CI green, diff read back against the stated intent. |
| **Tier 1** | Internal refactors with no contract change. UI copy and non-behavioral styling. Non-critical code paths. Additions fully covered by tests that already existed and were **not** modified to accommodate the change. Log message changes. Local configuration values with no security or money impact. | **1 focused reviewer** with a narrow, written brief. Independence required: the reviewer did not write the code. | Reviewer confirms the brief was met and the test coverage claim is real. | Tier 1 reviewer reports no findings above suggestion level. |
| **Tier 2** | Authentication and authorization. Billing, payment, and any movement of money. PII and personal data. Schema and data migrations. External API contracts. Cryptography. Concurrency and shared mutable state. Anything behind a feature flag with a kill switch. Anything a client will see a failure from. | **Full four-lens independent review**, plus **human line-by-line review**, plus a **written rollback plan produced before deploy**. | The named human reads every line. Not a skim, not a diff view at 40% zoom. The human is accountable for the merge. | Four-lens receipt with no candidate-caused severe finding, human sign-off, rollback plan written and reviewed. |

## The cost argument

An unbounded review budget spent uniformly is worth less than a bounded budget allocated by risk.

Consider the arithmetic. A team merges roughly 40 pull requests a week. Suppose you review all of them at Tier 2 rigor: 40 hours a week of senior engineering time. That budget is not a budget, it is an allocation that consumes the entire review capacity of the team, which means the reviews themselves get rushed, which means the rigor was never there in the first place. The apparent rigor is an artifact of the label.

Now suppose the real distribution is 15 Tier 0, 18 Tier 1, 7 Tier 2. The same 40 hours, allocated by risk, becomes roughly 0.4 hours on Tier 0, 4.5 hours on Tier 1, and 35 hours on Tier 2. Every one of those seven Tier 2 changes gets genuine line-by-line attention from someone who can afford to think. The Tier 0 reviews cost almost nothing because a structural readback is a two-minute job.

The failure mode is not that reviewers become careless. It is subtler: when everything is Tier 2, every reviewer learns that the label carries no information. Once a signal has no discriminative power, it stops being read. The reviewer who sees "Tier 2" on a whitespace change has been trained, by repetition, to skim. The next Tier 2 that arrives — the one that moves money — is skimmed by a reviewer who has skimmed forty changes already, and nobody notices because the process said Tier 2 every single time.

If everything is Tier 2, nothing is. That is the whole argument, and it is why the tier must be declared in the PR rather than inferred by the reviewer. A reviewer guessing at risk under time pressure will guess low, and the guess is invisible.

## Escalation triggers

These force **Tier 2** regardless of how small the diff looks. Diff size is not on this list, and that is deliberate.

- **Authentication or authorization.** Any change to a login path, a session, a token, a permission check, a role, a scope, or a policy. This includes changing a *default* permission, which is one line and can expose an entire surface.
- **Billing or any movement of money.** Price computation, tax, proration, refunds, credits, currency handling, rounding, invoice generation, payment state transitions.
- **PII or any personal data.** Collection, storage, logging, caching, indexing, retention, deletion, export, or transmission. Also anything that changes which fields are reachable from a context that leaves your infrastructure.
- **Schema or data migrations.** Including migrations that only add a nullable column, because the column becomes part of every subsequent query plan and every subsequent backup.
- **External API contracts.** Request or response shapes, webhook payloads, error codes, rate limits, pagination semantics, auth handshake, versioning and deprecation policy, published SLAs.
- **Cryptography.** Key generation and rotation, hashing, signing, random number generation, secret storage, transport configuration, anything where a mistake is silent.
- **Concurrency.** Locks, transactions, idempotency, retries, queue consumers, distributed coordination, anything that can interleave.
- **Feature flags with a kill switch**, and any change to the kill-switch mechanism itself.
- **Anything a client will see a failure from.** Not "anything critical". A client-visible failure is a support ticket, a phone call, or a credibility cost, and the review budget exists to buy exactly the avoidance of that.
- **Changes to the gates themselves.** `.github/workflows/ci.yml`, `.gga`, `openspec/config.yaml`, `AGENTS.md`. A change that alters what is checked alters what every future check is worth.

Two questions settle almost every borderline case: *what is the worst thing that happens if this is wrong*, and *how hard is it to undo*. If the answer to the first is a support call, it is Tier 1 at minimum. If the answer to the second is "we would need a migration to undo it", it is Tier 2.

## Worked example: why size is not the risk axis

**A 3-line diff that is Tier 2.**

```diff
- const canEdit = membership.role === "admin";
+ const canEdit = membership.role !== "guest";
```

Three lines. The test suite still passes, because the existing tests covered the admin case and the guest case was never tested. This change inverts the default: every role that is not explicitly a guest now has edit rights. A contractor with a `viewer` role created by a different team, a service account with an unusual role string, a role value that is `undefined` because a partial hydration path omitted it — all of them now have edit access.

Review this one line by line, check the role enumeration, check every place a `UserRole` can be created, confirm the test that now proves the wrong thing is corrected, and write the rollback plan before deploy. Three lines of diff, hours of real work, and a genuine chance of shipping an authorization defect to a client.

**A 400-line diff that is Tier 0.**

```
docs/api-reference.md | 384 ++++++++++++++++++++++++++++++++----
openapi.generated.ts | 16 +-
```

Four hundred lines, regenerated from an updated OpenAPI document by a fixed command. No logic changed. No public type changed in a way that alters behavior. The only thing worth verifying is that the generator is deterministic: run it twice and confirm byte-identical output, and confirm the source document was the thing you intended to change.

Zero reviewers. Structural readback, two minutes, done.

Now consider the reviewer who is told to rank these by effort using diff size. They will spend three minutes on the permission inversion and two hours on the regenerated documentation, and they will be wrong in both directions. The 400-line diff is where reading every line is most wasteful, because the lines are not authored, they are emitted. The 3-line diff is where reading every line is most valuable, because the lines are the entire change and none of their consequences are visible in the diff itself.

The risk axis is the size of the space of things that can go wrong, and that is a property of the code's position in the system, not of how many characters it occupies.

## Recording the decision so it survives

Tiering decays if it lives only in a PR nobody reads three months later. Record it in both places:

1. **A line in the PR.** `Risk tier: Tier 2 — changes the default permission check; worst case is unauthorized edit access on client accounts; rollback is reverting one line, no data involved.` That line is the record reviewers and auditors will find later, and it is searchable with `git log --grep="Risk tier"`.

2. **A topic entry in Engram**, keyed on the change, containing the tier, the trigger that set it, the reasoning, and what was actually verified. This is what survives a session boundary, a team change, and a client asking six months later why a particular change got the attention it did.

```bash
# Find the recorded tiers later
git log --grep="Risk tier" --oneline
```

The rule for updating a record: when a change is reclassified upward after review, correct the PR line and the Engram entry. Do not leave a stale tier behind, because the next person will read the old one and inherit the wrong level of confidence.
