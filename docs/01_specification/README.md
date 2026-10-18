# 01 — Specification

[← Back to project README](../../README.md)

This chapter defines **what** the SPI-Flash Controller must do. It is the
input to every later stage (architecture, design, verification,
integration). Everything in this chapter is **complete** and stable; any
change here is a breaking change that requires coordination with the
firmware and integration teams.

## Contents

| # | Document | Purpose | Status |
|---|---|---|---|
| 1 | [flash_memory.md](flash_memory.md) | Target device: class, reference part, organization, timing | ✅ Complete |
| 2 | [spi_protocol.md](spi_protocol.md) | SPI wire protocol, Mode 0, transaction template | ✅ Complete |
| 3 | [flash_commands.md](flash_commands.md) | The eleven supported commands with frames and waveforms | ✅ Complete |
| 4 | [requirements.md](requirements.md) | Functional (RF) and non-functional (RNF) requirements | ✅ Complete |
| 5 | [csr_interface.md](csr_interface.md) | Memory-mapped register map and C sequences | ✅ Complete |
| 6 | [operation_flow.md](operation_flow.md) | End-to-end READ / WRITE / ERASE sequences | ✅ Complete |

## Reading order

The chapter is designed to be read in numerical order. Each document
depends only on the ones before it:

1. **flash_memory.md** establishes the slave device.
2. **spi_protocol.md** establishes the vocabulary of the wire.
3. **flash_commands.md** establishes the eleven byte sequences.
4. **requirements.md** turns the above into traceable requirements.
5. **csr_interface.md** turns the requirements into a register map.
6. **operation_flow.md** ties commands and registers into user-level flows.

A reader who only has ten minutes should read **flash_memory.md** and
**flash_commands.md** — those two answer "what hardware are we talking
to, and with what words".

## Diagrams in this chapter

All diagrams are original SVG, kept in the shared `diagrams/` tree so
they can also be opened in draw.io.

| Figure | File | Used by |
|---|---|---|
| Memory hierarchy | [`diagrams/protocol/memory_hierarchy.svg`](../../diagrams/protocol/memory_hierarchy.svg) | flash_memory §4 |
| Address decomposition | [`diagrams/protocol/address_decomposition.svg`](../../diagrams/protocol/address_decomposition.svg) | flash_memory §5 |
| FPGA ↔ Flash connection | [`diagrams/protocol/connection_fpga_flash.svg`](../../diagrams/protocol/connection_fpga_flash.svg) | spi_protocol §8 |
| SPI Mode 0 timing | [`diagrams/protocol/spi_mode0_timing.svg`](../../diagrams/protocol/spi_mode0_timing.svg) | spi_protocol §6 |
| WREN waveform | [`diagrams/protocol/waveform_wren.svg`](../../diagrams/protocol/waveform_wren.svg) | flash_commands §3.1 |
| WRDI waveform | [`diagrams/protocol/waveform_wrdi.svg`](../../diagrams/protocol/waveform_wrdi.svg) | flash_commands §3.2 |
| RDSR1 waveform | [`diagrams/protocol/waveform_rdsr1.svg`](../../diagrams/protocol/waveform_rdsr1.svg) | flash_commands §3.3 |
| RDSR2 waveform | [`diagrams/protocol/waveform_rdsr2.svg`](../../diagrams/protocol/waveform_rdsr2.svg) | flash_commands §3.4 |
| WRSR waveform | [`diagrams/protocol/waveform_wrsr.svg`](../../diagrams/protocol/waveform_wrsr.svg) | flash_commands §3.5 |
| READ waveform | [`diagrams/protocol/waveform_read.svg`](../../diagrams/protocol/waveform_read.svg) | flash_commands §4.1 |
| FAST_READ waveform | [`diagrams/protocol/waveform_fast_read.svg`](../../diagrams/protocol/waveform_fast_read.svg) | flash_commands §4.2 |
| PAGE_PROGRAM waveform | [`diagrams/protocol/waveform_page_program.svg`](../../diagrams/protocol/waveform_page_program.svg) | flash_commands §5.1 |
| SECTOR_ERASE waveform | [`diagrams/protocol/waveform_sector_erase.svg`](../../diagrams/protocol/waveform_sector_erase.svg) | flash_commands §5.2 |
| BLOCK_ERASE waveform | [`diagrams/protocol/waveform_block_erase.svg`](../../diagrams/protocol/waveform_block_erase.svg) | flash_commands §5.3 |
| CHIP_ERASE waveform | [`diagrams/protocol/waveform_chip_erase.svg`](../../diagrams/protocol/waveform_chip_erase.svg) | flash_commands §5.4 |
| System context | [`diagrams/architecture/system_context.svg`](../../diagrams/architecture/system_context.svg) | requirements §2 |
| Register map | [`diagrams/design/csr_map.svg`](../../diagrams/design/csr_map.svg) | csr_interface §2 |
| Transaction lifecycle FSM | [`diagrams/design/csr_txn_fsm.svg`](../../diagrams/design/csr_txn_fsm.svg) | csr_interface §5 |
| READ flow | [`diagrams/design/flow_read.svg`](../../diagrams/design/flow_read.svg) | operation_flow §3 |
| WRITE flow | [`diagrams/design/flow_write.svg`](../../diagrams/design/flow_write.svg) | operation_flow §4 |
| ERASE flow | [`diagrams/design/flow_erase.svg`](../../diagrams/design/flow_erase.svg) | operation_flow §5 |

## Checkpoint status

This chapter satisfies:

- Flowchart of operation: ✅ (operation_flow.md §3–§5)
- CSR specification: ✅ (csr_interface.md)
- Requirements: ✅ (requirements.md)
- Per-command frames and waveforms: ✅ (flash_commands.md)
- Traceability to verification: partially specified — the mapping from
  RF to test lives in `requirements.md` §5 and will be filled with
  concrete test names in [`../04_verification.md`](../04_verification.md).

## What is *not* in this chapter

- Architecture (block decomposition, FSM details) — see
  [`../02_architecture.md`](../02_architecture.md).
- RTL and synthesis — see [`../03_design.md`](../03_design.md).
- Testbenches and simulation evidence — see
  [`../04_verification.md`](../04_verification.md).
- SoC integration and hardware validation — see
  [`../05_integration.md`](../05_integration.md).

## Next chapter

→ [02 — Architecture](../02_architecture.md)
