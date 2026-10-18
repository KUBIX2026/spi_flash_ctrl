# SPI-Flash Controller Peripheral — FemtoRV32 SoC

**Project:** FPGA Retro Video Game Console
**Course:** Digital Design I — 2026-2
**Institution:** Universidad Nacional de Colombia — Sede Bogotá
**Faculty:** Ingeniería — Departamento de Ingeniería Eléctrica y Electrónica
**Course Upstream Repository:** [digital_UN (Prof. Carlos Camargo)](https://github.com/cicamargoba/digital_UN)

## Development Team

- Jhon Dairon Canizalez Arias — [@jcanizalez16](https://github.com/jcanizalez16)
- Edwin Franco Sánchez — [@edfrancos](https://github.com/edfrancos)
- Carlos Alfonso Mahecha Gonzalez — [camahechag](https://github.com/camahechag)

## Quick Start

    git clone git@github.com:KUBIX2026/spi_flash_ctrl.git
    cd spi_flash_ctrl
    cd rtl
    make sim      # compile with iverilog and run the testbench
    make waves    # open GTKWave with the .gtkw file

> Requires `iverilog` and `gtkwave`. The `rtl/` directory is
> self-contained: it does not need the rest of the repository to
> simulate.

## 1. Project Overview

This repository contains the design, implementation, verification and
integration of the SPI-Flash Controller, a memory-mapped peripheral
that interfaces the FemtoRV32-based SoC with an external SPI NOR Flash
memory.

The controller implements the communication required to access the
Flash device through the Serial Peripheral Interface (SPI) and exposes
this functionality to the processor through a memory-mapped interface.

From the system perspective, the controller is an intermediate layer
between two different interfaces:

![System overview](diagrams/architecture/system_overview.svg)

*Figure 1 — The controller sits between the memory-mapped SoC and the
SPI slave.*

The project follows the course methodology — **flowchart → ASM → RTL →
hardware** — and each stage produces the artifact that is the input to
the next one:

![Methodology](diagrams/architecture/methodology.svg)

*Figure 2 — Top-down methodology. The six chapters under `docs/` mirror
the eight stages in the diagram.*

The repository therefore documents both the final implementation and
the engineering process used to obtain and verify it.

## 2. Scope

The scope of this project is the complete implementation of the
digital interface between the FemtoRV32-based SoC and the external SPI
Flash memory assigned to the project.

The scope of the design includes:

- specification of the interface between the processor and the
  peripheral;
- specification of the SPI Flash communication protocol required by
  the project;
- definition of the controller architecture;
- design of the control unit and datapath;
- definition of the memory-mapped registers used by the processor;
- implementation of the controller in synthesizable RTL;
- development of a simulation environment for functional verification;
- development of the corresponding firmware interface;
- integration with the SoC;
- validation of the peripheral on the FPGA hardware.

The FemtoRV32 processor is treated as a black-box component. Its
internal implementation is therefore outside the scope of this project;
the controller is designed according to the interfaces and system-level
requirements established by the course project.

The detailed requirements, architectural decisions, register
definitions, SPI transactions, RTL implementation and verification
procedures are developed in the subsequent sections of this
documentation.

## 3. Memory-Mapped Interface

| Attribute | Value |
|---|---|
| Base address | `0x420000` |
| Range | `0x420000 – 0x42FFFF` |
| Select (`cs`) | `cs[?]` — index to be confirmed with the integration team |
| Port contract | `d_in, cs, addr, rd, wr, d_out` |
| Data width | 32 bits |
| Clock domain | Single `clk`, async reset `rst` |

The full register map (six registers: `CTRL`, `STATUS`, `ADDR`, `DATA`,
`LEN`, `CFG`) is documented in
[`docs/01_specification/csr_interface.md`](docs/01_specification/csr_interface.md).

## 4. Documentation

Each documentation stage establishes the artifacts and decisions
required by the subsequent design stage. The initial stages establish
*what* the peripheral must do; subsequent chapters describe *how* those
requirements are transformed into an implementable architecture and
verified through simulation and hardware.

| Stage | Purpose |
|---|---|
| [01 — Specification](docs/01_specification/README.md) | Define what the peripheral must do |
| [02 — Architecture](docs/02_architecture.md) | Define how the required functionality is decomposed |
| [03 — Design](docs/03_design.md) | Transform the architecture into ASM, FSM, datapath and RTL |
| [04 — Verification](docs/04_verification.md) | Define and demonstrate how the design is verified |
| [05 — Integration](docs/05_integration.md) | Integrate the peripheral into the SoC and validate it on hardware |

Additional repository directories contain the implementation
artifacts:

| Directory | Contents |
|---|---|
| `rtl/` | Synthesizable RTL and RTL testbenches (`Makefile`, `waves/`) |
| `firmware/` | C driver (`driver/`), tests (`tests/`) and examples (`examples/`) |
| `diagrams/` | Own engineering diagrams (`architecture/`, `design/`, `protocol/`) |
| `references/spi_flash/` | Datasheet extracts (not own diagrams) — READ/WRITE/FAST_READ/RDSR waveforms |
| `planificacion/` | Project planning and task tracking |
| `docs/` | Book-style specification chapters 01–05 |

## 5. Planning & Tracking

| Item | Link |
|---|---|
| GitHub Organization | [KUBIX2026](https://github.com/KUBIX2026) |
| Team repository | [spi_flash_ctrl](https://github.com/KUBIX2026/spi_flash_ctrl) |
| Team planning & conventions | [`planificacion/README.md`](planificacion/README.md) |
| Issue template (shared repo) | [`tarea.yml`](https://github.com/cicamargoba/digital_UN/tree/main/template/.github/ISSUE_TEMPLATE) |
| Evaluation rubric | [`rubrica_evaluacion_proyecto.md`](https://github.com/cicamargoba/digital_UN/blob/main/rubrica_evaluacion_proyecto.md) |

## 6. Development Status

| Stage | Status | Target date |
|---|---|---|
| Requirements | ✅ Complete | 2026-10-XX |
| SPI Flash specification | ✅ Complete | 2026-10-XX |
| CSR specification | ✅ Complete | 2026-10-XX |
| Architecture | ⏳ Pending | 2026-11-XX |
| ASM | ⏳ Pending | 2026-11-XX |
| RTL | ⏳ Pending | 2026-11-XX |
| Verification | ⏳ Pending | 2026-11-XX |
| Firmware | ⏳ Pending | 2026-11-XX |
| SoC integration | ⏳ Pending | 2026-12-XX |
| FPGA validation | ⏳ Pending | 2026-12-XX |

---

*Last updated: 2026-10-XX*