# Troubleshooting

Known issues in the current prototype and how to work around them.

## FPGA stuck in `error`

**Symptom.** After a run, `cloud-fpga status` shows a board in `error`, and new
`run`/`submit` calls against it fail.

**Cause.** Releasing a session (`app.release()`, leaving a `with app:` block, or
`cloud-fpga reset`) reflashes the **base LiteX SoC**. That reflash currently
fails (a missing `base.bit` on the orchestrator), stranding the board in `error`.
Because the in-band `reset` does the same failing reflash, it can't recover the
board either.

**Fix.** Flush the orchestrator's Redis state, which returns every board to
`idle`. The `Cloud_FPGA` repo ships a helper:

```bash
# point it at your key + droplet, then run
export FPGA_SSH_KEY=~/.ssh/id_ed25519       # the team's DigitalOcean key
export FPGA_DROPLET=root@<orchestrator-ip>
examples/bert_ffn/reset_idle.sh
```

Or run the one line it boils down to:

```bash
ssh -i "$FPGA_SSH_KEY" root@<orchestrator-ip> "redis-cli FLUSHALL"
cloud-fpga status        # should show all idle again
```

!!! tip "Avoid the prompt every time"
    If the SSH key has a passphrase, load it once with `ssh-add ~/.ssh/id_ed25519`
    so subsequent calls don't re-prompt. (On macOS, `ssh-add` can be finicky; if
    it refuses, you can always pass the key inline with `ssh -i`.)

!!! note "This affects every `with app:` run"
    Until the base-SoC reflash is fixed, any example that releases its session
    (which includes the shipped SAT and BERT examples) will leave its board in
    `error`. Expect to flush between runs.

## `python -m venv` fails with an `ensurepip` error

**Symptom.**

```text
Error: Command '[...python3.14, -m, ensurepip, --upgrade, --default-pip]'
returned non-zero exit status 1.
```

**Cause.** Python **3.14** (notably the Homebrew build) ships a broken
`ensurepip`, so venv creation fails.

**Fix.** Use Python **3.12 or 3.13** explicitly:

```bash
python3.13 -m venv .venv
source .venv/bin/activate
pip install -e ./sdk
```

## `submit` jobs queue but never program

**Symptom.** A submitted design sits in `queued`/`building` and never reaches
`reserved`; `run` returns `409 fpga_not_reserved`.

**Cause.** The board's **worker** or the **build toolchain** isn't running on
that orchestrator deployment. The build server needs the full Amaranth → Yosys →
nextpnr-ecp5 → ecppack toolchain and a connected host agent.

**Fix.** Use a deployment with workers + toolchain wired up, or check
`cloud-fpga logs <fpga_id> <job_id>` for the failure. See
[Architecture → Jobs](../concepts/architecture.md#jobs).

## `secret(...)` raises `ValueError` at startup

`cloud_fpga.secret("CLOUD_FPGA_API_KEY")` fails fast if the variable is unset.
Export it (any non-empty value works during the prototype, since auth isn't
enforced):

```bash
export CLOUD_FPGA_API_KEY=dev
```

## A build failed — where are the logs?

```bash
cloud-fpga logs <fpga_id> <job_id>
```

This prints the synthesizer output (Yosys / nextpnr-ecp5), which is where most
design rejections show up — unsupported constructs, resource overflows, or an
interface that doesn't match the Wishbone B4 slave contract.
