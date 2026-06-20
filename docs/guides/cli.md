# The CLI

`cloud-fpga` is installed with the [SDK](sdk.md). It programs FPGAs, inspects
cluster and job state, and pokes registers on a live board.

```bash
cloud-fpga --help
cloud-fpga <command> [options]
```

**Global options** (accepted by every command):

| Option | Default | Purpose |
| --- | --- | --- |
| `--api-key KEY` | `$CLOUD_FPGA_API_KEY` | API key. |
| `--api-url URL` | `https://api.manhattanreasoning.com` | Orchestrator base URL. |

## `run` — program and run an app

```bash
cloud-fpga run <file.py> [--fpga-id N] [--no-program]
```

Loads the [`cloud_fpga.App`](sdk.md#app) from `file.py`, programs the FPGA, then
calls its [`@local_entrypoint`](sdk.md#applocal_entrypoint).

| Option | Purpose |
| --- | --- |
| `--fpga-id N` | Override the `fpga_id` set in the file. |
| `--no-program` | Skip programming (the board already has this design loaded). |

```bash
cloud-fpga run examples/sat_solver/client_sdk.py
cloud-fpga run my_design.py --fpga-id 3
```

## `status` — show FPGA states

```bash
cloud-fpga status [fpga_id]
```

With no argument, prints the full table; with an `fpga_id`, shows that board's
detail (state, owner, current job, session).

```text
  ID  STATE           OWNER         CURRENT JOB
  ─────────────────────────────────────────────
   0  idle            -             -
   3  reserved        dev           a1b2c3d4
```

States: `idle`, `queued`, `building`, `programming`, `reserved`, `error` — see
[Architecture](../concepts/architecture.md#fpga-state-machine).

## `read` — read register words

```bash
cloud-fpga read <fpga_id> <address> [--count N]
```

`address` is a **byte** address, hex or decimal. Reads `--count` words (default
1) from a live (reserved) board.

```bash
cloud-fpga read 0 0x10            #   0x10: 0x00000005
cloud-fpga read 0 0x20 --count 4
```

## `write` — write a register word

```bash
cloud-fpga write <fpga_id> <address> <value>
```

Both `address` and `value` accept hex or decimal.

```bash
cloud-fpga write 0 0x04 3
cloud-fpga write 0 0x00 0x1
```

## `reset` — return a board to idle

```bash
cloud-fpga reset <fpga_id>
```

Reflashes the base SoC and returns the board to `idle`.

!!! warning "Reset can fail in the prototype"
    The base-SoC reflash currently fails and can leave the board in `error`. If
    `reset` doesn't recover a board, use the Redis flush in
    [Troubleshooting](troubleshooting.md#fpga-stuck-in-error).

## `job`, `logs`, `cancel` — job control

```bash
cloud-fpga job    <fpga_id> <job_id>     # status + metadata
cloud-fpga logs   <fpga_id> <job_id>     # build log (great for build failures)
cloud-fpga cancel <fpga_id> <job_id>     # cancel a queued job
```

When a `run` build fails, `cloud-fpga logs <fpga_id> <job_id>` prints the Yosys /
nextpnr output so you can see what the synthesizer rejected.

## See also

- [Python SDK](sdk.md) — the `App` / `RegisterMap` API the CLI drives.
- [REST API](../api/rest.md) — the HTTP endpoints behind each command.
