# Examples

Runnable apps that ship in the [`Cloud_FPGA`](https://github.com/Barnard-PL-Labs/Cloud_FPGA)
repo under `examples/`. Each is a complete [`cloud_fpga.App`](../guides/sdk.md)
you launch with `cloud-fpga run`.

<div class="grid cards" markdown>

-   :material-check-decagram: **[SAT solver](sat-solver.md)**

    The flagship workload — a brute-force boolean satisfiability solver in
    hardware. Includes a generalized DIMACS runner and a 30-variable variant.

-   :material-brain: **[BERT feed-forward](bert-ffn.md)**

    A real transformer encoder block whose feed-forward matmuls run on the FPGA
    via an int8 4×4 matmul core.

-   :material-hand-wave: **[Hello Wishbone](hello-wishbone.md)**

    The minimal smoke test — write a pattern to an echo RAM and read it back.

</div>

!!! tip "Before you run"
    Install the SDK, set `CLOUD_FPGA_API_KEY`, and confirm a board is `idle`
    (`cloud-fpga status`). See the [Quickstart](../guides/quickstart.md).

!!! warning "Boards land in `error` after a run"
    These examples release their session on exit, which currently strands the
    board in `error`. Recover with the Redis flush in
    [Troubleshooting](../guides/troubleshooting.md#fpga-stuck-in-error).
