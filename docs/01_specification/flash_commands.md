# Flash Command Set

[← Back to Specification index](README.md) · [← Back to project README](../../README.md)

## Contents

1. [Why this document exists](#1-why-this-document-exists)
2. [Command summary](#2-command-summary)
3. [State and control commands](#3-state-and-control-commands)
4. [Read commands](#4-read-commands)
5. [Write and erase commands](#5-write-and-erase-commands)
6. [Cross-cutting design implications](#6-cross-cutting-design-implications)
7. [Commands explicitly out of scope](#7-commands-explicitly-out-of-scope)
8. [References](#8-references)

---

## 1. Why this document exists

The [SPI protocol chapter](spi_protocol.md) explains *how* bits travel on the
wire. This document answers a different question: *what sequence of bytes
does each command require, and what does the Flash send back?*

Each command is described with the same four elements so that a reader can
compare them side-by-side:

1. **Purpose** — what the command does, in one sentence.
2. **Wire sequence** — numbered steps from `CS` low to `CS` high.
3. **Frame table** — which bytes travel on `MOSI` and which on `MISO`.
4. **Waveform** — an engineering figure showing `CS`, `CLK`, `MOSI` and
   `MISO` together, with the actual bit values annotated.

The waveform figures are original to this project. Each one also cites the
equivalent figure in the reference datasheet
([`references/spi_flash/`](../../references/spi_flash/)) so the reader can
cross-check against the manufacturer's timing.

## 2. Command summary

The controller implements eleven commands, grouped into three families
that mirror the datasheet's own organization.

| Opcode | Name | Family | Bytes after opcode | Dummy | Data direction |
|---|---|---|---|---|---|
| `06h` | WREN | State / control | 0 | 0 | — |
| `04h` | WRDI | State / control | 0 | 0 | — |
| `05h` | RDSR1 | State / control | 0 | 0 | MISO (1 byte, repeats) |
| `35h` | RDSR2 | State / control | 0 | 0 | MISO (1 byte, repeats) |
| `01h` | WRSR | State / control | 0 | 0 | MOSI (1 byte) |
| `03h` | READ | Read | 3 (address) | 0 | MISO (N bytes) |
| `0Bh` | FAST_READ | Read | 3 (address) | 1 | MISO (N bytes) |
| `02h` | PAGE_PROGRAM | Write / erase | 3 (address) | 0 | MOSI (1–256 bytes) |
| `20h` | SECTOR_ERASE | Write / erase | 3 (address) | 0 | — |
| `D8h` | BLOCK_ERASE | Write / erase | 3 (address) | 0 | — |
| `C7h` | CHIP_ERASE | Write / erase | 0 | 0 | — |

**MSB-first rule.** Every byte in every command is shifted with the most
significant bit first (`b7` → `b0`). This is a JEDEC SPI convention and
matches the datasheet, §8.1.

## 3. State and control commands

These commands manage the 8-bit **status register** inside the Flash,
which reports device state and holds configuration bits.

### 3.1 Write Enable — WREN (06h)

**Purpose.** Set the **Write Enable Latch (WEL)** bit inside the Flash to
`1`. Any subsequent `PAGE_PROGRAM`, `SECTOR_ERASE`, `BLOCK_ERASE`,
`CHIP_ERASE` or `WRSR` will be ignored unless `WEL = 1`. WREN is therefore
the mandatory first step of every write or erase sequence.

**Wire sequence.**

1. `CS` goes low.
2. Master shifts the byte `06h` on `MOSI` (MSB first).
3. `CS` goes high.

The Flash drives `MISO` to high-impedance during the whole command; nothing
is returned.

**Frame table.**

| Phase | `MOSI` (FPGA → Flash) | `MISO` (Flash → FPGA) |
|---|---|---|
| Opcode | `06h` | High-Z (not driven) |

**Waveform.**

![WREN (06h) waveform](../../diagrams/protocol/waveform_wren.svg)

*Figure 1 — WREN (06h). `CS` frames the transaction, `CLK` idles low
(Mode 0), `MOSI` carries `06h` = `0000 0110`, and `MISO` is high-Z.*

**Datasheet reference.** §8.2.1, Figure 5.

**Design note.** Single-byte transaction, no return data. The CSR can
treat it as "set `CMD=WREN`, set `START`, wait for `DONE`". No FIFO.

### 3.2 Write Disable — WRDI (04h)

**Purpose.** Clear the WEL bit to `0`. After a WRDI, the Flash rejects any
write or erase command until the next WREN. Not required after a normal
`PAGE_PROGRAM` (the Flash clears WEL itself), but useful as a safety step
after an error path.

**Wire sequence.**

1. `CS` goes low.
2. Master shifts `04h` on `MOSI`.
3. `CS` goes high.

**Frame table.**

| Phase | `MOSI` (FPGA → Flash) | `MISO` (Flash → FPGA) |
|---|---|---|
| Opcode | `04h` | High-Z (not driven) |

**Waveform.**

![WRDI (04h) waveform](../../diagrams/protocol/waveform_wrdi.svg)

*Figure 2 — WRDI (04h). Identical envelope to WREN; opcode `04h` =
`0000 0100`.*

**Datasheet reference.** §8.2.3, Figure 7.

**Design note.** Same structure as WREN. Reuses the single-byte
transaction FSM with a different opcode.

### 3.3 Read Status Register 1 — RDSR1 (05h)

**Purpose.** Read the 8-bit **Status Register 1**, which contains the two
bits the firmware needs to wait on: **BUSY/WIP** and **WEL**. This is the
only way for the master to know whether a previously started program or
erase has finished.

**Wire sequence.**

1. `CS` goes low.
2. Master shifts `05h` on `MOSI`.
3. Flash drives `MISO` with `S7…S0` from the next clock edge onward.
4. Master can keep clocking to receive the same byte repeatedly.
5. `CS` goes high to end the transaction.

**Frame table.**

| Phase | `MOSI` (FPGA → Flash) | `MISO` (Flash → FPGA) |
|---|---|---|
| Opcode | `05h` | High-Z (not driven) |
| Data (repeatable) | Don't care | `S7…S0`, then `S7…S0`, … |

**Waveform.**

![RDSR1 (05h) waveform](../../diagrams/protocol/waveform_rdsr1.svg)

*Figure 3 — RDSR1 (05h). After the opcode, the Flash drives `MISO` on
every clock.*

**Datasheet reference.** §8.2.4, Figure 8a.

**Status Register 1 bit definitions (only relevant bits).**

| Bit | Name | Meaning | Exposed as |
|---|---|---|---|
| `S0` | BUSY / WIP | `1` while an internal program or erase is running. | `WIP` in `STATUS` |
| `S1` | WEL | `1` if WREN has been issued and not yet consumed. | `WEL` in `STATUS` |
| `S2..S7` | BP/TB/SEC/SRP0 | Write-protection configuration | Ignored |

**Design note.** RDSR1 is the **polling command**. Firmware issues it
in a loop until `WIP = 0`. The RTL does not loop; it exposes one-shot
transactions and lets firmware drive the loop.

### 3.4 Read Status Register 2 — RDSR2 (35h)

**Purpose.** Read the 8-bit **Status Register 2**, which contains the
`QE` (Quad Enable), `CMP` (Complement Protect), `SRL` (Status Register
Lock), `LB1..LB3` (OTP lock bits), and `SUS` (Suspend status) bits.
In this project only `SUS` and `QE` are informational; the rest is
ignored. RDSR2 is included for completeness and for future Quad mode
enablement.

**Wire sequence.**

1. `CS` goes low.
2. Master shifts `35h` on `MOSI`.
3. Flash drives `MISO` with `S15…S8` from the next clock edge onward.
4. Master can keep clocking to receive the same byte repeatedly.
5. `CS` goes high.

**Frame table.**

| Phase | `MOSI` (FPGA → Flash) | `MISO` (Flash → FPGA) |
|---|---|---|
| Opcode | `35h` | High-Z (not driven) |
| Data (repeatable) | Don't care | `S15…S8`, then `S15…S8`, … |

**Waveform.**

![RDSR2 (35h) waveform](../../diagrams/protocol/waveform_rdsr2.svg)

*Figure 4 — RDSR2 (35h). Identical envelope to RDSR1; only the opcode
byte changes (`35h` = `0011 0101`) and the byte returned is Status
Register 2.*

**Datasheet reference.** §8.2.4, Figure 8a (same instruction family as
RDSR1).

**Status Register 2 bit definitions (relevant bits).**

| Bit | Name | Meaning | Used |
|---|---|---|---|
| `S9` | QE | Quad Enable (must be 1 for Quad mode) | Not used (Standard SPI only) |
| `S14` | CMP | Complement Protect | Ignored |
| `S15` | SUS | Suspend status | Read-only, informational |

**Design note.** Same FSM as RDSR1; only the opcode byte differs. The
`DATA` register exposes the returned byte to firmware.

### 3.5 Write Status Register — WRSR (01h)

**Purpose.** Write the 8-bit Status Register 1. Used in this project only
for **`WEL`-protected writes of protection bits** — which the project
does not configure — so WRSR is included for completeness and for future
use (e.g., enabling `QE` if the design ever migrates to Quad SPI).
WRSR requires a preceding `WREN`.

**Wire sequence.**

1. `CS` goes low.
2. Master shifts `01h` on `MOSI`.
3. Master shifts the new status byte on `MOSI`.
4. `CS` goes high (must be on a full byte boundary).

**Frame table.**

| Phase | `MOSI` (FPGA → Flash) | `MISO` (Flash → FPGA) |
|---|---|---|
| Opcode | `01h` | High-Z (not driven) |
| Data | `S7…S0` | High-Z (not driven) |

**Waveform.**

![WRSR (01h) waveform](../../diagrams/protocol/waveform_wrsr.svg)

*Figure 5 — WRSR (01h). Two bytes on `MOSI`: opcode `01h` followed by the
status byte to be written. `MISO` remains high-Z.*

**Datasheet reference.** §8.2.5, Figure 9a.

**Design note.** Same FSM as `PAGE_PROGRAM` but with zero address bytes
and exactly one data byte. The controller can implement it as a
degenerate case of the write FSM: `CMD=WRSR`, `LEN=1`, `ADDR=0`.

## 4. Read commands

Read commands retrieve data from the memory array. The Flash returns the
bytes on `MISO` while the master continues to clock.

### 4.1 Read Data — READ (03h)

**Purpose.** Read one or more bytes from the memory array starting at a
24-bit address. The internal address pointer advances automatically, so
the whole memory can be read with one command as long as `CS` stays low.
Maximum clock frequency: 50 MHz (see `flash_memory.md` §7).

**Wire sequence.**

1. `CS` goes low.
2. Master shifts `03h` on `MOSI`.
3. Master shifts the 24-bit address `A23…A0` (MSB first, 3 bytes).
4. Flash begins returning data on `MISO` after the last address bit.
5. Master clocks as many bytes as it needs.
6. `CS` goes high to end the transaction.

**Frame table.**

| Phase | `MOSI` (FPGA → Flash) | `MISO` (Flash → FPGA) |
|---|---|---|
| Opcode | `03h` | High-Z |
| Address byte 1 | `A23…A16` | High-Z |
| Address byte 2 | `A15…A8` | High-Z |
| Address byte 3 | `A7…A0` | High-Z |
| Data 0 | Don't care | `D7…D0` |
| Data 1 | Don't care | `D7…D0` |
| … | Don't care | … |

**Waveform.**

![READ (03h) waveform](../../diagrams/protocol/waveform_read.svg)

*Figure 6 — READ (03h). After the 8-bit opcode and the 24-bit address,
the Flash drives `MISO` with data bytes. The address auto-increments on
every byte boundary.*

**Datasheet reference.** §8.2.6, Figure 14.

**Design note.** The controller exposes `ADDR` (24 bits) and `LEN`
(number of bytes) in the CSR. After `START`, the FSM shifts opcode +
address, then fills the RX FIFO with `LEN` bytes. `DONE` is asserted
when the FIFO is full.

### 4.2 Fast Read — FAST_READ (0Bh)

**Purpose.** Same as READ but with an extra dummy byte after the address.
The dummy gives the Flash internal time to prepare the first data byte,
which allows the clock to run faster (up to 133 MHz on the reference
device). At 50 MHz the two commands have the same effective throughput;
the extra byte matters at higher frequencies.

**Wire sequence.**

1. `CS` goes low.
2. Master shifts `0Bh` on `MOSI`.
3. Master shifts the 24-bit address `A23…A0`.
4. Master shifts **one dummy byte** (8 clocks, `MOSI` value is don't-care,
   `MISO` is don't-care).
5. Flash begins returning data on `MISO`.
6. Master clocks as many bytes as it needs.
7. `CS` goes high.

**Frame table.**

| Phase | `MOSI` (FPGA → Flash) | `MISO` (Flash → FPGA) |
|---|---|---|
| Opcode | `0Bh` | High-Z |
| Address byte 1 | `A23…A16` | High-Z |
| Address byte 2 | `A15…A8` | High-Z |
| Address byte 3 | `A7…A0` | High-Z |
| Dummy byte | Don't care | Don't care (usually High-Z or 0) |
| Data 0 | Don't care | `D7…D0` |
| Data 1 | Don't care | `D7…D0` |

**Waveform.**

![FAST_READ (0Bh) waveform](../../diagrams/protocol/waveform_fast_read.svg)

*Figure 7 — FAST_READ (0Bh). Identical to READ plus one dummy byte
between the address and the first data byte. The dummy is highlighted in
the figure to make its role explicit.*

**Datasheet reference.** §8.2.7, Figure 16a.

**Design note.** Same FSM as READ, with an extra 8-clock "dummy" state
between address and data. The CSR needs no new registers; `CMD` selects
`READ` or `FAST_READ` and the FSM handles the difference.

## 5. Write and erase commands

Write and erase commands modify memory. All of them **require a preceding
`WREN`** and all of them **must terminate on a full byte boundary**
(`CS` high after the last full byte). After the command ends, the Flash
internally executes the operation and sets `WIP = 1`; firmware must poll
`RDSR1` until `WIP = 0` before issuing the next command.

### 5.1 Page Program — PAGE_PROGRAM (02h)

**Purpose.** Program up to 256 bytes (one page) starting at the given
address. Bytes are ANDed with `0xFF` of the currently erased state; the
operation can only flip bits from `1` to `0`. The whole page should be
erased first if a full overwrite is desired.

**Wire sequence.**

1. `WREN` (separate transaction).
2. `CS` goes low.
3. Master shifts `02h` on `MOSI`.
4. Master shifts the 24-bit address.
5. Master shifts 1–256 data bytes on `MOSI`.
6. `CS` goes high (byte boundary).
7. Firmware polls `RDSR1` until `WIP = 0`.

**Frame table.**

| Phase | `MOSI` (FPGA → Flash) | `MISO` (Flash → FPGA) |
|---|---|---|
| Opcode | `02h` | High-Z |
| Address byte 1 | `A23…A16` | High-Z |
| Address byte 2 | `A15…A8` | High-Z |
| Address byte 3 | `A7…A0` | High-Z |
| Data 0 | `D7…D0` | High-Z |
| Data 1 | `D7…D0` | High-Z |
| … up to Data 255 | `D7…D0` | High-Z |

**Waveform.**

![PAGE_PROGRAM (02h) waveform](../../diagrams/protocol/waveform_page_program.svg)

*Figure 8 — PAGE_PROGRAM (02h). Opcode + 24-bit address + 1 to 256 data
bytes on `MOSI`. The Flash never drives `MISO`. `CS` must rise on a
byte boundary.*

**Datasheet reference.** §8.2.13, Figure 29a.

**Design note.** The controller exposes a TX FIFO on the `DATA` register.
Firmware pushes `LEN` bytes, then sets `CMD=PAGE_PROGRAM` and `START`.
The FSM shifts opcode + address + `LEN` bytes from the FIFO, deasserts
`CS`, and asserts `DONE`. Polling `WIP` is firmware's job.

### 5.2 Sector Erase — SECTOR_ERASE (20h)

**Purpose.** Erase the 4 KB sector containing the given address. Every
byte in the sector becomes `0xFF`. Faster than block erase but erases
the whole sector regardless of the address's low 12 bits.

**Wire sequence.**

1. `WREN`.
2. `CS` goes low.
3. Master shifts `20h` on `MOSI`.
4. Master shifts the 24-bit address (only `A23…A12` matter).
5. `CS` goes high.
6. Firmware polls `RDSR1` until `WIP = 0`.

**Frame table.**

| Phase | `MOSI` (FPGA → Flash) | `MISO` (Flash → FPGA) |
|---|---|---|
| Opcode | `20h` | High-Z |
| Address byte 1 | `A23…A16` | High-Z |
| Address byte 2 | `A15…A8` | High-Z |
| Address byte 3 | `A7…A0` (low 12 bits ignored by Flash) | High-Z |

**Waveform.**

![SECTOR_ERASE (20h) waveform](../../diagrams/protocol/waveform_sector_erase.svg)

*Figure 9 — SECTOR_ERASE (20h). Opcode + 24-bit address. `MISO` never
driven. The Flash erases the 4 KB sector containing the address.*

**Datasheet reference.** §8.2.15, Figure 31a.

**Design note.** Same FSM as PAGE_PROGRAM but with `LEN = 0` data bytes.
Typical time: 45 ms; maximum 400 ms. Firmware timeout must be ≥ 500 ms.

### 5.3 Block Erase — BLOCK_ERASE (D8h)

**Purpose.** Erase the 64 KB block containing the given address. Faster
per byte than sector erase but erases the whole block regardless of the
address's low 16 bits.

**Wire sequence.**

1. `WREN`.
2. `CS` goes low.
3. Master shifts `D8h` on `MOSI`.
4. Master shifts the 24-bit address (only `A23…A16` matter).
5. `CS` goes high.
6. Firmware polls `RDSR1` until `WIP = 0`.

**Frame table.**

| Phase | `MOSI` (FPGA → Flash) | `MISO` (Flash → FPGA) |
|---|---|---|
| Opcode | `D8h` | High-Z |
| Address byte 1 | `A23…A16` | High-Z |
| Address byte 2 | `A15…A8` (ignored by Flash) | High-Z |
| Address byte 3 | `A7…A0` (ignored by Flash) | High-Z |

**Waveform.**

![BLOCK_ERASE (D8h) waveform](../../diagrams/protocol/waveform_block_erase.svg)

*Figure 10 — BLOCK_ERASE (D8h). Opcode + 24-bit address (only
`A23…A16` are decoded). `MISO` never driven.*

**Datasheet reference.** §8.2.17, Figure 33a.

**Design note.** Same FSM as SECTOR_ERASE. Typical time: 150 ms; maximum
2000 ms. Firmware timeout must be ≥ 3 s.

### 5.4 Chip Erase — CHIP_ERASE (C7h)

**Purpose.** Erase the whole memory array. No address. The longest
erase operation (up to 100 s on the reference device).

**Wire sequence.**

1. `WREN`.
2. `CS` goes low.
3. Master shifts `C7h` on `MOSI`.
4. `CS` goes high.
5. Firmware polls `RDSR1` until `WIP = 0` (long wait).

**Frame table.**

| Phase | `MOSI` (FPGA → Flash) | `MISO` (Flash → FPGA) |
|---|---|---|
| Opcode | `C7h` | High-Z |

**Waveform.**

![CHIP_ERASE (C7h) waveform](../../diagrams/protocol/waveform_chip_erase.svg)

*Figure 11 — CHIP_ERASE (C7h). Single byte on `MOSI`, no address. `MISO`
never driven. The erase happens inside the Flash after `CS` goes high.*

**Datasheet reference.** §8.2.18, Figure 34.

**Design note.** Same FSM as WREN but with a longer timeout. Firmware
should warn the user (this is a destructive operation).

## 6. Cross-cutting design implications

These observations apply to every command above.

- **`CS` frames every transaction.** The Flash accepts commands only when
  `CS` is low. Between transactions `CS` must be high and the bus idle.
- **Byte boundaries.** Read commands can stop on any clock edge. Write
  and erase commands must finish on a **full byte boundary**. The
  controller's FSM must enforce this for `WREN`, `WRSR`, `PAGE_PROGRAM`,
  `SECTOR_ERASE`, `BLOCK_ERASE` and `CHIP_ERASE`.
- **One command per transaction.** The controller issues exactly one
  opcode per `CS` low–high pulse. Composite operations (`WREN` +
  `PAGE_PROGRAM` + polling) are sequences of separate transactions,
  orchestrated by firmware.
- **Polling after write/erase.** `WIP` must be checked via `RDSR1` after
  every program and erase. The RTL does not poll by itself.
- **Page-boundary wrap.** `PAGE_PROGRAM` wraps around to the start of the
  same 256-byte page if more than 256 bytes are sent. Firmware must
  split writes at page boundaries.

## 7. Commands explicitly out of scope

The reference datasheet documents 47 instructions. The controller does
not implement any of the following, and firmware must not issue them:

- Dual/Quad I/O reads (`3Bh`, `BBh`, `6Bh`, `EBh`) — require IO1/IO2/IO3
  wiring and `QE=1` configuration.
- Read Status Register 3 (`15h`), Write Status Register 2/3 (`31h`,
  `11h`) — not used.
- Individual Block/Sector Lock (`36h`, `39h`, `3Dh`, `7Eh`, `98h`) —
  not used.
- Erase/Program Suspend and Resume (`75h`, `7Ah`) — polling model is
  sufficient.
- Power-down (`B9h`) and Release Power-down (`ABh`) — the device stays
  powered during a game session.
- Software Reset (`66h`, `99h`) — not needed.
- Security Register operations (`42h`, `44h`, `48h`) — not used.
- Unique ID (`4Bh`), SFDP (`5Ah`), JEDEC ID (`9Fh`), Manufacturer/Device
  ID (`90h`, `92h`, `94h`) — the controller is designed against the
  reference device; identity checks are optional and can be added in a
  later revision of the verification plan.

> **Rationale.** Every command in this out-of-scope list either uses
> additional pins (Dual/Quad), writes registers the project does not use
> (block locks, SR3), or is a device-management feature not required by
> the KUBIX runtime.

## 8. References

- Winbond **W25Q128JV** datasheet, revision E (23-Nov-2017). Section 8
  (Instructions) and Section 9.6 (AC Electrical Characteristics).
- Local copy of datasheet: [`references/spi_flash/`](../../references/spi_flash/).
- SPI protocol: [`spi_protocol.md`](spi_protocol.md).
- Target device class: [`flash_memory.md`](flash_memory.md).

---

*Last updated: 2026-10-XX*
*Authors: @camahechag-bars, @jcanizalez16, @edfrancos*