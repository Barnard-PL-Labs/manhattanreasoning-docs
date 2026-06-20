# Using the REST API directly

The [Python SDK](sdk.md) is the recommended way to use the platform. If you'd
rather drive the orchestrator yourself — from another language, a shell script,
or to understand what the SDK does — this walks the full lifecycle with `curl`.
The same calls work from any HTTP client. See the
[REST API reference](../api/rest.md) for the complete endpoint contract.

!!! note "Base URL"
    Replace `$BASE` with your orchestrator URL — `https://api.manhattanreasoning.com`,
    or `http://localhost:8000` when [running locally](#run-the-orchestrator-locally).

## Authentication

Two headers are threaded through every request:

| Header | Purpose |
| --- | --- |
| `X-API-Key` | Identifies the caller (recorded as the job/session owner). |
| `X-Session-ID` | Identifies the active session for a reserved FPGA. |

!!! info "Not enforced yet"
    In the current prototype these headers are read but **not enforced** — auth
    will be layered in without changing the route signatures. Send them anyway
    so your integration is ready.

## 1. Find a free FPGA

```bash
curl $BASE/fpga
```

Each entry reports a `state`. Pick one that is `idle`.

## 2. Submit a design

Upload an HDL source file to an idle board. This enqueues a `build_and_program`
job and returns `202` immediately.

```bash
curl -X POST $BASE/fpga/3/submit \
  -H "X-API-Key: $API_KEY" \
  -F "file=@design.py"
```

```json
{ "job_id": "a1b2c3d4-...", "fpga_id": 3, "status": "queued" }
```

## 3. Poll the job

The board moves `queued → building → programming → reserved`. Poll the job until
it is `complete`:

```bash
curl $BASE/fpga/3/jobs/a1b2c3d4-...
```

Stream the build log at any point:

```bash
curl $BASE/fpga/3/jobs/a1b2c3d4-.../logs
```

When the job completes, the FPGA is `reserved` for you.

## 4. Run a transaction

Issue a Wishbone read/write against the programmed design. `op` is `1` for write
and `2` for read.

```bash
curl -X POST $BASE/fpga/3/run \
  -H "X-API-Key: $API_KEY" \
  -H "X-Session-ID: $SESSION_ID" \
  -H "Content-Type: application/json" \
  -d '{ "op": 1, "address": 0, "data": [42] }'
```

This returns a `job_id`; fetch the result once it completes:

```bash
curl $BASE/fpga/3/jobs/<run_job_id>/result
```

```json
{ "ok": true, "data": [42] }
```

## 5. Release the board

Return the FPGA to `idle` so others can use it:

```bash
curl -X POST $BASE/fpga/3/session/release \
  -H "X-API-Key: $API_KEY" \
  -H "X-Session-ID: $SESSION_ID"
```

---

## Run the orchestrator locally

The orchestrator is a FastAPI app backed by Redis.

**Prerequisites:** Python ≥ 3.12 (use 3.12/3.13; see
[Troubleshooting](troubleshooting.md#python-m-venv-fails-with-an-ensurepip-error))
and a running Redis (Docker is the easiest way to get one).

```bash
# From the repo root — use a virtualenv (system/conda Python may refuse
# `pip install` with an "externally-managed-environment" error).
python3.13 -m venv .venv
source .venv/bin/activate

pip install -e "orchestrator[dev]"

# Start Redis (defaults match REDIS_URL=redis://localhost:6379)
docker run -d -p 6379:6379 redis

# Boot the API
uvicorn cloud_fpga_orchestrator.api.app:app --reload
```

The interactive OpenAPI explorer is then available at
`http://localhost:8000/docs` (Swagger UI) — handy while developing.

!!! note "What works today vs. what needs hardware"
    The install/boot steps and the **list → submit → poll → cancel** flow are
    verified end to end against a local Redis: `submit` returns `202` and moves
    the board to `queued`, jobs poll correctly, and the documented `409`/`404`
    guards fire as described.

    The **`run` → `result` → `session/release`** steps require the FPGA to reach
    the `reserved` state, which only happens once a **worker** finishes a
    build-and-program job — and that depends on the build toolchain and physical
    boards being wired up. Until then, those calls return `409 fpga_not_reserved`
    by design.

Next: read [Architecture](../concepts/architecture.md) to understand the state
machine, or the full [REST API reference](../api/rest.md).
