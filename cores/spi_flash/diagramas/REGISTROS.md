# Registros y banderas del módulo de la flash

Esta es la **única fuente** de las direcciones del módulo. Si una dirección cambia, se edita aquí y se avisa al equipo.

- Rango del grupo D en el bus: `0x420000`–`0x42FFFF`.
- `addr` son los bits bajos de la dirección del bus y eligen un registro del módulo. **No** es la dirección dentro de la flash.
- Las direcciones son ejemplos mientras no se confirmen con el profesor y con el bus real.
- `X` = no importa lo que haya en ese pin.

## Registros

| Registro | addr | Acceso | Bits | Qué hace |
|---|---|---|---|---|
| `LEC` | `0x0007` | Escritura | `d_in` = X | Arranca la lectura: `BUSY` ← 1 y calcula `size_lec` y `size_lec_b` |
| `DIR_FIN` | `0x0008` | Escritura | `d_in[23:0]` | Guarda la dirección final en `buf_fin` |
| `DIR_INI` | `0x0009` | Escritura | `d_in[23:0]` | Guarda la dirección en `buf_ini`: inicio de la lectura, o sector o bloque a borrar |
| `DONE` | `0x0010` | Lectura | `d_out[0]` | Hay un paquete de 32 bits listo |
| `DATO` | `0x0011` | Lectura | `d_out[31:0]` | Entrega el paquete. Baja `DONE` y reanuda `sclk` |
| `LAST` | `0x0012` | Lectura | `d_out[0]` | El paquete listo es el último |
| `BYTES_VAL` | `0x0013` | Lectura | `d_out[1:0]` | Bits válidos del último paquete: `00`=8, `01`=16, `10`=24, `11`=32 |
| `DATO_LAST` | `0x0014` | Lectura | `d_out[31:0]` | Entrega el último paquete. Baja `LAST` y `BUSY` y reinicia el módulo |
| `BUSY` | `0x0015` | Lectura | `d_out[0]` | Hay una operación en curso (lectura o borrado) |
| `ERROR` | `0x0017` | Lectura | `d_out[0]` | La última operación falló: se agotó `t_tot` esperando a que la flash terminara |
| `BORRAR_SECTOR` | `0x0020` | Escritura | `d_in` = X | Borra el sector que contiene la dirección de `buf_ini` (comando SPI `0x20`) |
| `BORRAR_BLOQUE` | `0x0021` | Escritura | `d_in` = X | Borra el bloque que contiene la dirección de `buf_ini` (comando SPI `0xD8`) |
| `BORRAR_CHIP` | `0x0030` | Escritura | `d_in` = X | Borra el chip completo y descarta `buf_ini` (comando SPI `0xC7`) |

- En las lecturas de bandera, los bits de `d_out` que no se listan son X.
- Los comandos SPI de borrado son los habituales; hay que confirmarlos con el datasheet del chip real.
- Sector = 4 KB y bloque = 64 KB. La flash borra la sección a la que pertenece la dirección, sin importar en qué parte de ella caiga.
- Para borrar un sector o un bloque, primero se escribe `DIR_INI` y después el registro de borrado. Para borrar el chip no hace falta `DIR_INI`.

## Banderas

| Bandera | Registro | addr | Bit | Sube cuando | Baja cuando |
|---|---|---|---|---|---|
| `BUSY` | `BUSY` | `0x0015` | 0 | Llega `INIT` o se escribe un registro de borrado | La CPU lee `DATO_LAST` (lectura), o termina el borrado, o se agota `t_tot` |
| `DONE` | `DONE` | `0x0010` | 0 | Se completa un paquete de 32 bits que no es el último | La CPU lee `DATO` |
| `LAST` | `LAST` | `0x0012` | 0 | Llega el último paquete, después de calcular `bytes_val` | La CPU lee `DATO_LAST` |
| `ERROR` | `ERROR` | `0x0017` | 0 | Se agota `t_tot` sin que la flash termine el borrado | Empieza una operación nueva (junto con `BUSY` ← 1) |

- `DONE` y `LAST` nunca valen 1 a la vez. Para el último paquete, el firmware lee primero `LAST`, luego `BYTES_VAL` y al final `DATO_LAST`, porque esa última lectura reinicia el módulo.
- Ante un fallo de borrado, `ERROR` sube **antes** de que `BUSY` baje. Así, cuando la CPU ve `BUSY` en 0, `ERROR` ya tiene su valor definitivo.
- El módulo consulta `WIP` por su cuenta. La CPU nunca lo ve: solo ve `BUSY` y `ERROR`.


Si se cambia una dirección, se actualizan esta tabla y ese archivo en el mismo commit.
