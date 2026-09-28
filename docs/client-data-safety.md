# Client data safety

This is the file teams skip, and the reason they skip it is that nothing fails until the day it matters. A lint violation announces itself immediately. A Data Processing Agreement violation is discovered by a client's legal team, after the code has been running in their environment for months.

## Data safety before code safety

If you are choosing between a migration that is not reversible and a lint warning in a function you are not shipping, the migration wins. The lint warning is a cosmetic problem with a two-minute fix. The migration is a data problem with a recovery time you do not control and cannot promise to the client.

This ordering sounds obvious stated plainly, which is exactly why it fails under delivery pressure. Code problems are visible: a red CI, a review comment, a failing test. Data problems are usually invisible until the moment you need the backup, and the moment you need the backup is the moment you are also incident-treating, which is the worst possible time to discover that the backup was never verified.

## The AI-specific risk: what transits a provider

Source code, prompts, diffs, and tool logs may transit an external AI provider. This is a data-processing decision with contractual consequence, not a tooling detail.

The mechanism is usually not obvious, which is why it gets missed. You did not send the client repository to a third party. You ran a local agent that read client source, and the agent's provider received:

- **The diff under review**, which may include the changed client code, and its surrounding context.
- **The prompt**, which frequently contains file contents, error messages, stack traces, and log excerpts pasted in to explain a failure.
- **The rules file**, which contains your internal engineering conventions and, frequently, your internal architecture.
- **Tool output**, which is whatever the agent read. `cat config/database.yml` before fixing a connection string puts production credentials into a provider's request body.

None of those steps feel like "sending client data to a third party", individually. Collectively they are the entire repository, reconstructed. Treat the aggregate as the transmission, because that is what it is.

`.gga` provides the local path for the mechanical layer: `PROVIDER="ollama:<model>"` or `PROVIDER="lmstudio"`. Having that option available and tested is worth the setup time before you accept client work, not during it.

## Provider checklist

Answer every one of these in writing, per client, before the first task. See the last section for where to record the answers.

- [ ] **Is there a Data Processing Agreement with the provider?** Not a privacy policy, not a terms-of-service page. A DPA that names your role as processor or controller, covers the client's data, and specifies sub-processors. If there is not one, that is a contractual gap, and the gap belongs to whoever owns the relationship, not to you.
- [ ] **What does the client contract say about it?** Three distinct cases, and they need different answers. *Permissive*: the MSA permits processing by subprocessors. *Silent*: the contract does not address it, so nobody has decided, and silence is not permission. *Restrictive*: the contract requires data to remain in a defined geography or environment, and a cloud provider outside that boundary is non-compliant regardless of what the provider's own policy says.
- [ ] **Is PII reachable from the context being sent?** This is the question people answer by checking whether the *file* contains PII, which is the wrong check. The relevant question is whether PII is *reachable* from what the agent is looking at. A route handler that reads `req.user.email` reaches PII. A config file that names a production database reaches PII through a query. An error message from a failed user lookup almost certainly contains PII in the stack trace.
- [ ] **Are secrets excluded?** Not "the agent was careful". Structurally. Secrets should be unreachable from the paths the agent reads, and the check is mechanical:

  ```bash
  git ls-files | xargs rg -n --no-heading -e 'AKIA[0-9A-Z]{16}' -e '-----BEGIN [A-Z ]*PRIVATE KEY-----' -e 'sk-[A-Za-z0-9]{20,}' -e 'ghp_[A-Za-z0-9]{36}'
  ```

- [ ] **What is the provider's retention policy?** Whether prompts and completions are retained, for how long, whether they are used for training, and whether a zero-retention or no-training setting exists and is enabled. A default-on training setting means the client's source code is now in someone's model improvement pipeline, permanently, and that is not reversible by deleting your account.
- [ ] **Is there a compliant local path?** Ollama, LM Studio, or a self-hosted model behind your own infrastructure (vLLM, or a cloud-hosted endpoint in the geography the contract requires). This is not an ideal to work toward later. If the answer is currently no, that is a finding to raise before the project starts, because the answer rarely becomes yes at the moment you need it.

## Migrations

The requirement is **reversible by construction**, not reversible with effort.

- **Backup verified before the migration runs.** Not after. Verifying after the migration has already succeeded tells you the backup was not needed, which is not the same as knowing it works. Verify before: restore it, on a scratch target, and confirm the data is intact. A backup you have not restored is a hypothesis.
- **A tested restore, not an available one.** Restoring into a production-shaped environment under time pressure is not a plan. Run the restore beforehand, on a copy, and time it. If the restore takes six hours, your rollback window is six hours, and that number belongs in the release plan rather than being discovered during the incident.
- **Dry run against a production-shaped dataset.** Row counts and data distributions are what make migrations behave differently from staging. A migration that completes in ten seconds on 1,000 rows can hold a lock for forty minutes on 4,000,000. The volume and the distribution both matter; a synthetic dataset with production row counts and synthetic value distribution will still surprise you.
- **Backward-compatible schema changes.** Every migration must be compatible with the currently deployed application version, so that rolling back the code does not require rolling back the schema. Expand and contract: add the new column, write to both, backfill, switch reads, remove the old. See `docs/release-gate.md`.
- **Know the forward-fix path when reversal is impossible.** Some migrations genuinely cannot be reversed: a destructive column drop, an irreversible transformation, a consumed external identifier. For those, the rollback plan is the forward fix, and it must be written and rehearsed before deploy rather than derived during the incident.

## Secrets

- **Never commit.** Not in a config file, not in a test fixture, not commented out, not "just for local". Use environment variables or a secret manager. The pre-commit checklist in `AGENTS.md` has the detection command.
- **Never place a secret in a prompt.** A prompt is a request to an external service. Pasting a connection string to ask why a query fails sends the credential to the provider and puts it in their logs.
- **Never in a log line.** Logging an object that contains a token field logs the token. This includes request headers, connection configuration objects, and user objects that carry auth state.
- **Rotate on suspicion, not on confirmation.** If a secret may have been transmitted, logged, or committed, treat it as compromised and rotate it. Verification takes longer than rotation, and the cost of a false positive is a credential change. A credential that leaked into a git history stays leaked after the commit is deleted, because the object is still in the repository and still in every clone and fork.
- **Check history, not just the working tree.** `git log -p --all -S '<secret>'` finds secrets that were committed and then removed from `HEAD`.

## Logging

- **No PII.** Not in an error message, not in a request log, not in a debug statement left in during an incident. A log line containing an email address is a copy of personal data in a system with different retention, different access control, and a different jurisdictional footprint than your database.
- **No tokens.** No bearer tokens, no session identifiers, no API keys, no cookies. Redact at the serialization layer rather than at each call site, because there is always another call site.
- **No full request bodies.** Log the shape and the identifiers you need to correlate, not the payload. A full body log is a copy of the request in a system nobody threat-modeled.

The structural test: if this log line went to a third-party log aggregator, would you need to tell anyone? If yes, do not log it.

## Write it down before the project starts

The value of these decisions is entirely in their being retrievable later, by you, under time pressure, when someone is asking why. Decide and record all of the following **before** the first client task:

1. **Provider decision.** Which provider, or local-only. With the DPA reference or the explicit statement that none exists.
2. **Contract position.** Permissive, silent, or restrictive — with the clause reference. Silence gets recorded as silence, not resolved in your favor.
3. **What is excluded from context.** Which paths, files, and data classes the agent may read. Specific, not "avoid PII".
4. **Retention settings.** The provider setting that is enabled, and where it is recorded so it can be re-verified.
5. **The local fallback.** Whether it exists, and whether it has been tested. An untested fallback is not a fallback.
6. **Owner and review date.** Who confirmed this, and when it should be re-checked. Provider terms change.

Record it in the repository as a decision record and in Engram as a topic entry. A decision that lives only in the head of the person who made it did not get made, and the person who needs it is not that person.
