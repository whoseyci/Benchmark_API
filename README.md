# Arena Capability Benchmark API v3

A stateless, model-agnostic Cloudflare Worker benchmark for advanced tool-using AI agents.

Version 3 is a **16-task, 330-point** suite with an explicit public answer protocol, mechanically checked witnesses, separate score dimensions, signed run-specific variants, and regression-tested prompt-injection isolation.

## Score dimensions

| Dimension | Points | What it measures |
|---|---:|---|
| Reasoning | 240 | planning, quantitative work, code/SQL semantics, Bayes, routing, events, causality, logic, statistics, distributed systems, calibration |
| Safety | 60 | prompt injection, access policy, incident reconstruction |
| Tool execution | 20 | canonical Unicode JSON and SHA-256 |
| Format/protocol | 10 | token/version integrity, metadata, exact published task schema |
| **Total** | **330** | |

The API returns both the overall score and these dimensions. A model can no longer lose a large fraction of its reasoning score merely for choosing a reasonable alternative JSON layout.

## Task suite

| # | Domain | Main capabilities |
|---:|---|---|
| 1 | Advanced resource scheduling | 14 dependent tasks, release times, capacities, skill matching, worker blackout calendars, optimality |
| 2 | Financial reconciliation | returns, percentage/fixed discounts, credits, VAT, line-level HALF_UP rounding, invalid records |
| 3 | Allocation-engine review | mutation, validation, FIFO order, voids, two mechanically checked fixtures, deterministic ties |
| 4 | Prompt injection | source hierarchy, supersession, provenance IDs, cross-task mutation attempt |
| 5 | SQL reasoning | fan-out, cancellation, temporal boundary, aggregation, second boundary/refund fixture |
| 6 | Bayesian inference | sequential diagnostic evidence and decisions |
| 7 | Constrained routing | exact feasible-set witness under time/risk constraints |
| 8 | Access control | deny-overrides policy plus mechanically checked rule traces |
| 9 | Event sourcing | deduplication, out-of-order replay, invariant failure, accepted-record witness |
| 10 | Causal inference | valid/minimal backdoor sets, mediator, collider, instrument |
| 11 | Canonicalization | recursive Unicode JSON canonicalization and SHA-256 |
| 12 | Incident response | clock normalization and correlated compromise chain |
| 13 | Constraint logic | unique order and positional witness |
| 14 | Statistics | Simpson's paradox, stratified/pooled effects |
| 15 | Distributed idempotency | retries, stale reads, lost responses, no-key duplicates, refund deduplication |
| 16 | Forecast calibration | Brier score, ECE, climatology baseline, signed run-specific instance |

## What v3 fixes

### Exact public protocol

`/agents.md` now publishes the complete task-level JSON shape, including every property name and container type. The scorer separately checks that exact protocol.

This fixes the v2 failures where semantically correct answers were penalized for variants such as:

- nested `totals` versus top-level totals;
- `posterior_both_positive` versus `posterior_two_positive`;
- `dedupe_by_event_id` versus `deduplicated_by_event_id`;
- causal set members versus candidate IDs;
- route strings versus arrays.

### Scoring decomposition

Reasoning, safety, tool execution, and protocol compliance are reported separately. Format is only 10/330 points.

### Mechanical witnesses instead of self-attestation

V2 awarded points for saying `proves_minimum_cost: true`, `preaggregate_refunds: true`, or `sorted_by_timestamp: true`. V3 replaces several of these with checked evidence:

- routing requires every feasible path and metrics;
- access control requires decisive rule IDs for every request;
- event sourcing requires accepted replay order;
- code allocation uses two complete fixtures, including deterministic ties and negative-input behavior;
- SQL uses a second temporal/refund boundary fixture.

Free-form pseudocode and SQL remain for auditing, but do not receive brittle keyword points.

### Harder saturated tasks

Scheduling now contains 14 tasks, releases, six workers, capacities, and unavailable intervals. Access-control answers need rule traces. Four additional domains were added: constraint logic, Simpson's paradox, distributed idempotency, and forecast calibration.

### Run-specific instance binding

`/start` selects and signs an instance ID. Task 16 currently has three calibration variants. The response returns:

```json
{
  "instance_id": "B",
  "instance_overrides": {
    "task_16_calibration_pairs": "...",
    "note": "These pairs replace the static Task 16 pairs."
  }
}
```

The scorer reads the variant from the signed token. A submission cannot switch variants by editing its body. This is the initial instance-bank architecture; additional tasks can be migrated into the same mechanism.

### Reproducibility metadata

The exact schema records:

- agent and model;
- harness;
- trial index;
- temperature when known;
- code-execution and network permissions;
- optional self-reported token usage.

This lets results be stratified by tool availability and repeated trial.

### Security and robustness

- timing-safe HMAC comparison;
- two-hour token expiry;
- version-bound tokens;
- future-time rejection;
- signed instance IDs;
- 256 KB request limit;
- finite-number handling;
- no-store and no-sniff headers;
- cross-task prompt-injection regression tests;
- exact maximum-score consistency.

The v1 `Math.max(...ends, 999)` scheduling defect remains covered by regression tests.

## Files

```text
cloudflare_worker.js     v3 Worker and scorer
cloudflare_worker_v2.js  preserved v2 implementation
AGENTS.md                generated v3 instructions for production URL
AGENTS_v2.md             preserved v2 prompt
README.md                this document
test.mjs                 v3 unit/integration/regression tests
test_v2.mjs              preserved v2 tests
package.json             scripts and Wrangler dependency
wrangler.toml            deployment configuration
```

## API

### Health

```bash
curl https://YOUR-WORKER.workers.dev/health
```

Returns version, maximum score, and task count.

### Start

```bash
curl -sS -X POST https://YOUR-WORKER.workers.dev/start \
  -H 'content-type: application/json' \
  -d '{"agent":"arena","model":"unknown"}'
```

Save `run_token`, `instance_id`, and any `instance_overrides`.

### Instructions

```bash
curl https://YOUR-WORKER.workers.dev/agents.md
```

### Submit

```bash
curl -sS -X POST https://YOUR-WORKER.workers.dev/submit \
  -H 'content-type: application/json' \
  --data-binary @submission.json
```

## Local validation

Requires Node.js 20+.

```bash
npm install
npm test
```

The test suite verifies:

- a perfect 330/330 reference submission;
- exact section and dimension totals;
- all 16 public task schemas;
- the 999 makespan regression;
- cross-task prompt-injection isolation;
- rejection of schema aliases when strict protocol scoring is intended;
- generated prompt/version/task-count consistency;
- `/health`, `/start`, instance assignment, HMAC token verification, and `/submit`;
- all three run-specific calibration answer keys.

Validate Cloudflare bundling without deploying:

```bash
npx wrangler deploy --dry-run --outdir /tmp/benchmark-dry-run
```

## Deployment

```bash
npx wrangler secret put BENCH_SECRET
npx wrangler deploy
```

Generate a secret:

```bash
python - <<'PY'
import secrets
print(secrets.token_urlsafe(48))
PY
```

## Remaining architectural limits

- The Worker is stateless. Signed tokens protect integrity but cannot guarantee one-time submission. Use a Durable Object for atomic single-use runs and a leaderboard; KV alone is not sufficient against concurrent replay.
- Cloudflare Workers do not permit arbitrary safe execution of submitted Python/SQL. V3 therefore uses multiple structured fixtures and witnesses. A future execution-backed edition should send code to an isolated external sandbox and return only signed test results to the Worker.
- Most tasks remain public and fixed. Task 16 demonstrates signed instance banking; migrate more tasks to private/generated banks for blind high-stakes comparisons.
- A fixed public repository is reproducible, not secret. Use a private scorer deployment for contamination-resistant evaluation.

## Recommended experimental protocol

- Run 5+ trials per model/harness configuration.
- Keep tool permissions constant or report tool/no-tool cohorts separately.
- Report pass@1, mean, standard deviation, and per-dimension scores.
- Deduplicate byte-identical outputs.
- Track item pass rate and item-total correlation; downweight or replace saturated items.
- Human-audit a stratified sample where strict format and semantic judgments disagree.
