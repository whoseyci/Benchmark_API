# Arena Capability Benchmark API

A stateless Cloudflare Worker benchmark for Arena.ai-style agents.

## What this is

- No database.
- Same tasks for all models.
- Agent gets `AGENTS.md`, starts a run, submits JSON, gets score.
- Hidden scoring logic lives inside the Worker.
- Wall-clock time is tracked via a signed run token.
- True LLM token usage is not available unless the agent platform reports it.

## Files

```text
cloudflare_worker.js      paste into Cloudflare Worker
AGENTS.md                 optional static copy / template
README.md                 deployment instructions
wrangler.toml             optional CLI deploy
```

## Cloudflare Dashboard deployment

1. Go to Cloudflare Dashboard.
2. Open **Workers & Pages**.
3. Click **Create application** → **Worker**.
4. Name it, e.g.:

```text
arena-capability-benchmark
```

5. Click **Edit code**.
6. Delete the default Worker code.
7. Paste the full contents of:

```text
cloudflare_worker.js
```

8. Save and deploy.

## Add secret

In the Worker settings:

```text
Settings → Variables → Add variable
```

Add a secret variable:

```text
BENCH_SECRET = a long random string
```

Example local generation:

```bash
python - <<'PY'
import secrets
print(secrets.token_urlsafe(48))
PY
```

This signs run tokens. Without it, the Worker uses a development fallback and should not be trusted.

## Get AGENTS.md

After deploy, open:

```text
https://YOUR-WORKER.YOUR-SUBDOMAIN.workers.dev/agents.md
```

Copy that markdown and upload/paste it to the Arena.ai agent.

The generated `agents.md` contains the correct Worker URL and all tasks.

## Test endpoints

Health:

```bash
curl https://YOUR-WORKER.YOUR-SUBDOMAIN.workers.dev/health
```

Start:

```bash
curl -s -X POST https://YOUR-WORKER.YOUR-SUBDOMAIN.workers.dev/start \
  -H 'content-type: application/json' \
  -d '{"agent":"manual","model":"unknown"}'
```

Submit:

```bash
curl -s -X POST https://YOUR-WORKER.YOUR-SUBDOMAIN.workers.dev/submit \
  -H 'content-type: application/json' \
  --data-binary @submission.json
```

## Notes

Because this is stateless, the same run token can technically be submitted multiple times. Preventing retries or building a leaderboard requires D1/KV/R2 storage. For now, this is intentionally just:

```text
results in → score out
```

## Limitations

- No persistent leaderboard.
- No true token tracking.
- No retry prevention.
- Hidden scoring is hidden only because the Worker code is not shown to the model. If you make the Worker repo public, the scoring is public.
