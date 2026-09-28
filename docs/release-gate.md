# Release gate

## The plan is written before deploy

Not during the incident. Not while the deploy is running. Before.

The reason is not organizational discipline. It is that the plan's value comes from being complete at the moment you need it, and a plan written under incident pressure will omit exactly the step you need, because the person writing it is reasoning about the outage instead of enumerating the rollback. A rollback plan written calmly in fifteen minutes is complete. A rollback plan written at 2am during a partial outage contains the three steps you already know about.

Write it, put it in the PR, and have it reviewed. A rollback plan nobody read until the deploy is not a plan; it is a hope.

## Reversibility is the property that matters

Review tells you whether the change is correct. It says nothing about whether you can undo it.

A bounded correction process gives you **one fix attempt per candidate**. That constraint is usually understood as a development-speed limit, and it is not. Its real consequence is that when a change is wrong in production, you do not have a loop of increasingly safe edits available to you. You have one shot at a forward fix, and that shot will be made by someone who is also handling an incident. The forward-fix path is therefore a path of last resort under the worst conditions.

Which makes the way back the thing you actually need. Reversibility is a property you engineer, in advance:

- Can this be reverted with a config change rather than a code change?
- Can it be reverted by disabling a flag without a deploy at all?
- Can it be reverted by deploying a previous image, and is that image still in the registry?
- If it cannot be reverted, what is the forward fix, and is it rehearsed?

Ask these at design time, not at deploy time. The answer determines what the rollback plan says, and a change whose answer is "it cannot be reverted" is a Tier 2 change on that basis alone.

## Feature flags and the kill switch

For risky changes, a flag is not a nicety. It is the difference between a rollback that takes seconds and one that takes a deploy cycle, and it is the only mechanism that lets you stop the bleeding while you think.

- **Default the new path to off** for anything Tier 2. Turning it on is a separate, deliberate action with its own observation window.
- **The kill switch is one operation** and does not require a deploy. If disabling the feature means shipping a code change, it is not a kill switch, it is a rollback.
- **The switch is tested before the feature is enabled.** Toggle it in staging and confirm the disabled state actually works. An untested kill switch is a belief.
- **The flag is removed, not left behind.** A flag that outlives its feature is permanent conditional logic nobody remembers the reason for. Schedule the removal at the time you add the flag.
- **Changing the kill-switch mechanism is itself Tier 2.** Disabling the ability to disable is a two-line change with a large blast radius.

## Staged rollout

Each stage has an abort criterion. A stage without one is not a stage; it is a delay before the next stage.

| Stage | Exposure | Abort criterion |
| --- | --- | --- |
| **Internal** | Company traffic, or a synthetic account exercising the full path. | Any error rate above baseline on the new path. Any incorrect result from the synthetic account, without exception. |
| **Canary** | One instance, or a fixed low percentage of real traffic. | Latency p95 or p99 outside the pre-change baseline by more than the threshold you set in advance. Error rate up on the canary relative to the control. Any data-correctness signal, which aborts immediately regardless of magnitude. |
| **Percentage** | 10% → 50%, with a full observation window at each step. | The metric that moved the way it was predicted to move is not moving. A downstream service's error or latency budget is affected. Support volume ticks up. |
| **Full** | 100%. | n/a. Keep the flag and the kill switch live for the whole observation window. Do not remove them in the same week you enable them. |

Set the numeric thresholds **before** the deploy. A threshold chosen while watching a graph is a threshold chosen to justify continuing. The observation window at each stage should be long enough to include a full traffic cycle, which usually means at least one day of business-hours traffic, not ten minutes.

## Migrations must be backward-compatible with the deployed version

A code rollback and a schema rollback are different operations with different risks. Schema rollback is frequently impossible: you have already written new-format data, the old code cannot read it, and there is no way back.

So the rule is: **every migration is backward-compatible with the currently deployed application version.** When you roll the code back, the old code runs against the new schema and works.

Expand and contract, in order:

1. **Expand.** Add the new column, table, or field. Nullable, or with a default. Old code ignores it. No behavior change.
2. **Migrate.** Write to both old and new. Backfill existing rows. Old code still reads the old column and still works.
3. **Switch.** Read from the new column. Old column still being written, so a rollback works.
4. **Contract.** Stop writing the old column. Deploy again.
5. **Remove.** Drop the old column, in a separate deploy, long after the switch.

A rollback at any step 1 through 4 is a code-only rollback. Step 5 is the only step that needs its own justification, and by then nobody needs the old column.

If you cannot make a migration backward-compatible, that is a Tier 2 change with a forward-fix plan, not a rollback plan. Write down which one you have.

## Observability before traffic

The metrics, logs, and alerts that will tell you it broke must exist **before** the deploy.

Deploying first and instrumenting second means debugging blind. When the canary goes red you will be reading application logs at 2am, and application logs are the slowest and least reliable way to answer "is this working". You will be searching for an error message you hope exists, in a log format you hope is structured, for a request identifier you hope was propagated.

Before deploy, confirm all of these exist and are readable by whoever would be on call:

- [ ] **A metric for the new path's correctness**, not only its errors. An error-rate metric on a new path that fails silently — writing wrong data, returning wrong results with a 200 — will stay green through the entire failure.
- [ ] **A metric for the behavior you are changing.** If you are changing how a calculation works, the output distribution must be observable. A latency graph on a calculation that now returns the wrong number is a green dashboard during a data-correctness incident.
- [ ] **Latency and error rate** for the affected endpoint, with the pre-change baseline recorded so "above baseline" means something.
- [ ] **The kill switch**, in a dashboard, with the flag name next to it. Under incident pressure, nobody should be reading source to remember the flag's name.
- [ ] **An alert on the new path** routed to a channel someone is watching. An alert nobody receives is an alert that does not exist.

## What review does not tell you

Review says nothing about production behavior.

Every Tier 2 process described in `docs/risk-tiering.md` and `docs/review-protocol.md` operates on the diff. That is the correct scope, and it has a hard edge. A reviewer can establish that the permission check is logically right, that the migration is backward-compatible, that the retry cannot double-charge. A reviewer cannot establish that production traffic, real data distributions, real network partitions, real cache states, and a real downstream service under real load will produce the behavior the logic implies.

Review answers "is this built correctly". It does not answer "does this work". Those are different questions, they need different evidence, and the honest position is that the second one is only answered by shipping and watching.

So ship, then watch. Roll out in stages, keep the kill switch live for the whole window, and treat the observation period as part of the release rather than as a grace period after it.

## Pre-release checklist

- [ ] Risk tier declared in the PR, and the tier matches the escalation triggers in `docs/risk-tiering.md`.
- [ ] Rollback plan written, reviewed, and linked from the PR. Exact commands, expected outcome, estimated time.
- [ ] Recovery time measured, not estimated. If the restore has not been run, the rollback window is unknown.
- [ ] Kill switch exists, is one operation, requires no deploy, and has been tested in the disabled state.
- [ ] Feature flag defaults to off for Tier 2. Date scheduled for its removal.
- [ ] Migration is backward-compatible with the currently deployed version, or the forward-fix plan is written instead.
- [ ] Staged rollout defined with a numeric abort criterion per stage, set before the deploy.
- [ ] Observation window at each stage covers a full traffic cycle.
- [ ] Metrics, logs, and alerts for the new path exist and are verified readable by the on-call person.
- [ ] Correctness metric exists for the new path, not only an error-rate metric.
- [ ] Pre-change baseline for latency and error rate recorded.
- [ ] Data Processing Agreement and provider path confirmed for anything touching client data. See `docs/client-data-safety.md`.
- [ ] Secrets confirmed absent from the diff, from the prompts used to build it, and from the log configuration.
- [ ] CI green on the exact commit being released, running the command declared in `openspec/config.yaml`.
- [ ] The person releasing knows the abort criterion for the stage they are about to enable, without having to look it up.
