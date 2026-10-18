# Operation Flow

[← Back to Specification index](README.md) · [← Back to project README](../../README.md)

## Contents

1. [Purpose](#1-purpose)
2. [Where this fits](#2-where-this-fits)
3. [READ operation](#3-read-operation)
4. [WRITE operation](#4-write-operation)
5. [ERASE operation](#5-erase-operation)
6. [The WIP polling strategy](#6-the-wip-polling-strategy)
7. [Error paths](#7-error-paths)
8. [Timing budget](#8-timing-budget)
9. [References](#9-references)

---

## 1. Purpose

Previous documents described individual commands
([`flash_commands.md`](flash_commands.md)) and the register interface
([`csr_interface.md`](csr_interface.md)). This document describes the
**sequences** — how firmware and RTL cooperate across more than one
transaction to accomplish a user-level operation: reading bytes,
writing a page, erasing a sector.

Each sequence is presented as:

1. **A flowchart** (original SVG) showing every step, color-coded by who
   performs it (firmware, RTL, or the Flash device).
2. **Prose** explaining the intent behind each group of steps.
3. **A reference** to the CSR registers and commands involved.

The three sequences covered are the only ones the KUBIX firmware needs.
Everything else is a variation (READ vs FAST_READ, SECTOR_ERASE vs
BLOCK_ERASE vs CHIP_ERASE, WRSR being a degenerate WRITE).

## 2. Where this fits

    Specification documents (what)      ──►  This document (how, in sequence)
    flash_memory.md         (device)         operation_flow.md
    spi_protocol.md         (wire)
    flash_commands.md       (per command)
    csr_interface.md        (registers)

Downstream, `02_architecture.md` will use these sequences as the input
for the RTL FSM design.

**Color convention in the flowcharts:**

- Blue box — firmware (`mmio_*` calls)
- Green box — RTL (controller internal actions)
- Red box — Flash device (actions taken by the slave)
- Yellow diamond — decision

## 3. READ operation

Reading bytes from the Flash is a single transaction followed by a
short read-out of the RX FIFO. No polling of `WIP` is required because
reads do not modify memory.

![READ operation flowchart](../../diagrams/design/flow_read.svg)

*Figure 1 — READ sequence. Firmware configures the address and length,
starts the transaction, waits for `BUSY` to clear, checks `ERR`, and
drains the RX FIFO through the `DATA` register.*

The sequence in prose:

1. **Configure.** Firmware writes `ADDR` (24 bits) and `LEN` (number of
   bytes). Both must be written **before** `CTRL`.
2. **Start.** Firmware writes `CTRL` with `CMD = READ` (`0`) and
   `START = 1`. The RTL immediately begins.
3. **RTL transaction.** The RTL asserts `CS` low, shifts the opcode
   `03h`, shifts the 24-bit address, then captures `LEN` bytes from
   `MISO` into the RX FIFO. It deasserts `CS` and sets `DONE`.
4. **Wait.** Firmware polls `STATUS.BUSY` until it clears. Because the
   FIFO is small, the RTL only clears `BUSY` when all `LEN` bytes are
   already in the FIFO.
5. **Check.** Firmware reads `STATUS` to check `ERR` (which clears both
   `DONE` and `ERR`). If `ERR = 1`, return an I/O error.
6. **Drain.** Firmware reads `DATA` `LEN` times, storing each returned
   byte (bits `7:0`) into the destination buffer.

**CSR registers used:** `ADDR`, `LEN`, `CTRL`, `STATUS`, `DATA`.
**Commands used:** READ (`03h`).
**Datasheet reference:** §8.2.6, Figure 14.

## 4. WRITE operation

Writing bytes requires **three** transactions: a `WREN`, a
`PAGE_PROGRAM`, and a polling loop of `RDSR1` until the Flash's internal
programming finishes.

![WRITE operation flowchart](../../diagrams/design/flow_write.svg)

*Figure 2 — WRITE sequence (PAGE_PROGRAM). A `WREN` enables writing;
`PAGE_PROGRAM` sends the data; `RDSR1` polling waits until the Flash
clears `WIP`.*

The sequence in prose:

1. **Enable.** Firmware sends `WREN` (`CMD = 2`). The RTL shifts `06h`,
   deasserts `CS`, and sets `DONE`. Firmware polls `BUSY` and checks
   `ERR`. If `ERR = 1`, abort.
2. **Configure.** Firmware writes `ADDR` (page-aligned for best results),
   `LEN` (`1..256`), then pushes `LEN` bytes into the TX FIFO via `DATA`.
   If the FIFO depth is smaller than `LEN`, this loop must interleave with
   `BUSY` checks — the current implementation uses a FIFO of at least
   `256` bytes so the whole page can be pushed at once.
3. **Program.** Firmware writes `CTRL` with `CMD = PAGE_PROGRAM` (`7`)
   and `START = 1`. The RTL shifts `02h`, the 24-bit address, and the
   `LEN` data bytes from the TX FIFO. It deasserts `CS` and sets `DONE`.
4. **Flash internal.** The Flash starts programming its internal array.
   `WIP` becomes `1` inside the device. `DONE` in our `STATUS` is
   already set (the SPI transaction is over), but the operation is
   **not** complete yet.
5. **Poll.** Firmware enters the `RDSR1` loop: send `RDSR1` (`CMD = 4`),
   wait `BUSY` clear, read `STATUS.WIP`. If `WIP = 1`, loop again. When
   `WIP = 0`, the Flash has finished.
6. **Timeout.** Firmware applies a per-command timeout (see Section 8).
   If the loop exceeds it, return a timeout error.

**CSR registers used:** `CTRL`, `STATUS`, `ADDR`, `LEN`, `DATA`.
**Commands used:** WREN (`06h`), PAGE_PROGRAM (`02h`), RDSR1 (`05h`).
**Datasheet reference:** §8.2.1, §8.2.4, §8.2.13.

**Page-boundary note.** A `PAGE_PROGRAM` that crosses a 256-byte page
boundary wraps around to the start of the same page (see
`flash_memory.md` §5). Firmware must split any write that crosses a
page boundary into two `PAGE_PROGRAM` transactions.

## 5. ERASE operation

Erasing requires **three** transactions, same shape as WRITE but with
no data phase: `WREN`, then the erase command (`SECTOR_ERASE`,
`BLOCK_ERASE` or `CHIP_ERASE`), then the `RDSR1` polling loop. The
polling timeout is much longer because erase takes milliseconds to
seconds.

![ERASE operation flowchart](../../diagrams/design/flow_erase.svg)

*Figure 3 — ERASE sequence (SECTOR_ERASE shown; BLOCK_ERASE and
CHIP_ERASE follow the same shape). A `WREN` enables erasing; the erase
command is issued; `RDSR1` polling waits for `WIP` to clear.*

The sequence in prose:

1. **Enable.** Same as WRITE — send `WREN`, wait, check `ERR`.
2. **Configure.** Firmware writes `ADDR` with the sector- or block-aligned
   address. `LEN` is not used by erase commands.
3. **Erase.** Firmware writes `CTRL` with `CMD = SECTOR_ERASE (8)` or
   `CMD = BLOCK_ERASE (9)` or `CMD = CHIP_ERASE (10)`, and `START = 1`.
   The RTL shifts the opcode and (for sector/block) the address, then
   deasserts `CS` and sets `DONE`.
4. **Flash internal.** The Flash begins the erase. `WIP = 1`.
5. **Poll.** Same `RDSR1` loop as in WRITE, but with a longer timeout
   (see Section 8). When `WIP = 0`, the erase has finished.

**CSR registers used:** `CTRL`, `STATUS`, `ADDR`.
**Commands used:** WREN (`06h`), SECTOR_ERASE (`20h`), BLOCK_ERASE
(`D8h`), CHIP_ERASE (`C7h`), RDSR1 (`05h`).
**Datasheet reference:** §8.2.1, §8.2.4, §8.2.15, §8.2.17, §8.2.18.

**Destructive operation warning.** `CHIP_ERASE` erases the entire memory
array. Firmware must never call it without explicit user confirmation.

## 6. The WIP polling strategy

Both WRITE and ERASE depend on polling the Flash's internal `WIP` bit
via `RDSR1`. The reason is architectural: the controller has no
back-channel from the Flash that says "I'm done". The only way to know
is to read `STATUS` and look at `WIP`.

**Why not poll at the RTL level?** Because it would tie the controller to
a specific timeout (which depends on which command was issued and on the
specific Flash part) and would make the RTL less reusable. Polling in
firmware keeps the RTL generic and lets the firmware tune its own
timeout, retry policy, and error handling.

**Why not use an interrupt?** Because the CSR does not expose one
(see `requirements.md` §7 — interrupts are out of scope). Firmware
polls.

**Polling loop shape (in C):**

    do {
        start_cmd(CMD_RDSR1);
        wait_busy();
        st = read_status();      // clears DONE/ERR
        if (st & STATUS_ERR) return -EIO;
    } while (st & STATUS_WIP);

**Polling interval.** The loop as written is a tight loop with no delay.
At a 10 MHz `SCLK`, a single `RDSR1` transaction takes 16 clocks
(~1.6 µs). A tight loop with no delay hits the Flash with a new status
read every ~2 µs. This is well within the datasheet's recommendation
(status reads are allowed at any time, even during an erase) but wastes
CPU cycles and power. If needed, firmware can add a short delay between
iterations; the controller does not require it.

**Give the Flash a head start.** Immediately after `DONE` for the
program/erase command, the Flash has just received the last byte and
has not yet asserted `WIP = 1` internally. The first `RDSR1` should be
issued after a short delay (a few hundred microseconds) to avoid
reading `WIP = 0` before the Flash has started. The datasheet states
that `WIP` goes high within ~200 ns of `CS` rising, so a **1 µs** delay
is sufficient. Firmware should not skip this delay.

## 7. Error paths

The three sequences above share the same error-recovery policy:

- **RTL-detected errors** (`STATUS.ERR = 1`): caused by firmware bug
  (reserved `CMD`, `LEN=0` for a data command, FIFO under/overflow). Fix
  the caller, do not retry.
- **Flash-side errors** (`WIP` stuck at `1` beyond the timeout): the Flash
  is unresponsive or the erase/program has failed. **Retry once**, then
  return error. The controller cannot distinguish "Flash is dead" from
  "Flash is just slow" — the timeout is what makes the distinction.
- **Timeouts** (Section 8): return `-ETIMEDOUT`. Never retry a timeout
  more than once; a hung Flash will hang the whole game.

The controller itself never retries. It performs exactly one transaction
per `CTRL.START`. All retry logic lives in firmware.

## 8. Timing budget

Firmware must size its timeouts based on the reference device's worst-case
timings from `flash_memory.md` §7:

| Operation | Typical | Max | Recommended firmware timeout |
|---|---|---|---|
| PAGE_PROGRAM | 0.7 ms | 3 ms | 10 ms |
| SECTOR_ERASE (4 KB) | 45 ms | 400 ms | 1 s |
| BLOCK_ERASE (64 KB) | 150 ms | 2000 ms | 5 s |
| CHIP_ERASE | 20 s | 100 s | 120 s |

The controller does not enforce these timeouts. They are the firmware's
responsibility because only firmware knows which command it issued and
how long it can afford to wait.

**Bring-up recommendation.** Start with the maximum timeout values and
reduce them as the system is validated. During early bring-up it is
better to wait too long than to abort a correct operation.

## 9. References

- [`flash_commands.md`](flash_commands.md) — per-command frames.
- [`csr_interface.md`](csr_interface.md) — registers used by every step.
- [`flash_memory.md`](flash_memory.md) §7 — timing used to size timeouts.
- [`requirements.md`](requirements.md) — RFs that these sequences satisfy
  (RF-11 to RF-21).
- Winbond W25Q128JV datasheet, §8.2.4 (`RDSR1`), §8.2.13
  (`PAGE_PROGRAM`), §8.2.15–§8.2.18 (`SECTOR`/`BLOCK`/`CHIP_ERASE`).

---

*Last updated: 2026-10-XX*
*Authors: @camahechag-bars, @jcanizalez16, @edfrancos*