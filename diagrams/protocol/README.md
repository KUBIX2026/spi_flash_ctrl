# Protocol Diagrams

[← Back to README](../../README.md) · [← Back to Specification](../../docs/01_specification/README.md)

Engineering diagrams documenting the SPI protocol applied to Flash memory.
These diagrams are referenced from [`docs/01_specification/spi_protocol.md`](../../docs/01_specification/spi_protocol.md).

## Index

| Diagram | Description |
|---|---|
| [read_03h.png](read_03h.png) | READ (0x03h) transaction waveform |
| [fast_read_0bh.png](fast_read_0bh.png) | FAST READ (0x0Bh) transaction waveform |
| [dual_read_3bh.png](dual_read_3bh.png) | DUAL READ (0x3Bh) transaction waveform |
| [quad_read_6bh.png](quad_read_6bh.png) | QUAD READ (0x6Bh) transaction waveform |
| [wren_rdsr_rdid_fread.png](wren_rdsr_rdid_fread.png) | Control commands: WREN, RDSR, RDID, FREAD |
| [commands_summary.png](commands_summary.png) | Summary table of Flash commands |
| [spi_protocol_overview.png](spi_protocol_overview.png) | Overview of the SPI protocol applied to Flash |
| [flash_operation_flow.png](flash_operation_flow.png) | Complete operation flowchart (read, program, erase) |

PDF versions are available for the larger diagrams:
[commands_summary.pdf](commands_summary.pdf),
[flash_operation_flow.pdf](flash_operation_flow.pdf),
[spi_protocol_overview.pdf](spi_protocol_overview.pdf).

## Naming Convention

Lowercase `snake_case`, with opcode in hexadecimal when applicable.

Examples:
- `read_03h.png`
- `fast_read_0bh.png`
- `flash_operation_flow.png`

## Status

Some diagrams are marked for regeneration. See the corresponding section
in [`spi_protocol.md`](../../docs/01_specification/spi_protocol.md).
