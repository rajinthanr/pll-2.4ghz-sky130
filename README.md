# PLL2_4

Schematic design and simulation of a charge-pump phase-locked loop targeting 2.4 GHz, built with the
open-source analog flow (xschem + ngspice) on the SkyWater SKY130 process. This is a university team
project in progress: the phase-frequency detector, charge pump and loop filter are drawn and simulated;
the VCO and the feedback divider are not in the repository yet.

## Architecture

```
            +-------+  UP  +--------+       +-------------+  Vctrl  +-------+
F_REF ----->|       |----->| Charge |------>| Loop filter |-------->|  VCO  |----+--> F_OUT
            |  PFD  |  DN  |  pump  |       +-------------+         +-------+    |
F_VCO --+-->|       |----->|        |                                             |
        |   +-------+      +--------+                               +---------+   |
        +-----------------------------------------------------------| Divider |<--+
                                                                    +---------+
                     (in repo: PFD, charge pump, loop filter; not yet: VCO, divider)
```

| Block | Files | Description |
|---|---|---|
| NAND2, NAND3 | `src/NAND2.sch`, `src/NAND3.sch` | Static CMOS gates from `sky130_fd_pr` 1.8 V nfet/pfet devices |
| SR latch | `src/SR_NAND2.sch` | NAND-based SR latch with active-low reset (`RST_N`) |
| D flip-flop | `src/Dff.sch` | Edge-triggered DFF with reset, built from the SR latches and NAND3 gates above |
| PFD (custom) | `src/PFD_Dff.sch`, `src/PFD.sch` | Two-flip-flop PFD from the custom DFF and NAND2; `PFD.sch` is a separate transistor-level PFD (18 devices) |
| PFD (standard cells) | `src/PFD_std.sch` | Same topology from `sky130_fd_sc_hd` cells: two `dfrtp_1` flip-flops, a `nand2_4` reset gate and `inv_8` output buffers on `UP`/`DN` |
| Charge pump | `git_11/CHRG_PUMP.sch` | Six-transistor pump: mirrored PMOS/NMOS current sources biased from `I_ref`, switched by `up` and `down` |
| Loop filter | `git_11/LOOP_FILTER.sch` | Passive RC filter, R1 = 1 kΩ, R2 = 100 kΩ, with PMOS devices used as MOS capacitors to `VN` |

## Status

Taken from the commit history:

- **Logic gates, SR latch, D flip-flop:** tested ("NAND is working", "SR flip flop working", "NAND3 tested", "D flip flop tested").
- **PFD:** complete. First built from the custom DFF, then redesigned with standard cells for better performance ("PFD designed using standard cells - performance improvement", "PFD done"). The standard-cell version is the one used downstream.
- **Charge pump:** schematic and a PFD + charge pump + loop filter testbench exist; transistor sizes are still the 1 µm / 0.15 µm defaults.
- **Loop filter:** corrected in the latest commits ("loop filter correct").
- **VCO, divider, top-level PLL:** not started in this repository.
- **Layout:** `spice/and2.gds` places the four standard cells of an earlier `PFD_std` revision (two `dfrtp_1`, `nand2_2`, `nand2_8`) with no routing yet.

## Testbenches

All run transient analysis in ngspice with the SKY130 `tt` corner at VDD = 1.8 V.

| Testbench | What it simulates |
|---|---|
| `tb/tb_NAND2.sch`, `tb/tb_NAND3.sch` | Truth table with pulse inputs, 200 ns |
| `tb/tb_SR_NAND2.sch`, `tb/tb_Dff.sch` | Latch and flip-flop response to pulse inputs, 200 ns |
| `tb/tb_PFD_std.sch` | 10 MHz `F_REF` and `F_VCO` with a 15 ns offset; 10 runs with VDD (1.8 V, σ = 50 mV) and temperature (20–80 °C) drawn at random |
| `tb/tb_CHARGE_PUMP.sch` | `PFD_std` → charge pump → loop filter chain, 10 µs |
| `tb/tb_LOOP_FILTER.sch` | Loop filter driven by a 10 MHz 0–1.8 V pulse train |

The schematic-level sub-circuits `src/PFD.sch` and `src/PFD_Dff.sch` carry their own stimulus
(50 MHz vs 52.6 MHz, and 10 MHz clocks respectively).

## Tools and PDK

- [xschem](https://xschem.sourceforge.io/) 3.4.8 for schematics and symbols
- [ngspice](https://ngspice.sourceforge.io/) for simulation
- SkyWater SKY130 (`sky130A`): `sky130_fd_pr` devices and the `sky130_fd_sc_hd` standard-cell library, installed with `ciel`
- [IIC-OSIC-TOOLS](https://github.com/iic-jku/IIC-OSIC-TOOLS) container: the schematics reference `/foss/designs/PLL2_4/...` and `/foss/pdks/...`

## Repository layout

```
PLL2_4/
├── src/            Design schematics (.sch) and symbols (.sym): gates, latch, DFF, PFD variants
├── tb/             Testbench schematics (untitled-5.sch is a scratch loop-filter testbench)
├── git_11/         Charge pump and loop filter schematics and symbols
│                   (CHRG_PUMP_TB.sch is an older testbench that still points at IHP SG13G2 models)
├── spice/
│   ├── and2.spice  Exported netlist of an earlier PFD_std revision with its testbench
│   ├── and2.gds    Standard-cell placement of that PFD (unrouted)
│   └── and2        Netlist of a single sky130_fd_sc_hd__and2b_4 cell test
└── LICENSE
```

## Opening and simulating

Symbol paths are absolute, so clone the repository into the IIC-OSIC-TOOLS designs folder:

```sh
cd /foss/designs
git clone https://github.com/rajinthanr/PLL2_4.git
cd PLL2_4
xschem tb/tb_PFD_std.sch
```

In xschem, press **Netlist** and then **Simulate**; the testbenches with a *load waves* launcher write a
`.raw` file that can be shown in the embedded graphs. The exported netlist runs directly:

```sh
ngspice spice/and2.spice
```

## Team

- Rajinthan Rameshkumar ([@rajinthanr](https://github.com/rajinthanr)): logic gates, flip-flops, PFD, and the testbenches including the charge pump and loop filter ones (branch `rajinthan`)
- [@Vidurangajka](https://github.com/Vidurangajka): charge pump and loop filter schematics (branch `anjana`)

`main` holds both branches merged.

## Licence

MIT License, © 2025 Rajinthan Rameshkumar. See [LICENSE](LICENSE).
