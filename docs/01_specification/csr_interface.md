# CSR Interface Specification

[← Back to Specification index](README.md) · [← Back to project README](../../README.md)

## Contents

1. [Purpose](#1-purpose)
2. [Register map overview](#2-register-map-overview)
3. [Command encoding](#3-command-encoding)
4. [Register definitions](#4-register-definitions)
5. [Transaction lifecycle](#5-transaction-lifecycle)
6. [Error handling](#6-error-handling)
7. [Software sequences](#7-software-sequences)
8. [Driver skeleton](#8-driver-skeleton)
9. [Stability and update policy](#9-stability-and-update-policy)
10. [References](#10-references)

---

## 1. Purpose

This document defines how the processor sees the SPI-Flash Controller:
which 32-bit registers exist, what each bit means, and what sequences
firmware must follow to read, write, or erase.

It is the **contract** between the RTL and the C driver. Once this
document is agreed, neither side changes it without a coordinated
revision (Section 9).

The register map implements the functional requirements defined in
[`requirements.md`](requirements.md): every RF related to bus access or
status is materialized here as a specific bit.

## 2. Register map overview

The controller occupies 64 KB at base address `0x420000`
(`0x420000–0x42FFFF`) and exposes **six** word-aligned registers. The
rest of the window reads as zero and ignores writes.

| Offset | Name | Access | Width | Purpose |
|---|---|---|---|---|
| `0x00` | `CTRL` | Write | 32 bits | Command selector and start bit |
| `0x04` | `STATUS` | Read | 32 bits | Busy, done, error, WIP, WEL |
| `0x08` | `ADDR` | Write | 32 bits (24 used) | 24-bit Flash address |
| `0x0C` | `DATA` | Read / Write | 32 bits (8 used) | Data FIFO port |
| `0x10` | `LEN` | Write | 32 bits (9 used) | Number of bytes (1–256) |
| `0x14` | `CFG` | Write | 32 bits | Clock divider, SPI mode |

The layout is illustrated in the diagram below, which shows the six
registers occupying the first 24 bytes of the window.

![Register map layout](../../diagrams/design/csr_map.svg)

*Figure 1 — Register map of the SPI-Flash Controller at base
`0x420000`. The six registers occupy offsets `0x00` to `0x14`; the rest
of the 64 KB window is reserved.*

## 3. Command encoding

The `CTRL.CMD` field selects one of the eleven supported commands
(see [`flash_commands.md`](flash_commands.md)). The encoding is chosen
to be simple to decode in RTL and simple to remember in C.

| `CMD` value | Mnemonic | Opcode | Command name |
|---|---|---|---|
| `0`  | `READ`         | `03h` | READ |
| `1`  | `FAST_READ`    | `0Bh` | FAST_READ |
| `2`  | `WREN`         | `06h` | WREN |
| `3`  | `WRDI`         | `04h` | WRDI |
| `4`  | `RDSR1`        | `05h` | RDSR1 |
| `5`  | `RDSR2`        | `35h` | RDSR2 |
| `6`  | `WRSR`         | `01h` | WRSR |
| `7`  | `PAGE_PROGRAM` | `02h` | PAGE_PROGRAM |
| `8`  | `SECTOR_ERASE` | `20h` | SECTOR_ERASE |
| `9`  | `BLOCK_ERASE`  | `D8h` | BLOCK_ERASE |
| `10` | `CHIP_ERASE`   | `C7h` | CHIP_ERASE |
| `11`–`15` | Reserved  | —     | (RTL rejects with `ERR`) |

Four bits are enough to hold the eleven values. The remaining five
values are reserved and are rejected by the RTL (Section 6).

## 4. Register definitions

### 4.1 `CTRL` — Control (offset `0x00`, write-only)

| Bit | Name | Meaning |
|---|---|---|
| `0` | `START` | Writing `1` starts the command encoded in `CMD[4:1]`. Automatically cleared by RTL. |
| `4:1` | `CMD` | 4-bit command selector (Section 3). |
| `31:5` | — | Reserved, must be written as `0`. |

Writing `START=0` does nothing. Writing `START=1` while `STATUS.BUSY=1`
is ignored (the RTL never starts two transactions at once).

### 4.2 `STATUS` — Status (offset `0x04`, read-only)

| Bit | Name | Meaning |
|---|---|---|
| `0` | `BUSY` | `1` while a transaction is in progress. Cleared when `DONE` or `ERR` is asserted. |
| `1` | `DONE` | `1` when the last transaction completed successfully. Cleared on read of `STATUS`. |
| `2` | `ERR` | `1` if the last transaction was aborted (see Section 6). Cleared on read of `STATUS`. |
| `3` | `WIP` | Mirror of the Flash `WIP` bit captured by the last RDSR1 or RDSR2 transaction. |
| `4` | `WEL` | Mirror of the Flash `WEL` bit captured by the last RDSR1 transaction. |
| `31:5` | — | Reserved, read as zero. |

**DONE and ERR are cleared on read.** This is the standard course
convention for status registers: a read of `STATUS` returns the current
flags and clears `DONE` and `ERR` so that the next transaction starts
from a clean state. `BUSY`, `WIP`, `WEL` are **not** cleared on read.

### 4.3 `ADDR` — Address (offset `0x08`, write-only)

| Bit | Name | Meaning |
|---|---|---|
| `23:0` | `ADDR` | 24-bit Flash byte address. The value is shifted MSB-first on `MOSI` as three bytes: `A23…A16`, then `A15…A8`, then `A7…A0`. |
| `31:24` | — | Reserved, must be written as `0`. Writes to these bits are ignored by RTL. |

The full 24 bits are transmitted for every command that needs an address.
The Flash internally ignores the low bits that are irrelevant for a
given command (see `flash_commands.md` for the per-command decoding).

### 4.4 `DATA` — Data FIFO port (offset `0x0C`, read/write)

`DATA` is the only register that connects to the FIFO logic. Its behavior
depends on the direction of the active command:

| Direction | Trigger | Behavior |
|---|---|---|
| Write | Any write to `DATA` | Pushes one byte (bits `7:0` of the written word) into the TX FIFO. Used for `PAGE_PROGRAM` and `WRSR`. |
| Read | Any read from `DATA` | Pops one byte from the RX FIFO into bits `7:0` of the returned word. Used for `READ`, `FAST_READ`, `RDSR1`, `RDSR2`. |

- Bits `31:8` are **ignored on write** and **read as zero**.
- FIFO depth is implementation-defined. The recommended minimum is **8
  bytes**, which is enough for the longest protocol byte count (24-bit
  address + 1 status byte).
- Reading `DATA` when the RX FIFO is empty returns `0x00000000` and sets
  `STATUS.ERR` (RF-29).
- Writing `DATA` when the TX FIFO is full is ignored and sets
  `STATUS.ERR` (RF-28).

### 4.5 `LEN` — Length (offset `0x10`, write-only)

| Bit | Name | Meaning |
|---|---|---|
| `8:0` | `LEN` | Number of data bytes to transfer. Valid range: **1 to 256**. Value `0` is treated as `256` (wrap) for `PAGE_PROGRAM` only. |
| `31:9` | — | Reserved, must be written as `0`. |

`LEN` is used only by `READ`, `FAST_READ`, `PAGE_PROGRAM` and `WRSR`
(where it must be exactly `1`). It is ignored by commands that carry no
data.

### 4.6 `CFG` — Configuration (offset `0x14`, write-only)

| Bit | Name | Meaning |
|---|---|---|
| `7:0` | `CLK_DIV` | SCLK divider: `f_SCLK = f_sys / (2 · (CLK_DIV + 1))`. Range `0` to `255`. |
| `8` | `CPOL` | Clock polarity. Fixed at `0` in this project (SPI Mode 0). Writes of `1` are ignored. |
| `9` | `CPHA` | Clock phase. Fixed at `0` in this project (SPI Mode 0). Writes of `1` are ignored. |
| `31:10` | — | Reserved, must be written as `0`. |

With a 50 MHz system clock and `CLK_DIV = 249`, `f_SCLK = 100 kHz`, a
safe bring-up frequency. With `CLK_DIV = 0`, `f_SCLK = 25 MHz`, the
maximum for `READ` at 3.3 V is 50 MHz, so `CLK_DIV = 0` is safe for
`READ` but the design should not exceed it for `READ`.

## 5. Transaction lifecycle

Every transaction follows the same high-level sequence regardless of
command. The firmware:

1. Writes any needed setup registers (`ADDR`, `LEN`, `DATA`).
2. Writes `CTRL` with `CMD` and `START=1`.
3. Polls `STATUS.BUSY` until it clears (or `DONE`/`ERR` is set).
4. Reads `DATA` for the returned bytes (if any).
5. Reads `STATUS` to clear `DONE`/`ERR`.

The RTL is responsible for the state machine below, which is the same for
all eleven commands. Only the number of bytes shifted in each phase and
the direction of the data differs.

![Transaction lifecycle FSM](../../diagrams/design/csr_txn_fsm.svg)

*Figure 2 — Transaction lifecycle. `IDLE` waits for a write to `CTRL`
with `START=1`. `LOAD` reads setup registers and preloads the first bit.
`ACTIVE` shifts opcode, address, dummy and data bytes according to the
command. `FINISH` deasserts `CS`, asserts `DONE` (or `ERR`), and returns
to `IDLE`.*

The per-command phases inside `ACTIVE` are documented in
[`flash_commands.md`](flash_commands.md). The FSM here is generic; the
phase counter and the direction of each byte are driven by the `CMD`
field.

## 6. Error handling

The RTL sets `STATUS.ERR` (and asserts `DONE` so firmware can proceed)
when one of the following occurs:

| Condition | How to detect | Recovery |
|---|---|---|
| Reserved `CMD` value (`11`–`15`) | `STATUS.ERR=1` immediately | Firmware bug; fix the `CTRL` write |
| `START` while `STATUS.BUSY=1` | Second `START` ignored, no error | Firmware bug; poll `BUSY` before start |
| Write to `DATA` when TX FIFO full | `STATUS.ERR=1` | Firmware bug; check `LEN` vs FIFO depth |
| Read from `DATA` when RX FIFO empty | `STATUS.ERR=1`, read returns `0x00000000` | Firmware bug; check `LEN` vs available bytes |
| `LEN=0` for a data command | Treated as `256` for `PAGE_PROGRAM`; treated as `ERR` for `READ`/`FAST_READ` | Firmware bug |

`ERR` is a **sticky** flag cleared only by a read of `STATUS`. This
allows firmware to distinguish "transaction finished" (`DONE=1`,
`ERR=0`) from "transaction aborted" (`DONE=1`, `ERR=1`).

The RTL **does not** detect Flash-level errors such as "Flash did not
respond to WIP polling". That is a firmware responsibility, using the
`WIP` mirror in `STATUS`.

## 7. Software sequences

The three sequence examples below are the canonical firmware patterns.
They are written in plain C, targeting a `volatile uint32_t *` base
pointer. Addresses are offsets from `0x420000`.

### 7.1 Read N bytes

    // 1. Configure the address
    mmio_write32(BASE + 0x08, addr & 0x00FFFFFF);

    // 2. Configure the length
    mmio_write32(BASE + 0x10, len);

    // 3. Start READ (CMD = 0)
    mmio_write32(BASE + 0x00, (0u << 1) | 1u);

    // 4. Wait for completion
    while (mmio_read32(BASE + 0x04) & 1u) { /* poll BUSY */ }

    // 5. Check for errors
    uint32_t st = mmio_read32(BASE + 0x04);  // clears DONE/ERR
    if (st & (1u << 2)) return -EIO;          // ERR
    if (!(st & (1u << 1))) return -EIO;       // DONE not set

    // 6. Read LEN bytes from DATA
    for (uint32_t i = 0; i < len; i++)
        buf[i] = (uint8_t)(mmio_read32(BASE + 0x0C) & 0xFF);

### 7.2 Program a page

    // 1. WREN (CMD = 2)
    mmio_write32(BASE + 0x00, (2u << 1) | 1u);
    while (mmio_read32(BASE + 0x04) & 1u) {}
    if (mmio_read32(BASE + 0x04) & (1u << 2)) return -EIO;

    // 2. Address and length
    mmio_write32(BASE + 0x08, addr & 0x00FFFFFF);
    mmio_write32(BASE + 0x10, len);   // 1..256

    // 3. Push data into TX FIFO
    for (uint32_t i = 0; i < len; i++)
        mmio_write32(BASE + 0x0C, buf[i] & 0xFF);

    // 4. Start PAGE_PROGRAM (CMD = 7)
    mmio_write32(BASE + 0x00, (7u << 1) | 1u);
    while (mmio_read32(BASE + 0x04) & 1u) {}
    if (mmio_read32(BASE + 0x04) & (1u << 2)) return -EIO;

    // 5. Poll WIP via RDSR1 until the Flash finishes
    while (1) {
        mmio_write32(BASE + 0x00, (4u << 1) | 1u);   // RDSR1
        while (mmio_read32(BASE + 0x04) & 1u) {}
        uint32_t st = mmio_read32(BASE + 0x04);
        if (st & (1u << 2)) return -EIO;
        if (!(st & (1u << 3))) break;                 // WIP clear
    }

### 7.3 Erase a sector

    // 1. WREN
    mmio_write32(BASE + 0x00, (2u << 1) | 1u);
    while (mmio_read32(BASE + 0x04) & 1u) {}

    // 2. Address (only A[23:12] matter)
    mmio_write32(BASE + 0x08, addr & 0x00FFF000);

    // 3. Start SECTOR_ERASE (CMD = 8)
    mmio_write32(BASE + 0x00, (8u << 1) | 1u);
    while (mmio_read32(BASE + 0x04) & 1u) {}

    // 4. Poll WIP (up to 400 ms for sector erase)
    while (1) {
        mmio_write32(BASE + 0x00, (4u << 1) | 1u);
        while (mmio_read32(BASE + 0x04) & 1u) {}
        uint32_t st = mmio_read32(BASE + 0x04);
        if (!(st & (1u << 3))) break;
    }

## 8. Driver skeleton

A minimal driver header follows. It is a suggested starting point for
the firmware team (group K); it is not part of the specification
itself.

    #ifndef SPI_FLASH_H
    #define SPI_FLASH_H

    #include <stdint.h>

    #define SPI_FLASH_BASE  0x00420000u

    #define REG_CTRL        0x00u
    #define REG_STATUS      0x04u
    #define REG_ADDR        0x08u
    #define REG_DATA        0x0Cu
    #define REG_LEN         0x10u
    #define REG_CFG         0x14u

    #define CMD_READ        0u
    #define CMD_FAST_READ   1u
    #define CMD_WREN        2u
    #define CMD_WRDI        3u
    #define CMD_RDSR1       4u
    #define CMD_RDSR2       5u
    #define CMD_WRSR        6u
    #define CMD_PAGE_PROG   7u
    #define CMD_SECTOR_ER   8u
    #define CMD_BLOCK_ER    9u
    #define CMD_CHIP_ER     10u

    #define STATUS_BUSY     (1u << 0)
    #define STATUS_DONE     (1u << 1)
    #define STATUS_ERR      (1u << 2)
    #define STATUS_WIP      (1u << 3)
    #define STATUS_WEL      (1u << 4)

    int     spi_flash_init(uint8_t clk_div);
    int     spi_flash_read(uint32_t addr, uint8_t *buf, uint32_t len);
    int     spi_flash_write(uint32_t addr, const uint8_t *buf, uint32_t len);
    int     spi_flash_erase_sector(uint32_t addr);
    int     spi_flash_erase_block(uint32_t addr);
    int     spi_flash_erase_chip(void);
    int     spi_flash_wait_wip(uint32_t timeout_ms);

    #endif

## 9. Stability and update policy

This document is the contract between RTL and firmware. Any of the
following changes is considered **breaking** and requires coordination:

- Adding, removing, or renaming a register.
- Changing the offset of an existing register.
- Adding or removing a bit in `CTRL`, `STATUS`, `ADDR`, `LEN`, `CFG`.
- Changing the encoding of `CMD`.
- Changing the clear-on-read behavior of `DONE`/`ERR`.

When a breaking change is necessary:

1. Update this document first.
2. Update `requirements.md` (Section 5 traceability) so the RF-to-CSR
   mapping stays accurate.
3. Notify group K (firmware) and the integration team before the RTL
   change is merged.

Non-breaking changes (adding a reserved bit, changing the FIFO depth
within the documented range, changing the default `CLK_DIV` value) do
not require coordination and can be made in a normal PR.

## 10. References

- [`requirements.md`](requirements.md) — the RFs that this document
  implements.
- [`flash_commands.md`](flash_commands.md) — per-command phases that the
  FSM in Section 5 must sequence.
- [`flash_memory.md`](flash_memory.md) — device timing, used to size the
  polling timeout.
- [`spi_protocol.md`](spi_protocol.md) — Mode 0 timing that the `CFG`
  register configures.
- Winbond W25Q128JV datasheet, §7 (Status Registers) and §8
  (Instructions).

---

*Last updated: 2026-10-XX*
*Authors: @camahechag-bars, @jcanizalez16, @edfrancos*