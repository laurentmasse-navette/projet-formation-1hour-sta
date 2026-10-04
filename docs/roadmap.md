# Roadmap: shortest path to a first timing report

## Milestone 1 goal

Read a Liberty library, a flat structural Verilog netlist, an SDF delay file, and a minimal SDC,
then produce a setup `report_timing` from a PrimeTime-like Tcl shell.

All delays come from SDF. No parasitic extraction, no delay calculation, no physical data.

## Why SDF first, and why no LEF/DEF/tech file

- SDF supplies every delay value the first report needs:
  - `IOPATH`: cell arc delays.
  - `INTERCONNECT`: net delays (driver to load).
  - `TIMINGCHECK` (`SETUP`, `HOLD`, `SETUPHOLD`): register constraint values.
- LEF, DEF, and technology files only feed parasitic extraction. A PrimeTime-style tool consumes
  interconnect data as SPEF produced by an external extractor, so LEF/DEF are out of scope for core STA,
  not just for this milestone.
- Refinement path: SDF annotation, then SPEF + Elmore + NLDM delay calculation, then Ceff / AWE / CCS / ECSM.

### Consequences and caveats

1. **Liberty is still required, in part.** SDF gives values, not graph structure. Liberty provides pin direction,
   arc existence and `related_pin`, `timing_sense` (unateness), `timing_type`, sequential cells and clock pins, and units.
   NLDM tables may be parsed and stored but are unused in milestone 1.
2. **No slew propagation.** Transitions do not affect annotated delays; reports omit or zero the transition column.
3. **Ideal clocks.** Propagated clocks (using SDF-annotated clock tree delays) are a later refinement.
4. **Max (setup) analysis only.** SDF `min:typ:max` triples: use max for setup. Hold with min values comes next.
5. **SDF/netlist consistency.** Instance paths, `DIVIDER`, and escaped names must match. Flat netlists only at first.
6. **Unannotated arcs.** Policy: zero delay plus a warning; an annotation coverage report comes later.
7. **Units.** SDF `TIMESCALE` and Liberty `time_unit` are normalized to one internal time unit.

## Module layout (proposal, finalized in step 2)

| Directory | Content |
|---|---|
| `dm/library` | Cells, pins, timing arcs, units |
| `dm/design` | Modules, instances, nets, pins, ports (flattened after linking) |
| `dm/constraints` | Clocks, input/output delays |
| `dm/delays` | Annotated arc delays (min/max, rise/fall) |
| `timing/` | Timing graph, levelization, propagation, path reporting (derived from `dm/`) |
| `io/<format>/` | Liberty, Verilog, SDC, SDF readers |
| `shell/` | Tcl interpreter embedding and command registration |

`dm/technology` and `dm/parasitics` are planned for later stages and not created yet.

## Steps

Legend: **Planning** = documents and decisions only. **Code** = code-generating, opened explicitly when scheduled.

| # | Step | Type | Depends on |
|---|---|---|---|
| 0 | Language and build | Planning | - |
| 1 | Test data and reference | Planning | - |
| 2 | Architecture: data model and modules | Planning | 0 |
| 3 | Tcl shell skeleton | Code | 0, 2 |
| 4 | Liberty reader | Code | 2, 3 |
| 5 | Structural Verilog reader | Code | 2, 3 |
| 6 | Linking | Code | 4, 5 |
| 7 | Object query commands | Code | 6 |
| 8 | SDC subset | Code | 7 |
| 9 | SDF reader | Code | 6 |
| 10 | Timing graph construction | Code | 6, 9 |
| 11 | Timing propagation (simplified) | Code | 8, 10 |
| 12 | `report_timing` (minimal) | Code | 11 |
| 13 | End-to-end validation | Code (tests) | 1, 12 |

Steps 4 and 5 can proceed in parallel once 2 and 3 are done. Step 9 can proceed in parallel with 7 and 8.

### 0. Language and build (Planning)

Decisions are recorded in [decisions/0000-language-and-build.md](decisions/0000-language-and-build.md).

- Accepted: Linux (AlmaLinux 8) under WSL2; C++20 with `gcc-toolset-14`, Clang 21 for CI and `clang-tidy`.
- Accepted: CMake presets with Ninja; GoogleTest via `FetchContent`; system Tcl 8.6 embedded through its C API.
- Accepted: hand-written parsers; exceptions internally, Tcl errors at the shell boundary, project message IDs.
- Coding conventions recorded as a Cursor rule.
- Open: version control (D0.9), continuous integration (D0.10), license (D0.11).

### 1. Test data and reference (Planning)

- Pick an open Liberty library (for example Nangate45 or sky130).
- Small flat designs with matching SDF and SDC files.
- Hand-computed expected slack for the smallest design.
- Optional black-box reference: open-source OpenSTA, used only as an external tool to generate SDF and compare
  reports. Its source code is not read or copied.

### 2. Architecture: data model and modules (Planning)

- Classes and ownership for library, design, constraints, delays.
- Timing graph representation (nodes = pins/ports, edges = cell and net arcs).
- Data flow between readers, linker, graph, and reports. Final directory layout.

### 3. Tcl shell skeleton (Code)

- Embedded interpreter, command registration framework.
- PrimeTime-style argument parsing (`-option value`), consistent error and message reporting.
- `source` and interactive mode.

### 4. Liberty reader, minimal subset (Code)

- Generic group / simple attribute / complex attribute parser.
- Interpreted: library units, `cell`, `pin` (`direction`, `clock`), `ff`, `timing` (`related_pin`, `timing_type`, `timing_sense`).
- Command: `read_lib`.

### 5. Structural Verilog reader, minimal subset (Code)

- `module`, port and wire declarations (scalars, simple buses), cell instances with named connections, simple `assign`.
- Command: `read_verilog`.

### 6. Linking (Code)

- Commands: `link_design`, `current_design`.
- Bind instances to library cells, check pins, flatten (or reject hierarchy at first), report unresolved references.

### 7. Object query commands (Code)

- Minimal `get_ports`, `get_pins`, `get_cells`, `get_nets`, `get_clocks`, `all_inputs`, `all_outputs`.
- Collections start as simple Tcl handles. Required by SDC (`create_clock ... [get_ports clk]`).

### 8. SDC subset (Code)

- `create_clock` (`-name`, `-period`, `-waveform`, port source).
- `set_input_delay`, `set_output_delay` (`-clock`, `-max`/`-min`, value, ports).
- `read_sdc` (sources the file in the SDC command context).
- Not needed while delays come from SDF: `set_load`, `set_driving_cell`, `set_input_transition`.

### 9. SDF reader, minimal subset (Code)

- Header: `TIMESCALE`, `DIVIDER`.
- `CELL` / `INSTANCE` with `DELAY ABSOLUTE`: `IOPATH` (including edge specifiers), `INTERCONNECT`.
- `TIMINGCHECK`: `SETUP`, `HOLD`, `SETUPHOLD`.
- Warn and skip at first: `COND`, `INCREMENT`, `PORT`, `DEVICE`, `PATHPULSE`.
- Command: `read_sdf`.

### 10. Timing graph construction (Code)

- Build nodes and edges from linked design and Liberty arcs.
- Levelization; combinational loop detection (error at first).
- Annotate edges from SDF; apply the unannotated-arc policy.

### 11. Timing propagation, simplified (Code)

- Ideal clocks; rise/fall arrival propagation with max delays and unateness.
- Startpoints: input ports with input delay, register clock pins (clock-to-Q).
- Endpoints: register data pins (setup check), output ports (output delay).
- Single clock, setup check, required times and slack.

### 12. `report_timing`, minimal (Code)

- Worst path per endpoint, `-max_paths`, `-nworst 1`.
- Point / incr / path table with launch and capture sections, data arrival, data required, slack.
- Format inspired by, not copied from, PrimeTime.

### 13. End-to-end validation (Code: tests)

- Golden reports for the step 1 designs, run as regression tests.

## Deferred (after milestone 1)

1. Hold analysis.
2. Multiple clocks, inter-clock paths, `set_clock_uncertainty`, `set_false_path`, `set_multicycle_path`.
3. Propagated clocks.
4. Hierarchical netlists.
5. `report_constraint`, `report_annotated_delay`.
6. SPEF reader + Elmore net delay + NLDM cell delay (replaces SDF as the delay source).
7. CCS / ECSM, Ceff, AWE.
