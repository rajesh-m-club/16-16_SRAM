# 16×16 Transistor-Level SRAM Design

A transistor-level implementation of a **256-bit (16×16) Static
Random-Access Memory (SRAM)** using 6T bit cells and custom CMOS
peripheral circuits. The design was created in **Electric VLSI 9.06**
using a **TSMC 0.18 µm BSIM3** device model and verified through
**LTspice XVII** simulations.

> **Project status:** Functional transistor-level schematic and routed
> full-memory layout are available. The full-memory layout is **not yet
> DRC-clean**, and post-layout RC-extracted simulation has not been
> completed. See [Known Limitations](#known-limitations).

## Features

-   16 rows × 16 columns of 6-transistor (6T) SRAM cells --- **256 bits
    total**.
-   Five external functional signals: `CLK`, `RW`, `ADDR`, `DIN`, and
    `DOUT` (plus `VDD` and `GND`).
-   Serial, MSB-first 8-bit address input through a single `ADDR` pin.
-   Separate row and column decoding: 4-to-16 row selection and 4-to-16
    column selection.
-   Differential bit lines (`BL/BLB`) and shared data lines (`DL/DLB`).
-   Precharge/equalisation, bidirectional transmission-gate column mux,
    NMOS write driver, cross-coupled sense latch, output latch, and
    tapered output buffer.
-   Transistor-level schematics and layouts for the leaf and peripheral
    blocks, plus a routed full-memory layout.
-   LTspice functional simulations using the TSMC 180 nm BSIM3 model.

## Architecture

The top-level `SRAM_16x16` cell integrates the controller/periphery,
power-on-reset circuit, bit-cell array, column multiplexer, and
precharge network.

``` text
                   +----------------------+
 CLK, RW, ADDR, -->| Input buffers         |
 DIN               | Serial address logic |
                   | Timing/control       |
                   +----------+-----------+
                              |
                     +--------+--------+
                     |                 |
                +----v----+       +----v----+
                | Row     |       | Column  |
                | decoder |       | decoder |
                +----+----+       +----+----+
                     |                 |
                Word lines        Column selects
                     |                 |
             +-------v-----------------v------+
             |       16 × 16 6T SRAM array    |
             |       256 stored bits          |
             +---------------+----------------+
                             |
                       BL/BLB pairs
                             |
                     +-------v-------+
                     | 16:1 column   |
                     | multiplexer   |
                     +-------+-------+
                             |
                          DL/DLB
                             |
              +--------------+--------------+
              |                             |
        +-----v------+               +------v------+
 DIN -> | Write      |               | Sense latch |
        | driver     |               +------+------+
        +------------+                      |
                                      +-----v------+
                                      | Output latch|
                                      +-----+------+
                                            |
                                      +-----v------+
                                      | Output      |
                                      | buffer      |
                                      +-----+------+
                                            |
                                           DOUT
```

### Main blocks

  -----------------------------------------------------------------------
  Block                               Function
  ----------------------------------- -----------------------------------
  `SRAM_6T`                           Stores one bit using cross-coupled
                                      CMOS inverters and two access
                                      transistors

  `SRAM_Row` / `bitarray`             Organises 16 cells per row and 16
                                      rows into a 16×16 array

  `addr_sr`                           Captures the serial address using a
                                      sentinel shift-register scheme;
                                      captures `RW` and `DIN`

  `rowdec`                            Decodes the upper four address bits
                                      to select one of 16 word lines

  `coldec`                            Decodes the lower four address bits
                                      to select one of 16 columns

  `precharge16`                       Precharges and equalises the
                                      bit-line and shared data-line pairs

  `colmux16`                          Connects the selected bit-line pair
                                      to shared `DL/DLB` using
                                      transmission gates

  `wdrv`                              Drives the selected data line low
                                      to write the required bit

  `sa_latch`                          Regenerates the small differential
                                      read signal to full logic levels

  `latch` / `outbuf`                  Holds the read result and drives
                                      the external output load

  `por`                               Generates the power-on reset signal
  -----------------------------------------------------------------------

## Interface and access protocol

  Signal         Direction   Description
  -------------- ----------- ---------------------------------------------
  `CLK`          Input       Clock; rising edges shift address bits
  `RW`           Input       Read/write control: `1` = write, `0` = read
  `ADDR`         Input       Serial address input, MSB first
  `DIN`          Input       Serially sampled write-data bit
  `DOUT`         Output      Read-data output; holds the last read value
  `VDD`, `GND`   Supply      1.8 V supply and ground

**One access takes nine clock cycles:**

1.  **Edges 0--7:** Shift the eight address bits `A7` through `A0` into
    the serial address register, MSB first.
2.  **Edge 8:** Capture `RW` and `DIN`; the sentinel indicates that the
    address is ready.
3.  **CLK-high phase:** Precharge and equalise the bit lines and shared
    data lines.
4.  **CLK-low access phase:** Enable the selected word line and column.
    A write uses `WREN`; a read develops a bit-line differential and
    then enables the sense amplifier.
5.  **Next frame:** The frame flag clears and the next serial address
    can be shifted in.

The row is selected by `A7:A4`; the column is selected by `A3:A0`. The
theoretical access throughput is `fCLK / 9`.

## Technology and design parameters

  Parameter                 Value
  ------------------------- -----------------------------
  Process model             TSMC 0.18 µm CMOS, BSIM3
  Model file                `02_TSMC_180nm/tsmc018.lib`
  Supply voltage            1.8 V
  Process corner            Typical (TT)
  Simulation temperature    27 °C
  Electric VLSI             9.06
  LTspice                   XVII
  Memory organisation       16 × 16 × 1 bit
  Bit-cell topology         6T SRAM
  Address width             8 bits, serial input
  Clock cycles per access   9

## Simulation and reported results

The transistor-level schematic was exported from Electric and simulated
in LTspice with the TSMC BSIM3 model. The test setup includes
pad/package parasitics and an output load.

  -----------------------------------------------------------------------
  Test                                Reported result
  ----------------------------------- -----------------------------------
  `SRAM_tb`                           3/3 reads correct for writes/reads
                                      at `0x13` and `0xA7`

  `SRAM_tb4`                          4/4 reads correct across `0x13`,
                                      `0xA7`, `0xFF`, and `0x68`

  Address sweep                       504/504 reads correct across 252 of
                                      256 simulated addresses

  Mean write energy                   Approximately 1.34 pJ (write 0) /
                                      1.42 pJ (write 1) per access

  Mean read energy                    Approximately 1.99 pJ per access

  Core maximum clock                  Approximately 700 MHz (reported
                                      core-only result)

  With 5 pF output load               Approximately 330 MHz (reported
                                      loaded result)
  -----------------------------------------------------------------------

These are **schematic-level, typical-corner simulation results**, not
silicon measurements. The address-sweep result covers 252 of the 256
addresses, not the complete address space.

## Layout and physical verification

-   Leaf and peripheral cells were laid out in a standard-cell-like
    arrangement with horizontal supply rails.
-   The full-memory layout assembles the 16-row bit-cell array,
    precharge row, column mux, and peripheral blocks, with
    Metal-2/Metal-3 routing and vertical power straps.
-   The document reports zero DRC errors for 41 individual new
    leaf/block cells.
-   NCC was reported as matching for 35 cell pairs.
-   The **full-memory layout reports approximately 1,209 DRC findings**,
    including real Metal-2/Metal-3 spacing violations. Full-layout NCC
    was not run.

Do not describe the full-chip layout as DRC-clean or signoff-ready.

## Running the simulation

The project documentation describes this flow for the Electric library:

1.  Open `SRAM_Complete_V3.jelib` in **Electric VLSI 9.06**.
2.  In **Preferences → Tools → Simulation (SPICE)**, set **Spice Header
    Card Info** to the model file `02_TSMC_180nm/tsmc018.lib`.
3.  Open the `SRAM_tb{sch}` test cell. Its SPICE code note contains the
    stimulus and simulation parameters.
4.  Select **Tools → Simulation (SPICE) → Write SPICE Deck** to export
    the deck.
5.  Open/run the generated `.spi` deck in **LTspice XVII** and inspect
    the read voltages and pass flags in the simulation log.

Documented testbench settings include `VDD = 1.8 V`, `TCLK = 5 ns`,
`TR = 0.1 ns`, `TSTART = 150 ns`, `CLOAD = 5 pF`, and `TSTEP = 50 ps`.

The exact exported deck filename and any local model-file path depend on
the working directory and are not hard-coded here.

## Repository contents

The design document identifies the Electric library as
`SRAM_Complete_V3.jelib` and describes the test cells `SRAM_tb` and
`SRAM_tb4`. A suggested repository organisation is:

``` text
.
├── README.md
├── SRAM_Complete_V3.jelib
├── 02_TSMC_180nm/
│   └── tsmc018.lib
├── simulation/
│   ├── SRAM_tb
│   ├── SRAM_tb4
│   └── exported_spice_decks/
└── docs/
    └── SRAM_16x16_Design_Document.pdf
```

Adjust this tree to match the files actually committed to the
repository. The test-cell names above are Electric library cells;
include separate files only if you have exported them.

## Known limitations

-   The power-on-reset (`por`) block has no layout; `RSTB` is exposed as
    a layout pin.
-   The full-memory layout is routed but not DRC-clean; approximately 80
    real metal-spacing violations are reported among the full set of
    findings.
-   Full-memory NCC was not run.
-   No post-layout RC-extracted simulation is reported.
-   Only the typical (TT) process corner was simulated; Monte Carlo and
    process-corner sweeps are not reported.
-   The bit cell is based on the V1 implementation and is not
    area-optimised.
-   The shared sense amplifier and write driver reduce peripheral
    duplication, but the architecture performs one-bit accesses.

## Future work

-   Resolve full-array Metal-2/Metal-3 spacing violations and rerun
    full-layout DRC.
-   Complete full-layout NCC and post-layout parasitic extraction.
-   Run simulations across process, voltage, and temperature corners.
-   Perform Monte Carlo analysis and robustness checks.
-   Optimise the 6T bit-cell layout and evaluate array/periphery area.
-   Characterise timing, read stability, write margin, and energy under
    extracted parasitics.

## References

-   `SRAM_16x16_Design_Document.pdf` --- design architecture,
    transistor-level implementation, layout, simulation results, and
    limitations.
-   TSMC 0.18 µm BSIM3 model: `02_TSMC_180nm/tsmc018.lib` (used by the
    simulation flow; follow the applicable model-file licensing and
    distribution terms).
