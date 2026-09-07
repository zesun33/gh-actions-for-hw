# gh-actions-for-hw

> Reusable GitHub Actions for hardware CI — the `actions/setup-python` equivalent for RTL-to-GDS flows.

[![CI](https://github.com/zesun33/gh-actions-for-hw/actions/workflows/ci.yml/badge.svg)](https://github.com/zesun33/gh-actions-for-hw/actions/workflows/ci.yml)
[![License: Apache-2.0](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](./LICENSE)

Six composite actions backed by the published [`zesun33` EDA images](https://github.com/zesun33/eda-docker-images) (`ghcr.io/zesun33/...`), so CI runners pull in seconds instead of building toolchains. Every action is dogfooded by this repo's own `ci.yml`.

| Action | Engine | What it gates |
| :--- | :--- | :--- |
| `verilog-lint` | verible | Style/syntax diagnostics (fails the job) |
| `verilog-simulate` | iverilog + vvp | Testbench pass, `$fatal` triage, hang timeout |
| `yosys-synthesize` | Yosys | Gate mapping with `cells` output |
| `openroad-pnr` | OpenROAD + OpenSTA | Floorplan→route→STA with `wns` output and DEF-existence gate |
| `cocotb-test` | cocotb + icarus | Python regression with JUnit `results.xml` gate |
| `spice-simulate` | ngspice | Batch transient/OP simulation |

## Usage

```yaml
jobs:
  hw:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Lint
        uses: zesun33/gh-actions-for-hw/verilog-lint@v1
        with:
          files: rtl/counter.v

      - name: Simulate
        uses: zesun33/gh-actions-for-hw/verilog-simulate@v1
        with:
          files: rtl/counter.v tb/counter_tb.v
          top-module: counter_tb

      - name: Synthesize
        id: synth
        uses: zesun33/gh-actions-for-hw/yosys-synthesize@v1
        with:
          files: rtl/counter.v
          top-module: counter
          target: liberty
          liberty-file: pdk/NangateOpenCellLibrary_typical.lib
          output-netlist: counter_gate.v

      - name: Place and route
        id: pnr
        uses: zesun33/gh-actions-for-hw/openroad-pnr@v1
        with:
          netlist: counter_gate.v
          top-module: counter
          tech-lef: pdk/NangateOpenCellLibrary.tech.lef
          macro-lef: pdk/NangateOpenCellLibrary.macro.lef
          liberty-file: pdk/NangateOpenCellLibrary_typical.lib
```

## Notes and limits

- **All file inputs must live inside the workspace.** Only `github.workspace` is mounted into the containers; absolute host paths such as `/tmp/x.lib` are unreachable from the tools. Download PDK data into the workspace (e.g. `pdk/`) first.

- **P&R needs liberty-mapped netlists.** Generic (operator-expression) netlists are unreadable to OpenROAD; synthesize with `target: liberty` first. The action fails loudly (missing DEF) instead of reporting vacuous success.
- **PDK files are caller-supplied.** The actions ship no foundry data; [`mcp-openroad/platforms`](https://github.com/zesun33/mcp-openroad/tree/main/platforms) hosts the Nangate45 set this repo's CI pins for self-tests.
- **Cocotb Verilator path** needs Verilator ≥ 5.036 in the image; default `simulator: icarus` works everywhere.
- **Artifacts stay in the workspace** (`sim_build/`, `*.def`, `*.vcd`, `rc_out.txt`): gitignore them or upload via `actions/upload-artifact`.
- Pin to a release tag (`@v1`) in production; `@main` tracks development.

## License

Apache-2.0 © 2026 Md Zesun Ahmed Mia
