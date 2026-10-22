# Registros y banderas del módulo de la flash

Esta es la **única fuente** de las direcciones del módulo. Si una dirección cambia, se edita aquí y se avisa al equipo.

- Rango del grupo D en el bus: `0x420000`–`0x42FFFF`.
- `addr` son los bits bajos de la dirección del bus y eligen un registro del módulo. **No** es la dirección dentro de la flash.
- Las direcciones son ejemplos mientras no se confirmen con el profesor y con el bus real.
- `X` = no importa lo que haya en ese pin.

## Registros

| Registro | addr | Acceso | Bits | Qué hace |
|---|---|---|---|---|
| `INIT` | `0x0007` | Escritura | `d_in` = X | Arranca la lectura: `BUSY` ← 1 y calcula `size_lec` y `size_lec_b` |
| `DIR_FIN` | `0x0008` | Escritura | `d_in[23:0]` | Guarda la dirección final en `buf_fin` |
| `DIR_INI` | `0x0009` | Escritura | `d_in[23:0]` | Guarda la dirección de inicio en `buf_ini` |
| `DONE` | `0x0010` | Lectura | `d_out[0]` | Hay un paquete de 32 bits listo |
| `DATO` | `0x0011` | Lectura | `d_out[31:0]` | Entrega el paquete. Baja `DONE` y reanuda `sclk` |
| `LAST` | `0x0012` | Lectura | `d_out[0]` | El paquete listo es el último |
| `BYTES_VAL` | `0x0013` | Lectura | `d_out[1:0]` | Bits válidos del último paquete: `00`=8, `01`=16, `10`=24, `11`=32 |
| `DATO_LAST` | `0x0014` | Lectura | `d_out[31:0]` | Entrega el último paquete. Baja `LAST` y `BUSY` y reinicia el módulo |
| `BUSY` | `0x0015` | Lectura | `d_out[0]` | Hay una lectura en curso |

En las lecturas de bandera, los bits de `d_out` que no se listan son X.

## Banderas

| Bandera | Registro | addr | Bit | Sube cuando | Baja cuando |
|---|---|---|---|---|---|
| `BUSY` | `BUSY` | `0x0015` | 0 | Llega `INIT` | La CPU lee `DATO_LAST` |
| `DONE` | `DONE` | `0x0010` | 0 | Se completa un paquete de 32 bits que no es el último | La CPU lee `DATO` |
| `LAST` | `LAST` | `0x0012` | 0 | Llega el último paquete, después de calcular `bytes_val` | La CPU lee `DATO_LAST` |

`DONE` y `LAST` nunca valen 1 a la vez. Para el último paquete, el firmware lee primero `LAST`, luego `BYTES_VAL` y al final `DATO_LAST`, porque esa última lectura reinicia el módulo.


Si se cambia una dirección, se actualizan esta tabla y ese archivo en el mismo commit.
