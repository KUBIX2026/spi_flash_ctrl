# Nomenclatura del módulo de la flash (grupo D)

Reglas para que los tres usemos los mismos nombres en diagramas, tablas y Verilog. Si hay que agregar un nombre nuevo, se escoge el prefijo según su tipo.

## Tipos de dato interno (minúsculas con guion bajo)

| Prefijo | Tipo | Criterio | Ejemplos |
|---|---|---|---|
| `cont_` | Contador | Cuenta eventos, de 1 en 1 | `cont_env`, `cont_paq`, `cont_tot` |
| `buf_` | Buf | Guarda un dato tal como llega, sin transformarlo | `buf_ini`, `buf_fin`, `buf_lec` |
| `size_` | Variable calculada | Resultado calculado a partir de otros datos | `size_lec`, `size_lec_b` |

- Otras variables calculadas que no son tamaños usan un nombre descriptivo en minúsculas, por ejemplo `bytes_val`.
- El sufijo `_b` indica que el valor está en **bits**. Sin sufijo, está en **bytes**.

## Registros y banderas que ve la CPU (MAYÚSCULAS)

- Los registros a los que accede la CPU van en mayúsculas: `DIR_INI`, `DIR_FIN`, `INIT`, `DATO`, `DATO_LAST`, `BYTES_VAL`.
- Las banderas también van en mayúsculas: `BUSY`, `DONE`, `LAST`.
- Un registro es el **acceso** de la CPU (una dirección `addr`). Lo que se guarda dentro del módulo es un `buf_`. Por ejemplo, la CPU escribe en `DIR_INI` y el valor queda en `buf_ini`.
- La lista completa de registros, direcciones y banderas está en [`REGISTROS.md`](REGISTROS.md).

## Señales del bus y pines SPI

- Bus con la CPU: `csc`, `wr`, `rd`, `addr`, `d_in`, `d_out`.
- Pines SPI: `sclk`, `mosi`, `miso`. El chip select es `CS#` en los diagramas y `cs_n` en Verilog.
- `addr` son los bits bajos de la dirección del bus y eligen un registro del módulo. **No** es la dirección dentro de la flash.

## Convenciones de escritura

- Valor que no importa: `X`.
- Números hexadecimales con prefijo `0x` (por ejemplo `0x0007`), en README, tablas y diagramas.
- Las direcciones de los registros están en un solo lugar: `REGISTROS.md`. Si cambian, se corrigen allí y se avisa al equipo.
