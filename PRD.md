# PRD: Software Factory on Claude Code

Nov 17, 2026 · Lorenzo Uriel

## Summary

We will build a software factory: a system that turns well-specified tickets into reviewed, tested pull requests without a human writing code. It runs Claude Code through the Claude Agent SDK in isolated containers, one per ticket, and keeps humans at two gates: plan approval (for risky ticket classes) and PR merge.

The agents do the judgment work: planning, coding, reviewing. Deterministic orchestrator code decides everything else: when work advances, retries, or escalates. "Done" means the target repo's own `make verify` passes and independent reviewer agents agree, never the coding agent's own assessment.

The pilot targets two repositories and low-risk ticket classes (dependency bumps, test backfills, small bug fixes). The expected model cost is about $500–$950 a month for 200 tickets.

## Problem and context

Engineering time goes to repetitive, well-understood changes that block feature work. Dependency upgrades, test coverage gaps, small bug fixes and schema tweaks pile up because each one is too small to prioritize and too tedious to enjoy.

Interactive AI assistants help one engineer at a time, at the keyboard. They don't scale to a backlog: someone still has to start every session, watch it, and decide when it's done.

By 2026 the capability is there. Companies such as Spotify, Stripe and Ramp merge thousands of agent-written PRs a week. What separates a working factory from a demo is not the model; it's the harness around it: isolation, deterministic verification, retry budgets and clear escalation.

## Goals and non-goals

**Goals**

- Turn tickets labelled `factory:ready` into PRs that pass verification and review, with no human in the loop between pickup and PR.
- Reach a merge-without-edit rate of 70% or more on pilot ticket classes within 8 weeks.
- Serve any number of target repos through a per-repo contract (`.factory.yaml`), with no factory code changes per repo.
- Make every run observable, replayable per stage, and capped in cost.

**Non-goals**

- Removing human merge review. A dark factory with no review is out of scope for this phase.
- Greenfield product design or ambiguous feature work. Those stay interactive.
- Deploying to production. The factory stops at an open PR; existing CI/CD handles the rest.
- Replacing the team's ticketing or CI systems. The factory plugs into GitHub and Azure DevOps as they are.

## Users and use cases

Three groups use the factory: engineers who file and review tickets, tech leads who approve plans and tune autonomy per repo, and the platform owner who runs the factory itself.

| Ticket class | Example | Autonomy | Pilot? |
| --- | --- | --- | --- |
| Dependency bump | Upgrade a library to the next minor version | Full: straight to PR | Yes |
| Test backfill | Add tests for an untested module | Full | Yes |
| Small bug fix | Fix a failing edge case with a reproducible test | Full | Yes |
| Schema change | Add an index or nullable column in SQL Server | Plan approval first | Yes |
| Cross-repo change | New column in the DB repo, exposed in the API repo | Plan approval first | After pilot |
| Small feature | New endpoint following existing patterns | Plan approval first | After pilot |

## Prior art

The design follows the pattern already proven in production. All three companies below isolate each task, trigger from events, and let deterministic verification decide when work is done.

| Company | System | Engine | Reported scale | Human gate | Lesson we take |
| --- | --- | --- | --- | --- | --- |
| [Spotify](https://engineering.atspotify.com/2026/6/code-with-claude-coding-is-no-longer-the-constraint) | Honk | Claude via Agent SDK, in Kubernetes pods | 1,500+ merged PRs; migrations 60–90% faster | PR review | Closest to our design: same engine, background agent, fleet-wide maintenance first |
| [Stripe](https://stripe.dev/blog/minions-stripes-one-shot-end-to-end-coding-agents-part-2) | Minions | Fork of Block's Goose | 1,300+ merged PRs a week | PR review | Strong test suite and a hard cap of 2 CI rounds per run |
| [StrongDM](https://factory.strongdm.ai/) | Software Factory | Own agent (Attractor) | Not disclosed | None (dark factory) | Hidden test scenarios the agents can't see; we keep human review instead |

Ramp also reports that its Inspect agent writes over half of merged PRs in its frontend and backend repos ([Ona white paper](https://ona.com/files/background_agents_white_paper.pdf)).

## Solution overview and architecture

The factory has five layers. Only the execution layer calls a model; everything that decides whether work moves forward is ordinary, testable code.

```
┌─ INTAKE ─────────────── where work comes from ───────────────────────────┐
│  GitHub Issues            Azure DevOps             Schedule              │
│  label factory:ready      work item tags           cron or webhook       │
└───────────────────────────────────┬──────────────────────────────────────┘
                                    │ claim ticket
┌─ ORCHESTRATION ──────── deterministic Python, no LLM ────────────────────┐
│  Queue adapter       State machine       Gates + budgets    Escalation   │
│  claim, transition   stages, retries     plan, size, $ caps needs-human  │
└───────────────────────────────────┬──────────────────────────────────────┘
                                    │ spawn pod per ticket
┌─ EXECUTION ──────────── one container per ticket, Claude Agent SDK ──────┐
│  Planner             Implementer         Reviewers          Guards       │
│  read-only, plan.md  edits, Stop hook    fresh, JSON verdict deny push   │
└───────────────────────────────────┬──────────────────────────────────────┘
        ▲ red or veto: retry        │ changes to check
┌─ VERIFICATION ───────── the only definition of done, no LLM ─────────────┐
│  make verify              Diff checks               Integration verify   │
│  lint, types, tests       size, allowed/forbidden   cross-repo tests     │
└───────────────────────────────────┬──────────────────────────────────────┘
                                    │ green and approved
┌─ DELIVERY AND OPS ───── a human merges ──────────────────────────────────┐
│  Pull request             Telemetry                 Run artifacts        │
│  linked, idempotent       OTel spans, $ per ticket  plan, review, logs   │
└──────────────────────────────────────────────────────────────────────────┘
```

Work enters at the top and leaves as a PR at the bottom. A red verify or a reviewer veto loops back to execution, within the attempt budget.

## End-to-end flow

Each ticket passes five stages in its own container. A ticket leaves only two ways: as a verified, reviewed PR, or escalated to a human with the reason attached.

```
 factory:ready
      │            ┌──── red or veto, attempts left: feedback ────┐
      ▼            ▼                                              │
 ┌─────────┐   ┌───────────┐   ┌─────────────┐   ┌──────────────┐   ┌───────────────┐
 │  Plan   │──▶│ Implement │──▶│   Verify    │──▶│    Review    │──▶│    Deliver    │
 │ planner │   │ Stop hook │   │ make verify │   │ 2 fresh      │   │ push, open PR │
 │ + gate  │   │ : green   │   │ diff+paths  │   │ agents, JSON │   │ label review  │
 └────┬────┘   └─────┬─────┘   └──────┬──────┘   └──────────────┘   └───────┬───────┘
      │ plan gate    │ BLOCKED.md     │ budget spent                        │ PR opened
      │ fails        │                │                                     ▼
      ▼              ▼                ▼                             ┌───────────────┐
 ┌──────────────────────────────────────────────────┐               │ Human review  │
 │ ESCALATE: label factory:needs-human,             │               │ merge, or     │
 │ comment why + artifacts                          │               │ comments      │
 └──────────────────────────────────────────────────┘               │ re-queue      │
                                                                    └───────────────┘
```

Retries share one budget: by default 3 implement attempts, plus caps on turns per stage, dollars per ticket and wall-clock time. For ticket classes set to plan approval, the run pauses after Plan until a human comments `/approve`.

The ticket's labels are its state, so anyone can see progress from the issue tracker:

| Label | Meaning | Set by |
| --- | --- | --- |
| `factory:ready` | Spec is ready for the factory | A trusted engineer |
| `factory:in-progress` | Claimed; a pod is working on it | Orchestrator |
| `factory:review` | PR open, waiting for human review | Orchestrator |
| `factory:needs-human` | Escalated; the comment says why | Orchestrator |

Changes requested on a PR move the ticket back to `factory:ready`, with the review comments as input to the next run.

## Functional requirements

| ID | Requirement | Priority |
| --- | --- | --- |
| FR-1 | Poll GitHub issues and Azure DevOps work items for `factory:ready`; claim each ticket atomically so two runs never take the same one. | Must |
| FR-2 | Run each ticket in its own container with a fresh git worktree; no state shared between tickets. | Must |
| FR-3 | Planner agent (read-only tools) writes `plan.md` from a template; a deterministic gate checks sections, file scope and size. | Must |
| FR-4 | For ticket classes set to `plan-approval`, post the plan as a comment and wait for `/approve`. | Must |
| FR-5 | Implementer agent follows `plan.md`; it writes `BLOCKED.md` instead of guessing when the plan can't be followed. | Must |
| FR-6 | A Stop hook blocks the agent from finishing while `make verify` fails, feeding the failure output back. | Must |
| FR-7 | The orchestrator re-runs verification itself and enforces diff size and allowed/forbidden paths. | Must |
| FR-8 | Reviewer and security-reviewer agents run in fresh sessions and return a JSON verdict; any fail feeds back into the next attempt. | Must |
| FR-9 | Cap each ticket by attempts (default 3), turns per stage, dollars per ticket and wall-clock time; escalate on any limit. | Must |
| FR-10 | Open an idempotent PR with plan summary, verify results and cost; label the ticket `factory:review`. | Must |
| FR-11 | Escalate with label `factory:needs-human` and a comment that says why and links the run artifacts. | Must |
| FR-12 | Re-queue a ticket when a reviewer requests changes on its PR, with the review comments as input. | Should |
| FR-13 | Replay any single stage from saved artifacts for debugging. | Should |
| FR-14 | Support multi-repo changesets: one plan, ordered implementation, integration verify, linked PRs. | Could (phase 3) |

## Repository structure and target repo contract

The factory lives in one repo; each target repo carries only its own context and a contract file. A fix to a shared agent ships once, and onboarding a new repo is a PR to that repo, not a factory change.

```
software-factory/
├── orchestrator/factory/   # queue adapters, pipeline, stages, gates, guards, scm, telemetry
├── orchestrator/tests/     # pipeline tests with a fake agent and a fixture repo
├── agents/                 # planner, implementer, test-writer, reviewer, security-reviewer
├── skills/                 # write-plan, tsql-migration, dbt-model, python-service
├── hooks/                  # require-green.sh, format-on-edit.sh, guard-bash.sh
├── templates/              # plan, review schema, PR body, escalation comment
├── target-repo-template/   # what a repo copies in to onboard
├── runtime/                # Dockerfile, entrypoint, per-stack toolchain images
├── deploy/                 # GitHub Actions, Azure Pipelines, k8s CronJob + Job template
├── observability/          # OTel collector, Grafana dashboard, alerts
└── repos.yaml              # registry of target repos

target-repo/
├── CLAUDE.md               # build/test commands, architecture rules, never-do list
├── .claude/settings.json   # permissions + hooks; optional agent/skill overrides
├── .factory.yaml           # the contract
└── Makefile                # make verify = lint + typecheck + tests
```

The contract tells the orchestrator everything it needs about a repo:

```yaml
verify: make verify            # the only definition of done
allowed_paths: ["src/**", "tests/**", "migrations/**"]
forbidden_paths: ["infra/prod/**", ".github/workflows/**"]
max_diff_lines: 400
ticket_classes:
  dependency-bump: { autonomy: full }
  feature:         { autonomy: plan-approval }
reviewers: [reviewer, security-reviewer]
```

Run artifacts (`plan.md`, `review.json`, `verify.log`, `BLOCKED.md`, the SDK transcript) live outside the checkout, so they never land in a commit.

## Multi-repo changesets

Tickets that touch one repo need nothing extra: each runs in its own pod under that repo's contract, in parallel. A ticket that spans repos becomes a changeset: one plan, ordered implementation, an integration check, and linked PRs.

1. The planner gets read access to every repo in the changeset and writes one `changeset-plan.md` with a section per repo, the order, and the compatibility strategy.
2. Implementation runs one session per repo, in dependency order from `depends_on` in `repos.yaml`, so each session loads that repo's own CLAUDE.md, permissions and verify command.
3. Each repo passes its own verify and review. Then a new integration-verify stage runs the consumer's tests against the producer's branch (for example, API tests against a SQL Server container with the branch's migrations applied).
4. Delivery opens linked PRs that reference each other and state the merge order ("1/2, merge first").
5. If any repo fails, the whole changeset escalates and no PRs are opened.

The planner must use expand/contract: every step stays backward-compatible (add a nullable column, then adopt it in the consumer, then tighten it in a later ticket). With that rule, merge order can't break production, so no atomic cross-repo merge is needed.

## Non-functional requirements

**Security**

- Each ticket runs in a throwaway container with a scoped Git token: no admin rights, no force-push, branch protection on `main`, and no production credentials.
- A Kubernetes network policy limits egress to the Anthropic API, the git host and package mirrors.
- Agent tools are restricted per stage: planner and reviewers are read-only; a PreToolUse hook denies `git push`, `curl`, destructive commands and reads of `.env` or secrets. Only the orchestrator pushes.
- Issue text is treated as data. Only trusted authors can apply `factory:ready`, which limits prompt injection through tickets.
- Authentication uses an Anthropic API key from the Console, or Bedrock, Vertex or Foundry credentials, not a personal subscription login.

**Reliability**

- Stages are idempotent and hand off through files, so any stage can be re-run from its inputs.
- Every run is bounded by attempts, turns, dollars and wall-clock time; nothing loops forever.
- `factory:needs-human` works as a dead-letter queue with the reason and artifacts attached.

**Observability**

- One OTel trace per ticket, with a span per stage tagged with ticket id, repo, attempt and cost.
- Claude Code's own telemetry is exported to the same collector.
- A Grafana dashboard shows throughput, escalation rate, merge-without-edit rate and cost per ticket, with alerts on escalation and cost spikes.

## Cost model

Model tokens are the main cost: about $2.40 for a medium ticket on Sonnet 5.5, and $470–$950 a month for 200 tickets depending on the model mix. These figures are estimates; we replace them with measured `total_cost_usd` from the first 20 real tickets.

Prices used ([pricing page](https://platform.claude.com/docs/en/about-claude/pricing)): Opus 5.5 at $4 / $20 per million input / output tokens, Sonnet 5.5 at $2 / $10, cache reads at $0.20 (Opus) and $0.10 (Sonnet). Cache writes are assumed at about 1.25× the input price. Agent loops re-send their context every turn, so a cache-hit rate of 85–90% is assumed.

| Ticket class | Tickets / month | All Sonnet 5.5 (USD) | All Opus 5.5 (USD) |
| --- | --- | --- | --- |
| Small (dependency bump), ~$0.55 each on Sonnet | 120 | 66 | 131 |
| Medium (bug fix), ~$2.40 each on Sonnet | 60 | 143 | 287 |
| Large (feature or changeset), ~$10 each on Sonnet | 20 | 203 | 406 |
| Plus ~15% waste from escalated runs | | ~60 | ~125 |
| **Total** | **200** | **~470** | **~950** |

The recommended mix is Opus for planning and review (few tokens, high judgment) and Sonnet for implementation (most tokens), at about $3.15 per medium ticket. Infrastructure is small: pods run for minutes on the existing Kubernetes cluster, and observability uses the existing Grafana stack.

The largest real cost is human review: at 200 PRs and 10 minutes each, about 33 engineer-hours a month. Fewer attempts per ticket and a higher merge-without-edit rate lower both bills more than a cheaper model does.

## Success metrics

The headline metric is merge-without-edit rate: it measures trust in the output, and it drives both review time and token cost.

| Metric | Definition | Pilot target (week 8) |
| --- | --- | --- |
| Merge-without-edit rate | PRs merged with no human commits on the branch ÷ PRs opened | ≥ 70% |
| Escalation rate | Tickets labelled `needs-human` ÷ tickets claimed | ≤ 20% |
| Attempts per ticket | Mean implement attempts per delivered PR | ≤ 1.6 |
| Cost per delivered PR | Total model spend ÷ PRs opened, waste included | ≤ $4 |
| Lead time | Ticket claimed → PR opened, median | ≤ 30 minutes |
| Post-merge defects | Reverts or hotfixes traced to factory PRs within 14 days | 0 for full-autonomy classes |

A ticket class moves from plan approval to full autonomy only after 20 consecutive tickets above the merge-without-edit target.

## Rollout plan

The pilot runs eight weeks in four phases, and each phase starts only when the previous gate is met. Weeks are relative to kickoff.

```
 ┌─────────────────────┐ ┌─────────────────────┐ ┌─────────────────────┐ ┌─────────────────────┐
 │ Foundations         │ │ Orchestrator MVP    │ │ Pilot               │ │ Scale               │
 │ Weeks 1 to 2        │ │ Weeks 3 to 4        │ │ Weeks 5 to 8        │ │ Week 9 onward       │
 │                     │ │                     │ │                     │ │                     │
 │ CLAUDE.md+contract  │ │ queue + pipeline    │ │ K8s pod per ticket  │ │ multi-repo          │
 │ make verify, 2 repos│ │ plan, build, review │ │ deps, tests, fixes  │ │   changesets        │
 │ claude -p+Stop hook │ │ fake-agent tests    │ │ OTel + Grafana      │ │ more repos+classes  │
 │ by hand, 5 tickets  │ │ approval, every plan│ │ autonomy by class   │ │ cost tuning, evals  │
 └─────────────────────┘ └─────────────────────┘ └─────────────────────┘ └─────────────────────┘
 ──────────────────────◆───────────────────────◆───────────────────────◆──────────────────────▶
                     Gate 1                  Gate 2                  Gate 3
              verify catches defects   10 tickets end to end   70% merged unedited
```

Gate 3 also requires escalation at 20% or below. If a gate isn't met, the phase extends; autonomy never rises on schedule alone.

## Risks and mitigations

| Risk | Impact | Mitigation |
| --- | --- | --- |
| Weak test suite lets plausible but wrong code through | High | Onboard only repos whose `make verify` catches real defects; add tests before raising autonomy |
| Vague tickets produce confident wrong work | High | Plan gate rejects missing acceptance criteria; plan approval for risky classes |
| Runaway loops or cost spikes | Medium | Attempt, turn, dollar and wall-clock caps per ticket; cost alerts |
| Prompt injection through ticket or repo text | High | Trusted labellers only, read-only stages, deny-list hook, no secrets and limited egress in the pod |
| Review fatigue: humans rubber-stamp agent PRs | High | Small diffs (`max_diff_lines`), PR body shows plan and verify results, track post-merge defects |
| Shared agent prompt change degrades every repo | Medium | Version agents and skills in git; run the fixture-repo test suite before each release |
| Model or SDK changes break the harness | Medium | Pin SDK versions; the fake-agent test suite catches pipeline regressions |

## Open questions

- [ ] Which two repositories make up the pilot, and does each already have a `make verify` that catches real defects?
- [ ] Is the ticket source GitHub Issues, Azure DevOps, or both?
- [ ] Where do pods run: the existing Kubernetes cluster, or hosted CI runners?
- [ ] Which cloud is billed for model usage: Anthropic Console, Bedrock, Vertex or Foundry?
- [ ] Who approves plans for schema changes, and what is the approval timeout before escalation?
- [ ] What monthly budget cap applies to the pilot?

## Sources

- [Spotify Engineering: Coding Is No Longer the Constraint](https://engineering.atspotify.com/2026/6/code-with-claude-coding-is-no-longer-the-constraint)
- [Stripe: Minions, Part 2](https://stripe.dev/blog/minions-stripes-one-shot-end-to-end-coding-agents-part-2)
- [Port: Inside the Software Factory, Stripe](https://software-factories.port.io/stripe)
- [Ona: Background agents white paper](https://ona.com/files/background_agents_white_paper.pdf)
- [StrongDM: Software Factories and the Agentic Moment](https://factory.strongdm.ai/)
- [Claude API pricing](https://platform.claude.com/docs/en/about-claude/pricing)
- [Vexa: model credentials and licensing](https://docs.vexa.ai/model-credentials-licensing)