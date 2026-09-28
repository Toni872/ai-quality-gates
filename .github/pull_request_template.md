# Pull request

## What and why

<!-- Two to four sentences. What changes, and the problem it solves. The
     reviewer reads this to know where to look; it is not a substitute for
     reading the diff. -->

Closes #

## Risk tier

<!-- REQUIRED. Check exactly one. Reviewers calibrate their effort by this
     declaration, so a tier set too low is worse than no tier at all: it
     spends the budget in the wrong place.

     If you are unsure, choose the higher tier. Reclassifying upward mid-review
     is cheap. Missing a Tier 2 change is not. -->

- [ ] **Tier 0** — no independent review required. Structural readback of the diff only.
  Typical: documentation, comments, formatting, generated artifacts, dependency bumps with no behavior change, mechanical renames that touch no logic.
- [ ] **Tier 1** — one focused reviewer with a narrow brief.
  Typical: internal refactors, UI copy, non-critical code paths, changes fully covered by tests that already existed and were not modified to accommodate the change.
- [ ] **Tier 2** — full four-lens independent review **plus human line-by-line review plus a written rollback plan**.
  Typical: authentication or authorization, billing or money movement, PII or personal data, schema or data migrations, external API contracts, cryptography, concurrency, anything with a real rollback cost.

Escalation triggers force Tier 2 regardless of how small the diff looks. The full list is in `docs/risk-tiering.md`. If your change touches any of those categories, check Tier 2.

## Tier 2 checklist (complete only if Tier 2)

- [ ] **Rollback plan written before deploy.** Not written during the incident. State the exact commands and the expected outcome. See `docs/release-gate.md`.
- [ ] **Data migration reversibility.** Can this migration be undone? If the schema change is not backward-compatible with the currently deployed application version, a code rollback will not be enough. State the forward-fix path if the migration cannot be reversed.
- [ ] **Backup taken and a restore tested before the migration runs.** Verified before, not after. An untested backup is not a backup.
- [ ] **PII exposure assessed.** Is personal data written, logged, cached, indexed, or returned to a client that previously did not receive it? If personal data becomes reachable from a context that is sent to an AI provider, say so explicitly.
- [ ] **AI-provider data handling confirmed.** Which provider receives source, diffs, or logs for this change, and is there a DPA in place, or a local-only path? See `docs/client-data-safety.md`.
- [ ] **External contract touched.** If this changes a request or response shape, a webhook payload, a rate limit, an error code, or a documented SLA, name the consumer and confirm they were notified.

## Verification

<!-- Check only what you actually ran. An unchecked box here is a factual
     statement that the thing was not verified. -->

- [ ] Tests pass. Paste the exact command and the result, not a description of it.
- [ ] The test command is the one declared in `openspec/config.yaml`, unmodified for this PR.
- [ ] Every dependency newly imported by this diff is actually installed and present in the manifest, at the version the code expects. Not assumed from a transitive install, not assumed because the import looks right. See the dependency hallucination rule in `AGENTS.md`.
- [ ] No secrets, tokens, keys, or credentials in the diff. Checked with the command in `AGENTS.md`.
- [ ] No generated file was hand-edited. If a generated file changed, it was regenerated and both the input change and the output are in this commit.
- [ ] No gate was weakened to make this pass. If a check needed changing, that is its own change.
- [ ] Lint and typecheck run, or their fields in `AGENTS.md` genuinely say `none configured`.
- [ ] A CI run on this exact commit has been observed, not just requested.

## Notes for the reviewer

<!-- Where would you look first? What did you already check, so it is not
     checked twice? What are you unsure about? Saying "I am not sure the retry
     path is correct" saves the reviewer more time than a clean checklist. -->
