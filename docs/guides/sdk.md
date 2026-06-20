# The Python SDK

The `cloud_fpga` package wraps the full FPGA lifecycle — build, program, register
I/O, and release — behind a small [Modal](https://modal.com)-style decorator API.
It is the recommended way to use the platform; the [REST API](../api/rest.md) is
the lower-level contract underneath.

## Install

```bash
pip install -e ./sdk      # from a clone of the Cloud_FPGA repo
```

Requires **Python 3.12 or 3.13**. Installs the `cloud_fpga` package and the
[`cloud-fpga`](cli.md) CLI.

## The mental model

You describe an **application** with three things:

1. a **design** — an Amaranth `.py` file whose top-level module is a Wishbone B4
   slave (see [Architecture](../concepts/architecture.md));
2. a **register map** — the byte offsets your design exposes;
3. **API config** — which board (`fpga_id`) and your key.

Then you read and write registers. The first access programs the FPGA
automatically.

```python
import cloud_fpga

class Regs(cloud_fpga.RegisterMap):
    CTRL     = 0x0000
    DATA_IN  = 0x0004
    DATA_OUT = 0x0008

app = cloud_fpga.App(
    "my_design",
    design="design.py",
    fpga_id=0,
    registers=Regs,
    api_key=cloud_fpga.secret("CLOUD_FPGA_API_KEY"),
)

@app.local_entrypoint()
def main():
    with app:
        app.write(Regs.DATA_IN, 0x1234)
        app.write(Regs.CTRL, 1)            # kick off
        print(hex(app.read(Regs.DATA_OUT)))
```

Run it with the CLI (which calls the entrypoint after programming):

```bash
cloud-fpga run my_design.py
```

## `App`

```python
cloud_fpga.App(
    name,
    *,
    design,                 # path to the Amaranth .py design
    fpga_id,                # which board (0–9)
    registers=None,         # a RegisterMap subclass (optional)
    api_key=None,           # defaults to $CLOUD_FPGA_API_KEY
    api_url="https://api.manhattanreasoning.com",
)
```

Creating an `App` registers it so the CLI can discover it — you don't export
anything. If you define several in one file, `cloud-fpga run` uses the last one.

### `app.read(addr, count=1)`

Read `count` 32-bit words starting at byte address `addr`. Returns a single
`int` when `count == 1`, otherwise a `list[int]`. Programs the FPGA first if it
hasn't been programmed yet.

```python
status = app.read(Regs.CTRL)            # one word -> int
block  = app.read(0x0020, count=8)      # eight words -> list[int]
```

### `app.write(addr, value)`

Write one or more 32-bit words to byte address `addr`. `value` may be a single
`int` or a `list[int]` for a burst write. Programs the FPGA first if needed.

```python
app.write(Regs.DATA_IN, 0xDEADBEEF)     # single word
app.write(0x0020, [0] * 200)            # burst (e.g. clear a region)
```

### `app.release()`

Release the active session, returning the board to `idle`. Returns the reset
`job_id`.

```python
app.release()
```

### Context manager

`App` is a context manager that releases on exit — the idiomatic way to scope a
session:

```python
@app.local_entrypoint()
def main():
    with app:
        app.write(Regs.DATA_IN, 42)
        result = app.read(Regs.DATA_OUT)
    # session released here
```

!!! warning "Releasing currently strands the board in `error`"
    Releasing reflashes the base SoC, which currently fails and leaves the FPGA
    in `error`. Until that's fixed, recover with the Redis flush in
    [Troubleshooting](troubleshooting.md#fpga-stuck-in-error). You can also skip
    the auto-release by not using `with app:` and resetting manually later.

## `RegisterMap`

A marker base class for your design's address map. Subclass it and set integer
class attributes for the **byte** offsets:

```python
class Regs(cloud_fpga.RegisterMap):
    CTRL     = 0x0000      # word 0
    N_VARS   = 0x0004      # word 1
    MODEL    = 0x000C      # word 3
```

Byte offset = 4 × word offset. Passing `registers=Regs` to `App` is optional
(it's just for your own organization — `read`/`write` take raw addresses).

## `@app.local_entrypoint()`

Marks the function the CLI runs after programming. It is **not** called when you
`import` or `python` the file — only by `cloud-fpga run`. This lets the same file
double as an importable module and a runnable app.

## `secret(env_var)`

Reads a required environment variable, raising `ValueError` immediately (at
import time) if it's unset — so a missing key fails loudly before any work
starts.

```python
api_key = cloud_fpga.secret("CLOUD_FPGA_API_KEY")
```

## Module-level functions

For session management outside an `App`:

```python
cloud_fpga.get_session(fpga_id, api_key, api_url)      # current session info
cloud_fpga.release_session(fpga_id, api_key, api_url)  # release, returns job_id
```

## A complete example

The [SAT solver](../examples/sat-solver.md) example is a good template — a
`RegisterMap`, helper encoders, an `App`, and a `@local_entrypoint` that writes a
formula, starts the solve, polls for completion, and reads back the model.

## See also

- [CLI reference](cli.md) — driving apps from the terminal.
- [Examples](../examples/index.md) — runnable apps.
- [REST API](../api/rest.md) — what the SDK calls under the hood.
