# Hello Wishbone

The minimal smoke test — the "hello world" of the platform. The design exposes a
512-word (2 KB) **echo RAM** over Wishbone; the app writes a pattern and reads it
back, confirming the full path (build → program → write → read) works end to end.

Source: `examples/hello_wishbone/` in the
[`Cloud_FPGA`](https://github.com/Barnard-PL-Labs/Cloud_FPGA) repo.

## Run it

```bash
cloud-fpga run examples/hello_wishbone/client_sdk.py
```

```text
writing pattern ...
reading back ...
  [0] 0xdeadbeef → OK
  [1] 0xcafebabe → OK
  [2] 0x12345678 → OK
  [3] 0xabcdef01 → OK
```

## The whole app

It's about as small as a `cloud_fpga.App` gets:

```python
import cloud_fpga

class Regs(cloud_fpga.RegisterMap):
    ECHO = 0x0000          # 512-word echo RAM, byte address 0

app = cloud_fpga.App(
    "hello_wishbone",
    design="examples/hello_wishbone/design.py",
    fpga_id=0,
    registers=Regs,
    api_key=cloud_fpga.secret("CLOUD_FPGA_API_KEY"),
)

@app.local_entrypoint()
def main():
    with app:
        pattern = [0xDEADBEEF, 0xCAFEBABE, 0x12345678, 0xABCDEF01]
        for i, word in enumerate(pattern):
            app.write(Regs.ECHO + i * 4, word)
        for i, expected in enumerate(pattern):
            got = app.read(Regs.ECHO + i * 4)
            status = "OK" if got == expected else f"MISMATCH (got {got:#010x})"
            print(f"  [{i}] {expected:#010x} → {status}")
```

This is the best starting point for [writing your own app](../guides/sdk.md):
swap in your own design and register map, and replace `main()` with your logic.
