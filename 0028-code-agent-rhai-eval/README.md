---
title: "28. Code agent on feature-scale epics, graded without reference solutions"
status: Active
topics:
  - evaluation
  - code-agent
---

# 28. Code agent on feature-scale epics, graded without reference solutions

Date: 2026-09-26 (first results; experiment continues)
Authors: Bill Murdock, with Claude Code
Tracks Jira AISDLC-5. Builds on [0006](../0006-code-agent-evaluation/) (the
first code-agent evaluation, 20 synthetic bug-fix scenarios) and
[0026](../0026-eval-statistical-significance/) (how many trials a comparison
needs). The runnable eval lives in `fullsend-ai/agents` under
`eval/code-rhai/` (proposed in a pull request from this work; see Related).

## Hypothesis

The production code agent can be measured on real, feature-scale work, the
epics an engineering organisation actually hands it, without anyone writing
reference solutions or hidden tests, and that measurement is sharp enough to
tell two agent configurations apart.

Secondary questions: how the unmodified agent compares with a cheaper model
on the same work, and whether the agent's own review step is useful evidence
for a model judge.

## Approach

**Cases are recipe cards, not bundles.** A case is an epic from a public
Red Hat AI repository, given to the agent verbatim as a GitHub issue carrying
the `ready-to-code` label, against the repository as it was just before the
humans' first accepted PR for that epic (or the default-branch tip if none
has merged). The card records the upstream repository, the snapshot commit
and the rule that chose it, the human PRs (used only to pick the snapshot,
never shown to the agent or the judge), and the repository's own test
command. The repository tree is rebuilt from the card at run time and never
committed. Five cases were built from the September 2026 epic sample:

| Case | Epic | Repository | Language |
|---|---|---|---|
| 001 | RHAI-517 capability manifest endpoint | trustyai-explainability/nemo-guardrails | Python |
| 002 | RHAI-331 deprecate legacy checks | trustyai-explainability/nemo-guardrails | Python |
| 003 | RHAI-284 pod- and model-level dashboard panels | opendatahub-io/kserve | YAML/Go |
| 004 | RHAI-369 Konflux onboarding for a component | opendatahub-io/odh-dashboard | TypeScript/Dockerfile |
| 005 | RHAI-503 Data Connect Hub views | opendatahub-io/odh-dashboard | TypeScript |

**Grading has no reference.** Each run is classified first: `pr` (the agent
opened a PR), `declined` (it stopped and said why), or `failed`. Deterministic
judges then check that the repository's own tests still pass on the PR head
and that fullsend's review agent ran. A tool-using model judge reads the task,
the diff, the PR write-up, the test result, the review agent's findings and a
checkout of the PR head, and scores 1 to 5 against the task's acceptance
criteria. A decline is scored separately on whether its stated reason holds
up against the task and the repository. Every artifact is kept so a different
judge can re-score later without re-running agents.

**Two arms, same pipeline.** Arm 1 is the code agent exactly as the fleet
runs it (Claude Code runtime, Opus 4.6, effort high). Arm 2 swaps only the
model and runtime (pi runtime, GPT-6 Luna, effort high). The review step is
the production review agent in both arms, so it is a constant. The judge is
the same model for both arms.

## Results

### First look, 2026-09-25/26: Sonnet 4.6 vs Luna, five cases, different judges

A pipeline shake-down rather than a result: the Sonnet arm was judged by
Opus 5 and the Luna arm by GPT-6 Sol, so the arms are not comparable, and
the host slept for most of the night. Kept because it produced the Luna
refusal (below) and the first PR-quality scores.

| Case | Sonnet 4.6 | Luna |
|---|---|---|
| 001 | PR, tests pass, quality 4/5 | declined (repo AGENTS.md), decline 4/5 |
| 002 | PR, tests pass (19 tests deleted), quality 3/5 | declined (unspecified sunset date), decline 4/5 |
| 003 | PR, checks pass, quality 3/5 | PR, checks pass, quality 3/5 |
| 004 | PR, tests pass, quality 3/5 | PR, tests pass, quality 3/5 |
| 005 | PR, tests fail (lockfile not regenerated), quality 2/5 | did not finish (infrastructure) |

### 2026-09-26 and 2026-10-06: Opus 4.6 (fleet as-is) vs Luna, five cases, one judge

Cases 001, 003, 004 on 09-26 and 002, 005 on 10-06, same configuration and
the same agent checkout throughout. Judge: GPT-6 Sol on the codex runner, one
sample per case. Reviewer: the production review agent on Claude Code, Opus
4.6 orchestrator, sub-agents on their configured tiers. "Fleet as-is" means
the agents repository's harness defaults; the fleet's own deployed
configuration has since moved to newer models, so this baseline is dated.

| Case | Opus 4.6 | Luna |
|---|---|---|
| 001 | PR, 95 turns, tests pass, 12 review findings, quality 3/5 | declined (same reason as before), 13 turns, decline 3/5 |
| 002 | PR, 62 turns, tests pass (21 test functions removed, 11 added), 22 findings, quality 3/5 | declined, 9 turns, decline 3/5 |
| 003 | PR, 54 turns, checks pass, 7 findings, quality 3/5 | PR, 53 turns, checks pass, 6 findings, quality 3/5 |
| 004 | PR, 76 turns, tests pass, 5 findings, quality 3/5 | PR, 29 turns, tests pass, 6 findings, quality 3/5 |
| 005 | PR, 248 turns, tests fail (lockfile not regenerated), 16 findings, quality 1/5 | PR, 172 turns, tests pass, 16 findings, quality 3/5 |

Totals under this judge: Opus 4.6 five PRs, quality 3, 3, 3, 3, 1 (mean
2.6), repo tests pass on 4 of 5; Luna three PRs at 3, 3, 3 (mean 3.0) with
tests passing on all three, plus two declines at 3. Cost at published rates:
Opus coder $29 for five cases; Luna coder about $0.50; the eight production
reviews $4 to $11 each ($55 total; the two largest PRs each drew a review
over $10); judging about $1 per verdict.

### Three judges on the same ten artifacts

The stored artifacts were re-scored without re-running any agent: Opus 5 on
the first six (via Vertex), then Opus 5.5 on all ten (via the EnMaaS gateway
on the harness side). One sample each.

| Artifact | GPT-6 Sol | Opus 5 | Opus 5.5 |
|---|---|---|---|
| Opus 4.6, 001 PR | 3 | 3 | 3 |
| Opus 4.6, 002 PR | 3 | - | 3 |
| Opus 4.6, 003 PR | 3 | 3 | 3 |
| Opus 4.6, 004 PR | 3 | 3 | 4 |
| Opus 4.6, 005 PR | 1 | - | 1 |
| Luna, 001 decline | 3 | 2 | 2 |
| Luna, 002 decline | 3 | - | 2 |
| Luna, 003 PR | 3 | 3 | 4 |
| Luna, 004 PR | 3 | 4 | 4 |
| Luna, 005 PR | 3 | - | 4 |

Every judge is within one point of every other on every artifact. The
disagreements all lean the same way: the stronger judge rates Luna's PRs
higher (mean 4.0 under Opus 5.5 against 2.8 for Opus 4.6) and its declines
lower (2, for leaving no proposal or plan behind, and in one case for a false
claim about upstream). Opus 5.5 is now the eval's judge for both arms.

### Observations

- **Reproducible refusals, on both sides.** Luna declined RHAI-517 in every
  run (five of five) and RHAI-331 in every run (three of three), each time
  because the repository's `AGENTS.md` and `CONTRIBUTING.md` require an
  assignee and an agreed approach before non-trivial work, and for RHAI-331
  also because the epic leaves the deprecation sunset date to a pending
  product decision. Sonnet and Opus implemented both. On RHAI-331 Opus chose
  a sunset date itself, which the judge flagged as "asserted without
  evidence of the required PM decision". The judges rated Luna's declines
  2 to 3: facts correct, but no proposal or plan left behind, and the RHAI-331
  decline included a false claim about which endpoint an upstream PR had
  added. Obedience versus initiative, visible from both sides because the
  outcome classification does not hide a decline as a failure.
- **Judges agree, and the stronger one separates more.** The all-3s of the
  first Sol run were partly the work and partly the judge: on identical
  artifacts Opus 5.5 found 4s that Sol did not, always in Luna's favour on
  PRs and against Luna on declines, while never disagreeing by more than a
  point. Judge choice changes the gap between arms, not the ranking of
  individual artifacts.
- **Luna is cheap, and not worse here.** Roughly 60 times cheaper than Opus
  4.6 per case, with equal or better judge scores on the three cases both
  completed. Opus's extra turns did not buy quality on the largest case: 248
  turns produced a bigger scaffold that does not install, where Luna's 172
  turns passed the repo's tests. The sample is far too small to conclude
  anything; it sets the stake for the question.
- **One repo convention decided every test failure.** Both Claude models,
  on different days, added a workspace package to the odh-dashboard monorepo
  without regenerating `package-lock.json`, so `npm ci` refused to install.
  Luna did not. Repository-specific conventions of this kind are where
  "agent readiness" of a repo shows up in the numbers.
- **Reported costs can be wrong.** The pi runtime priced GPT-6 Luna at a
  Claude-tier rate, about 50 times its published price. Costs above were
  recomputed from token counts.
- **The review step is expensive relative to the coder** on these cases ($4
  to $6 per review against $2 to $4.50 per Opus implementation). Whether the
  judge needs it is an open question worth a controlled test.

## Limitations

One sample per case, one run per arm, five cases, one sample per judge.
Runs were made from a laptop against a personal cloud budget; nothing here
is a fleet measurement yet. Two of the five cases are from the same
repository. The baseline is the agents repository's harness default of late
August 2026, not the fleet's current deployed configuration.

## Conclusion

Open. The pipeline works end to end on real epics without reference
solutions, and it already measures something the synthetic benchmark in 0006
could not: how an agent behaves when the repository's own rules push back.
The quality signal is consistent across three judges; what it lacks is
sample size.

Next: repeated judge samples per 0026 on the stored artifacts; a second case
set built from 0006's twenty scenarios, which also brings in the safety
traps; a re-baseline against the agent as currently deployed; a finer
outcome taxonomy that separates "asked for clarification" from "declined"
and judges the stated need; an instruction-compliance judge; the
"implementation, not PR" instruction as a matrix arm; then one treatment at
a time (a minimal platform-contract-only agent as a floor, and practices
from epic-code-gen).

## Related

- `fullsend-ai/agents`: `eval/code-rhai/` (pipeline, scripts, cards); the
  proposing pull request is linked from this experiment's pull request.
- fullsend-ai/fullsend#7748, a hardening note filed from this work.
