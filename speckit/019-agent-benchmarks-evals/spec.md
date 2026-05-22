# 019 — Agent Benchmarks & Evals

**Status:** Draft
**Constitutional anchor:** Principle VII (Benchmarked continuously)
**Depends on:** 002, 006, 008
**Required by:** 009, 022

## Purpose

Each agent has its own eval suite, scored on every self-modification and on a nightly cadence. Quality trend visible per-commit. Regressions block self-modification merges. Inspired by SWE-bench's methodology but embedded into the production lifecycle.

## Why this matters

The user's requirement: "I want to have the framework regularly score and benchmark the agents etc, that way we can identify if the agents become smarter or worse, or something like that, and show it even based on commit etc."

Survey: SWE-bench is reproducible but external. Clawie embeds eval as a continuous, production-aware practice.

## Fixture anatomy

```
benchmarks/
├── 001-stripe-billing/
│   ├── fixture.yaml        # task, inputs, environment
│   ├── workspace/          # starting state (mini-repo)
│   ├── expectations.yaml   # rubric (cheap deterministic checks + LLM judge prompts)
│   └── solution.diff       # reference good answer (for comparison)
├── 002-laravel-auth/
│   └── ...
└── README.md
```

`fixture.yaml`:

```yaml
name: stripe-billing
intent: implement-feature
budget_usd: 0.50
timeout_minutes: 10
inputs:
  brief: "Add Stripe checkout to this minimal Laravel app for one Pro tier."
expected_artifacts:
  - path: routes/web.php
    must_contain: ["stripe", "checkout"]
expectations:
  - kind: deterministic
    check: "grep -r 'StripeClient' app/"
  - kind: tests
    command: "php artisan test --filter=StripeBillingTest"
  - kind: llm_judge
    rubric: |
      1. Does the implementation follow Laravel idioms?
      2. Are webhooks handled and verified?
      3. Is the secret reference scoped, not hardcoded?
```

## Functional requirements

| ID | Requirement |
|---|---|
| 019-FR-001 | Every agent MUST ship with at least one fixture. |
| 019-FR-002 | The eval harness MUST be a separate process / runtime — never the production control plane. |
| 019-FR-003 | Each fixture MUST run in a fresh, isolated container per spec 002 (no eval leak into prod state). |
| 019-FR-004 | A fixture run produces: pass/fail, cost, latency, qualitative judge scores. |
| 019-FR-005 | Eval runs MUST be triggered: on every self-modification merge candidate, nightly across the repo, and on-demand by operator. |
| 019-FR-006 | Score = composite (configurable weights of pass-rate, latency, cost, qualitative). |
| 019-FR-007 | Each agent commit MUST be annotated with its score; the score MUST be queryable per commit and per timestamp. |
| 019-FR-008 | Score regression beyond threshold MUST block the self-mod merge. Threshold configurable per agent/team. |
| 019-FR-009 | The dashboard MUST show per-agent score trend (chart) over commits and time, with annotations. |
| 019-FR-010 | Cross-agent benchmarking: composite team-level score available. |
| 019-FR-011 | Fixtures MUST be versioned in the agent repo and reviewable like any other change. |
| 019-FR-012 | An agent MAY propose changes to its own fixture, but such changes are flagged HIGH for operator scrutiny ("agent rewriting its own test"). |
| 019-FR-013 | External benchmarks (SWE-bench, custom) MAY be plugged in as fixture sources. |
| 019-FR-014 | Eval failures MUST be debuggable: the failing fixture's container logs + trace persisted for operator inspection. |

## Non-functional requirements

| ID | Requirement |
|---|---|
| 019-NFR-001 | A typical agent's eval suite MUST complete in <10 min on default hardware. |
| 019-NFR-002 | Eval runs MUST NOT block production tasks (separate worker pool / hardware). |
| 019-NFR-003 | Score variance MUST be tracked; high-variance fixtures flagged as noisy. |

## User stories

- **019-US-001 [P0]** — As an operator, I see coder's score trend over the last 30 commits.
- **019-US-002 [P0]** — As an operator, a self-mod regresses score by 10%; merge is blocked with the diff and the regressed fixture.
- **019-US-003 [P1]** — As an operator, I run `clawie eval coder --fixture stripe-billing` ad-hoc and see results.
- **019-US-004 [P1]** — As an operator, I plug in SWE-bench Verified as a fixture source for coder.

## Acceptance criteria

- Every agent ships a fixture; CI blocks otherwise.
- Score regressions block self-mod merges (with override path).
- Dashboard trend renders correctly with annotations.
- Eval runs isolated; no leak into prod.

## Edge cases & failure modes

- Fixture infinite loop → wall-clock timeout aborts.
- Judge model unavailable → defer eval; mark unscored; surface for operator.
- Fixture explicitly bad (passes trivially) → noise-flagger surfaces low variance.
- Agent rewrites its own fixture to be easier → flagged HIGH; merge requires explicit operator opt-in.

## Related specs

`002-container-runtime-outcall`, `006-observability-audit-cost`, `008-agent-definition`, `009-agent-self-modification-pr`, `022-web-dashboard`.
