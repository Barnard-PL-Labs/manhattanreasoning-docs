# Quickstart

Install the SDK, program an FPGA, and run a real workload — the brute-force
**SAT solver** — on hardware, in about five minutes.

!!! info "What you need"
    - **Python 3.12 or 3.13.** (Python 3.14 currently ships a broken `ensurepip`
      on Homebrew, so `python -m venv` fails — use 3.12/3.13.)
    - A `CLOUD_FPGA_API_KEY`. Auth is **not enforced** in the prototype, so any
      non-empty value works for now, but the variable must be set.
    - A clone of the [`Cloud_FPGA`](https://github.com/Barnard-PL-Labs/Cloud_FPGA)
      repo (for the SDK and the example designs).

## 1. Install the SDK

```bash
cd Cloud_FPGA

# Use 3.12 or 3.13 explicitly if your default python3 is 3.14
python3.13 -m venv .venv
source .venv/bin/activate

pip install -e ./sdk
```

This installs the `cloud_fpga` package and the `cloud-fpga` CLI:

```bash
cloud-fpga --help
```

## 2. Set your API key

```bash
export CLOUD_FPGA_API_KEY=dev      # any non-empty value during the prototype
```

The SDK targets `https://api.manhattanreasoning.com` by default. Point it
elsewhere with `--api-url` (CLI) or `api_url=` (in code).

## 3. Check the cluster

```bash
cloud-fpga status
```

```text
  ID  STATE           OWNER         CURRENT JOB
  ─────────────────────────────────────────────
   0  idle            -             -
   1  idle            -             -
   ...
```

Pick any board showing `idle`. If a board shows `error`, see
[Troubleshooting](troubleshooting.md#fpga-stuck-in-error).

## 4. Run the SAT solver

The [`sat_solver`](../examples/sat-solver.md) example ships a ready-to-run SDK
app. From the repo root:

```bash
cloud-fpga run examples/sat_solver/client_sdk.py
```

You'll see the FPGA get programmed, then two formulas solved:

```text
[cloud-fpga] programming 'sat_solver' onto FPGA 0 ...
[cloud-fpga] done.
solving ...
SAT  (8 cycles)  model={'x1': True, 'x2': True, 'x3': True}
solving (x1) ∧ (¬x1) ...
UNSAT  (3 cycles)
```

The solver evaluates all clauses in parallel each clock cycle while a counter
sweeps candidate assignments — worst case `2^n_vars` cycles (≈20 µs for 10
variables at 50 MHz).

!!! warning "It may leave the board in `error`"
    The example runs inside `with app:`, which **releases** the session on exit.
    Releasing reflashes the base SoC, which currently fails and parks the FPGA in
    `error`. Recover it with the Redis flush described in
    [Troubleshooting](troubleshooting.md#fpga-stuck-in-error).

## 5. Solve your own formula

The repo includes a generalized runner that accepts a
[DIMACS](https://www.cs.utexas.edu/~marijn/sat/) formula via an environment
variable:

```bash
# (x1 ∨ x2 ∨ x3) ∧ (¬x1 ∨ x2) ∧ (¬x2 ∨ x3) ∧ (x1 ∨ ¬x3)
SAT_FORMULA="1 2 3 0 -1 2 0 -2 3 0 1 -3 0" \
  cloud-fpga run examples/sat_solver/solve.py

# from a .cnf file
SAT_CNF=myproblem.cnf cloud-fpga run examples/sat_solver/solve.py
```

Limits for the default design: ≤10 variables, ≤20 clauses, ≤10 literals/clause.
See the [SAT solver example](../examples/sat-solver.md) for the register map and
a 30-variable variant.

## What's next

<div class="grid cards" markdown>

-   :material-language-python: **[The Python SDK](sdk.md)** — write your own `App`.
-   :material-console: **[The CLI](cli.md)** — every `cloud-fpga` command.
-   :material-sitemap: **[Architecture](../concepts/architecture.md)** — the state machine behind it all.
-   :material-wrench: **[Troubleshooting](troubleshooting.md)** — when a board gets stuck.

</div>
