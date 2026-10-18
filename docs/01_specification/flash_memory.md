# Target Flash Device

[← Back to Specification index](README.md) · [← Back to project README](../../README.md)

## Contents

1. [Why this document exists](#1-why-this-document-exists)
2. [The device class: 128 Mbit SPI NOR Flash](#2-the-device-class-128-mbit-spi-nor-flash)
3. [Reference device: W25Q128JV](#3-reference-device-w25q128jv)
4. [Memory organization](#4-memory-organization)
5. [Address decomposition](#5-address-decomposition)
6. [Pinout and signals](#6-pinout-and-signals)
7. [Timing parameters](#7-timing-parameters)
8. [Device identification](#8-device-identification)
9. [What the controller depends on](#9-what-the-controller-depends-on)

---

## 1. Why this document exists

The SPI-Flash controller has to talk to a real chip. That chip has an internal
structure, a fixed pinout, and timing constraints that the controller must
respect. Without knowing the target device, it is impossible to decide the
address width, the transaction length, the polling strategy, or the maximum
SCLK frequency.

This document defines the **target device** that the controller is designed
against. It is the input to the command set (`flash_commands.md`), the
requirements (`requirements.md`), and the CSR interface (`csr_interface.md`).

## 2. The device class: 128 Mbit SPI NOR Flash

The KUBIX project stores game data (sprites, level data, high-score tables)
in an external SPI NOR Flash. The course does not mandate a specific part
number; it only requires:

| Constraint | Reason |
|---|---|
| SPI-compatible interface (Standard SPI, 4 signals) | Same bus used by the course |
| ≥ 16 MiB of storage | Holds game binary and assets |
| 4 KB sector erase granularity | Allows in-place updates of small blocks |
| Single-supply 3.3 V operation | Matches the FPGA I/O voltage |

These constraints define the **device class**: a **128 Mbit (16 MiB) SPI NOR
Flash with 4 KB sectors**. Any part in this class is compatible with the
controller.

> **Note on the reference device.** This documentation uses the **Winbond
> W25Q128JV** as the reference device because (a) it is a widely available
> JEDEC-standard part, (b) its public datasheet covers all timings needed by
> the controller, and (c) its internal organization (Section 4) matches the
> 16 MiB / 4 KB-sector requirement exactly. If the final hardware uses a
> different part in the same class, only the timing table (Section 7) and
> the JEDEC ID (Section 8) change; the RTL, the CSR interface, and the
> command set remain unchanged.

## 3. Reference device: W25Q128JV

| Parameter | Value |
|---|---|
| Density | 128 Mbit (16 MiB) |
| Organization | 256 blocks × 64 KB; 4096 sectors × 4 KB; 65,536 pages × 256 B |
| Supply voltage | 2.7 V – 3.6 V |
| Max clock frequency | 133 MHz (Fast Read, 3.0–3.6 V) |
| Interface | SPI (Single, Dual, Quad) |
| Programming cycles | 100,000 erase/program cycles |
| Data retention | > 20 years |

## 4. Memory organization

Understanding how the Flash memory is internally divided is essential to
understand the address that is sent in every command. A 24-bit address is
not a flat pointer: it is decoded hierarchically by the device.

The memory is organized into **four nesting levels**:

| Level | Count | Size | Selected by |
|---|---|---|---|
| Block | 256 | 64 KB | `A[23:16]` |
| Sector | 16 per block | 4 KB | `A[15:12]` |
| Page | 16 per sector | 256 B | `A[11:8]` |
| Byte | 256 per page | 1 B | `A[7:0]` |

![Memory hierarchy](../../diagrams/protocol/memory_hierarchy.svg)

*Figure 1 — Hierarchical organization of a 128 Mbit SPI NOR Flash. Each level
divides the previous one by a fixed power of two, which is why the address
bits split cleanly into the four fields shown in Section 5.*

The division is not arbitrary: **erase** operates on whole sectors (4 KB) or
blocks (64 KB), **program** operates on at most one page (256 B), and **read**
can stream across page boundaries without interruption. The controller must
be aware of these boundaries to avoid wrapping or over-erasing (Section 5).

## 5. Address decomposition

The 24-bit address `A[23:0]` sent over SPI is a single continuous bit stream,
but it is decoded into four fields by the device:

![Address decomposition](../../diagrams/protocol/address_decomposition.svg)

*Figure 2 — The 24-bit Flash address is decoded into block, sector, page and
byte-offset fields. The 256 × 16 × 16 × 256 product gives the total capacity:
16,777,216 bytes = 16 MiB.*

Three consequences matter for the controller:

1. **Page boundary.** `A[7:0]` is the offset inside a 256-byte page. A
   `PAGE_PROGRAM` that crosses a page boundary wraps around to the start of
   the same page, overwriting the data already written. Firmware must never
   issue a write that crosses a page boundary; the driver must split it.
2. **Sector boundary.** `SECTOR_ERASE (0x20)` erases the whole 4 KB sector
   that contains the address, regardless of `A[11:0]`. Firmware must be
   aware that the effective erase granularity is 4 KB.
3. **Block boundary.** `BLOCK_ERASE (0xD8)` erases the whole 64 KB block.

## 6. Pinout and signals

The reference device is available in an 8-pin SOIC-208 package. Only four of
the eight pins are used by Standard SPI:

| Pin | Name | Direction | Function |
|---|---|---|---|
| 1 | `/CS` | FPGA → Flash | Chip Select, active low |
| 2 | `DO` (IO1) | Flash → FPGA | Data Out (MISO) |
| 3 | `/WP` (IO2) | — | Write Protect (tied high in this project) |
| 4 | `GND` | — | Ground |
| 5 | `DI` (IO0) | FPGA → Flash | Data In (MOSI) |
| 6 | `CLK` | FPGA → Flash | Serial Clock (SCLK) |
| 7 | `/HOLD` (IO3) | — | Hold (tied high in this project) |
| 8 | `VCC` | — | 3.3 V supply |

![Physical connection](../../diagrams/protocol/connection_fpga_flash.svg)

*Figure 3 — Physical connection between the FPGA (SPI master) and the Flash
(SPI slave). Only SCLK, MOSI, MISO and CS carry data; `/WP` and `/HOLD` are
tied to VCC to disable the corresponding features.*

## 7. Timing parameters

These values bound the controller design. They come from the reference
datasheet (AC Electrical Characteristics, §9.6) and are used in the CSR
timeout/watchdog (`csr_interface.md`) and in verification
(`04_verification.md`).

| Symbol | Parameter | Min | Typ | Max | Unit |
|---|---|---|---|---|---|
| `f_R` | Read clock frequency (03h) | DC | — | 50 | MHz |
| `f_R` (Fast Read) | Fast Read clock frequency (0Bh) | DC | — | 133 | MHz |
| `t_PP` | Page Program time | — | 0.7 | 3 | ms |
| `t_SE` | Sector Erase time (4 KB) | — | 45 | 400 | ms |
| `t_BE` | Block Erase time (64 KB) | — | 150 | 2000 | ms |
| `t_CE` | Chip Erase time | — | 20 | 100 | s |
| `t_SHSL2` | CS deselect time (after erase/program) | 50 | — | — | ns |
| `t_PUW` | Power-up delay before write | 5 | — | — | ms |

> **Design implication.** The maximum erase time (2000 ms for a 64 KB block,
> 100 s for chip erase) is far longer than any reasonable polling timeout.
> The CSR must expose a timeout register, and firmware must implement a
> retry-or-abort policy. This is captured in `requirements.md` (RNF-…).

## 8. Device identification

The controller can verify that it is talking to the expected device by
reading the **JEDEC ID** (`9Fh` command, datasheet §8.2.27):

| Byte | Value | Meaning |
|---|---|---|
| Manufacturer ID | `EFh` | Winbond |
| Memory Type | `40h` | SPI NOR, 3.3 V |
| Capacity | `18h` | 128 Mbit |

The ID is fixed per part number; it is documented here so that the
verification plan (`04_verification.md`) can include an identity check at
startup.

> **Note.** Because the course does not mandate a specific part, the ID
> check is **recommended** in verification but **not required** in
> production firmware. If the final hardware uses a different part, the
> expected ID changes accordingly; the controller itself does not depend on
> the ID value.

## 9. What the controller depends on

This section ties the memory description back to the RTL and CSR design. It
is the traceability anchor for `requirements.md`.

| Memory property | Controller decision | Where |
|---|---|---|
| 24-bit address | `ADDR` register is 24 bits wide | `csr_interface.md` |
| 256-byte page | `LEN` register max = 256 | `csr_interface.md` |
| 4 KB sector | `SECTOR_ERASE` granularity | `flash_commands.md` |
| Status register WIP | Polling loop after program/erase | `csr_interface.md` |
| Max `f_R` = 50 MHz | SCLK divider must reach ≤ 50 MHz | `requirements.md` RNF-… |
| Erase time up to 2 s | Timeout register required | `requirements.md` RNF-… |
| JEDEC ID `EFh 40h 18h` | Optional startup identity check | `04_verification.md` |

---

*Last updated: 2026-10-XX*
*Authors: @camahechag-bars, @jcanizalez16, @edfrancos*