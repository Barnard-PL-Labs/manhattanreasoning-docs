# BERT feed-forward

An **architecturally faithful BERT encoder block** whose **feed-forward network
runs on the FPGA**. The pipeline is a real transformer:

```text
text → WordPiece tokenizer → embeddings → multi-head self-attention
     → FEED-FORWARD (on the FPGA) → residual + LayerNorm → block output
```

The feed-forward network is the classic BERT FFN — `GELU(x·W₁ + b₁)·W₂ + b₂` with
the 4× hidden expansion — and **its two matrix multiplications execute on the
FPGA** using an int8 4×4 matmul core. Everything else (attention, GELU,
LayerNorm, softmax) runs in float NumPy on the host.

Source: `examples/bert_ffn/` in the
[`Cloud_FPGA`](https://github.com/Barnard-PL-Labs/Cloud_FPGA) repo.

## How the FFN reaches the FPGA

1. **Quantize** activations `x` and weights `W` to per-tensor int8 (symmetric).
2. **Tile** the int matmul `[M,K]·[K,N]` into 4×4 blocks, accumulating partial
   products across K host-side in int64. All-zero tiles are skipped.
3. **Run each block on the FPGA**: burst-write the two operands, pulse `start`,
   poll `done`, burst-read the 4×4 result.
4. **Dequantize** with the product of the two scales.

Every 4×4 block is a network round trip, so the demo dimensions are kept tiny
(`d_model=8`, `d_ff=32`, `seq≤4`) — about 32 tiles / ~60 s. The tiling engine
handles arbitrary sizes; the dimensions are bounded by patience, not correctness.

## Run it

```bash
# hardware-free smoke test (NumPy backend, real tokenizer)
python examples/bert_ffn/client_sdk.py --sim --text "fpga runs bert"

# on hardware (needs: pip install numpy transformers, and CLOUD_FPGA_API_KEY set)
cloud-fpga run examples/bert_ffn/client_sdk.py
```

```text
  input    : 'fpga runs bert'
  tokens   : ['[CLS]', 'f', '##pg', '##a']
  config   : d_model=8  d_ff=32  heads=2  seq=4
  feed-forward backend: FPGA bert_ffn (fpga0)
  done: 32 tiles run, 0 skipped (zero), 63.8s
  ...
  FPGA vs host-int matmul : max abs diff = 0.000e+00 (EXACT MATCH)
  int8 FFN vs exact float : relative L2 error = 0.838%
```

## Verification

The run cross-checks the hardware three ways: the FPGA result must be
**bit-exact** against the same int matmul done on the host (proving the hardware
is correct), and the int8 FFN is compared against an exact-float forward to report
the quantization error (typically <1%).

## Contents

- `client_sdk.py` — the SDK app + `@local_entrypoint`. Reuses the `matrix_mult`
  design unchanged as the FPGA design.
- `model.py` — the BERT block in NumPy (embeddings, attention, FFN, LayerNorm)
  and int8 quantization helpers.
- `fpga_matmul.py` — the 4×4 tiling engine; FPGA and NumPy tile backends.
- `tests/unit/` — pure-Python checks of tiling, quantization, and the register map.

See `examples/bert_ffn/README.md` for full details.
