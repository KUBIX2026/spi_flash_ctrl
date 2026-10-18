[← Volver al índice de Specification](README.md) · [← Volver al README](../../README.md)

## Contenido

1. [Explicación general del protocolo](#1-explicación-general-del-protocolo)
2. [Comandos de estado y control](#2-comandos-de-estado-y-control)
3. [Comandos de lectura](#3-comandos-de-lectura)
4. [Comandos de escritura y borrado](#4-comandos-de-escritura-y-borrado)
5. [Diagrama de flujo general del protocolo](#5-diagrama-de-flujo-general-del-protocolo)

---

# 1. Explicación general del protocolo

## 1.1 ¿Qué es SPI?

SPI (**Serial Peripheral Interface**) es un protocolo de comunicación **sincrónico, serie y full-duplex**. Un dispositivo **maestro** (en nuestro caso, la FPGA) controla a uno o varios **esclavos** (en este caso, la memoria Flash).

## 1.2 Señales del bus

El protocolo cuenta con cuatro señales principales de conexión:

| Señal       | Nombre completo            | Función                               |
| ----------- | -------------------------- | ------------------------------------- |
| `SCLK`      | Serial Clock               | Reloj generado por master         |
| `MOSI`      | Master Out Slave In        | Datos de master hacia el slave    |
| `MISO`      | Master In Slave Out        | Datos del slave hacia el master    |
| `CS` / `SS` | Chip Select / Slave Select | Activo en bajo. Selecciona el slave |

## 1.3 Características principales

- **Protocolo sincrónico:** los datos se leen en los flancos del reloj. La configuración del flanco de lectura y el estado base del reloj determinan los cuatro modos de operación del protocolo.
- **Comunicación full-duplex:** el protocolo soporta comunicación bidireccional simultánea mediante las líneas independientes `MOSI` y `MISO`.
- **Generación del reloj:** la señal de reloj es generada por el maestro. El esclavo nunca genera el reloj.
- **Estabilidad de los datos:** cuando ocurre el flanco utilizado para leer un dato, la información debe encontrarse estable.
- **Velocidad de transmisión:** la frecuencia utilizada debe respetar los tiempos de operación establecidos para la memoria Flash.

> **Nota sobre velocidad:** en SPI la velocidad de transmisión no afecta la integridad del dato en sí — solo el tiempo que tarda en llegar. A diferencia de protocolos como I2S, donde la frecuencia de muestreo debe coincidir exactamente, en SPI se puede ir más rápido o más lento siempre que se respeten los tiempos mínimos del chip esclavo.

## 1.4 Modos SPI (CPOL / CPHA)

La comunicación SPI puede utilizar cuatro configuraciones determinadas por **CPOL** (Clock Polarity, estado base del reloj) y **CPHA** (Clock Phase, flanco de muestreo):

| Modo | CPOL | CPHA | Flanco de muestreo |
| ---- | ---- | ---- | ------------------ |
| 0    | 0    | 0    | Subida             |
| 1    | 0    | 1    | Bajada             |
| 2    | 1    | 0    | Bajada             |
| 3    | 1    | 1    | Subida             |

En memorias Flash se utiliza normalmente **Modo 0** o **Modo 3**, dependiendo de la memoria específica.

> **Regla de oro:** el dato cambia en un flanco del reloj y se lee en el otro. En el momento del flanco de lectura, el dato debe permanecer estable — no puede haber cambios en la señal de datos al mismo tiempo que el flanco de lectura.

## 1.5 Estructura general de la comunicación

Toda transacción con la Flash sigue, de manera general, el siguiente patrón:

```text
CS baja → [Comando] → [Dirección] → [Dummy, si aplica] → [Datos] → CS sube
```

Los elementos principales de la transacción son:

- **Comando:** 1 byte que indica la operación que se desea realizar.
- **Dirección:** 3 bytes (24 bits) que indican la posición de memoria.
- **Dummy:** 0 o más bytes de relleno utilizados por algunos comandos.
- **Datos:** información que se transmite o recibe.

La secuencia general de la comunicación es:

1. `CS` baja al inicio de la transacción.
2. El maestro genera el reloj.
3. Se transmite el comando.
4. Se transmite la dirección cuando la operación la requiere.
5. Se transmiten ciclos dummy cuando el comando los requiere.
6. Se transmiten o reciben los datos.
7. `CS` sube al finalizar la transacción.

## 1.6 Forma física

### 1.6.1 Conexión FPGA ↔ Flash

```mermaid
flowchart LR
    FPGA["FPGA (Maestro)"]
    FLASH["Flash (Esclavo)"]

    FPGA -- SCLK --> FLASH
    FPGA -- MOSI --> FLASH
    FLASH -- MISO --> FPGA
    FPGA -- CS --> FLASH
    FPGA -- 3.3V --> FLASH
    FPGA -- GND --> FLASH
```

### 1.6.2 Tabla de conexión

| FPGA (Maestro) | Dirección | Flash (Esclavo) | Pin Flash |
| -------------- | --------- | --------------- | --------- |
| SCLK           | →         | CLK             | 6         |
| MOSI           | →         | DI (IO0)        | 5         |
| MISO           | ←         | DO (IO1)        | 2         |
| CS             | →         | CS#             | 1         |
| 3.3V           | →         | VCC             | 8         |
| GND            | →         | GND             | 4         |

### 1.6.3 Niveles de voltaje

- La FPGA típicamente maneja **3.3 V**.
- La Flash SPI también trabaja a **3.3 V**.
- Si un chip trabaja a 5 V y otro a 3.3 V, es necesario un adaptador de niveles.

### 1.6.4 Pines típicos de una Flash (SOIC-8)

| Pin | Nombre      | Función                              |
| --- | ----------- | ------------------------------------ |
| 1   | CS#         | Chip Select (activo bajo)            |
| 2   | DO (IO1)    | Datos de salida / IO1 en Quad        |
| 3   | WP# (IO2)   | Write Protect / IO2 en Quad          |
| 4   | GND         | Tierra                               |
| 5   | DI (IO0)    | Datos de entrada / IO0 en Quad       |
| 6   | CLK         | Reloj                                |
| 7   | HOLD# (IO3) | Hold / IO3 en Quad                   |
| 8   | VCC         | Alimentación (3.3 V)                 |

## 1.7 Estructura interna de la memoria

Comprender cómo se divide internamente la memoria Flash es importante para entender la funcionalidad de las direcciones que se envían en múltiples comandos.

> ⚠️ **TODO:** incluir diagrama `estructura_memoria.png` (pendiente de crear).

Estas memorias típicamente se dividen en **3 niveles**. La memoria completa tiene una capacidad de **16 MiB**(Mebibyte), la cual se divide en **256 bloques** de **64 KB** cada uno. Cada bloque se divide en **16 sectores** de **4 KB** cada uno. Cada sector se vuelve a dividir en **16 páginas** de **256 Bytes** cada una, y estas últimas se dividen en los mencionados 256 Bytes.

> ⚠️ **TODO:** incluir diagrama `estructura_detallada_memoria.png` (pendiente de crear).

Las direcciones de memoria que se envían desde el maestro hacia el esclavo se dividen en **4 partes** (esto se refiere únicamente a la distribución de la información, las direcciones se envían de forma continua bit tras bit).

> ⚠️ **TODO:** incluir diagrama `estructura_direcciones.png` (pendiente de crear).

---
# 2. Comandos de estado y control

Esta sección presenta los comandos utilizados para controlar la memoria y consultar su estado.

## 2.1 Índice de comandos

| Código | Comando                | Función                                        |
| ------ | ---------------------- | ---------------------------------------------- |
| `06h`  | Write Enable           | Habilita escritura/programación                |
| `04h`  | Write Disable          | Deshabilita escritura/programación             |
| `05h`  | Read Status Register 1 | Consulta el registro de estado (WIP, WEL)      |
| `35h`  | Read Status Register 2 | Consulta información adicional de estado       |
| `01h`  | Write Status Register  | Escribe información en el registro de estado   |
| `C7h`  | Chip Erase             | Borra toda la memoria                          |

## 2.2 Uso de los comandos

### 2.2.1 Write Enable (`06h`)

Habilita la memoria para operaciones de escritura o borrado. Es **obligatorio** antes de cualquier `Page Program` o comando de `Erase`. No lleva dirección ni datos — solo el opcode.

### 2.2.2 Write Disable (`04h`)

Deshabilita las operaciones de escritura. Se usa cuando se quiere garantizar que la memoria no acepte comandos de escritura hasta un nuevo Write Enable.

### 2.2.3 Read Status Register 1 (`05h`)

Devuelve el registro de estado de 8 bits. Los bits relevantes son:

- **WIP** (Write In Progress, bit 0): indica si hay una operación en curso.
- **WEL** (Write Enable Latch, bit 1): indica si la memoria está habilitada para escritura.

> **Remark:** después de enviar un comando de escritura o borrado, el maestro debe **consultar repetidamente** este registro con `05h` hasta verificar que `WIP = 0`, confirmando que la operación interna terminó.

### 2.2.4 Read Status Register 2 (`35h`)

Devuelve un segundo registro con información adicional del chip (bits de protección, Quad Enable, etc.).

### 2.2.5 Write Status Register (`01h`)

Permite escribir en el registro de estado para configurar bits de protección, Quad Enable, etc.

## 2.3 Formas de onda

![Comandos de estado y control](../../diagrams/protocol/wren_rdsr_rdid_fread.png)

*(source: [wren_rdsr_rdid_fread.png](../../diagrams/protocol/wren_rdsr_rdid_fread.png))*



## 4. Protocolo de comunicación

### 4.1 Estructura general de una transacción

Toda transacción con la Flash sigue este patrón:

```
CS baja  →  [Comando]  →  [Dirección]  →  [Dummy]  →  [Datos]  →  CS sube
```

- **Comando:** 1 byte que dice qué hacer.
- **Dirección:** 3 bytes (24 bits) que indican la posición de memoria.
- **Dummy:** 0 o más bytes de relleno (solo en algunos comandos).
- **Datos:** bytes de lectura o escritura.

### 4.2 Reloj y flancos

- **CS baja** al inicio de la transacción.
- El maestro genera el reloj.
- **El dato cambia en un flanco y se lee en el otro.**
- Regla de oro vista en clase:

> *"No puedo tener cambios de la señal de dato al mismo tiempo que el flanco de lectura. Siempre que el clock cambia, el dato tiene que estar estable."*

---

## 5. Comandos disponibles

### 5.1 Comandos de estado y control

| Código | Nombre | Función |
|--------|--------|---------|
| `06h` | Write Enable | Habilita escritura (obligatorio antes de escribir/borrar) |
| `04h` | Write Disable | Deshabilita escritura |
| `05h` | Read Status Register 1 | Revisa WIP (busy) y WEL |
| `35h` | Read Status Register 2 | Info adicional del chip |
| `C7h` / `60h` | Chip Erase | Borra toda la memoria |

### 5.2 Comandos de lectura

| Código | Nombre | Dummy | Velocidad típica |
|--------|--------|-------|-------------------|
| `03h` | READ | No | 50 MHz |
| `0Bh` | FAST READ | 1 byte | 104 MHz |
| `3Bh` | FAST READ DUAL OUTPUT | 1 byte | 104 MHz |
| `BBh` | FAST READ DUAL I/O | 1 byte (modo) | 104 MHz |
| `6Bh` | FAST READ QUAD OUTPUT | 1 byte | 104 MHz |
| `EBh` | FAST READ QUAD I/O | 1 byte (modo) | 104 MHz |

### 5.3 Comandos de escritura y borrado

| Código | Nombre | Función |
|--------|--------|---------|
| `02h` | Page Program | Escribe hasta 256 bytes en una página |
| `20h` | Sector Erase | Borra 4 KB |
| `52h` | Block Erase 32 KB | Borra 32 KB |
| `D8h` | Block Erase 64 KB | Borra 64 KB |
| `C7h` / `60h` | Chip Erase | Borra toda la memoria |

---

## 6. READ (03h) paso a paso

### 6.1 Secuencia

1. **CS baja.**
2. Enviar `03h` por MOSI.
3. Enviar **dirección de 24 bits** (3 bytes, MSB primero).
4. La Flash empieza a devolver datos por MISO.
5. **CS sube** al terminar.

### 6.2 Trama (diagrama Mermaid)

```mermaid
sequenceDiagram
    participant FPGA as FPGA (Maestro)
    participant FLASH as Flash (Esclavo)

    Note over FPGA,FLASH: CS baja
    FPGA->>FLASH: 03h
    FPGA->>FLASH: A23..A0 (24 bits)
    FLASH-->>FPGA: D0
    FLASH-->>FPGA: D1
    FLASH-->>FPGA: D2
    FLASH-->>FPGA: D3...
    Note over FPGA,FLASH: CS sube
```

### 6.3 Tabla de la trama

| Fase | MOSI (FPGA → Flash) | MISO (Flash → FPGA) |
|------|---------------------|---------------------|
| Comando | `03h` | X (no importa) |
| Dirección | A23..A0 (24 bits) | X (no importa) |
| Datos | (no se usa) | D0, D1, D2, D3… |

### 6.4 ¿Cuánta información devuelve?

**Ilimitada.** Mientras CS esté bajo, la Flash sigue sacando bytes y la dirección interna avanza sola. Al llegar al final, vuelve al inicio (**wrap around**).

> Cita de clase: *"hay que tener cuidado porque ese se sobrecribe… termina a los 64 espacios y vuelve a escribir el primero."*

---

## 7. FAST READ (0Bh) paso a paso

### 7.1 Secuencia

1. **CS baja.**
2. Enviar `0Bh` por MOSI.
3. Enviar **dirección de 24 bits**.
4. Enviar **1 byte dummy** (8 ciclos de reloj, MISO ignorado).
5. La Flash devuelve datos por MISO.
6. **CS sube.**

### 7.2 Trama (diagrama Mermaid)

```mermaid
sequenceDiagram
    participant FPGA as FPGA (Maestro)
    participant FLASH as Flash (Esclavo)

    Note over FPGA,FLASH: CS baja
    FPGA->>FLASH: 0Bh
    FPGA->>FLASH: A23..A0 (24 bits)
    FPGA->>FLASH: Dummy (8 ciclos)
    FLASH-->>FPGA: D0
    FLASH-->>FPGA: D1
    FLASH-->>FPGA: D2...
    Note over FPGA,FLASH: CS sube
```

### 7.3 Tabla de la trama

| Fase | MOSI (FPGA → Flash) | MISO (Flash → FPGA) |
|------|---------------------|---------------------|
| Comando | `0Bh` | X (no importa) |
| Dirección | A23..A0 (24 bits) | X (no importa) |
| Dummy | (nada) | X (8 ciclos vacíos) |
| Datos | (no se usa) | D0, D1, D2, D3… |

---

## 8. Preguntas resueltas

### 8.1 ¿Por qué FAST READ es más rápido si **añade** un byte dummy?

Porque el byte dummy **no es trabajo extra**, es **tiempo de preparación** para la Flash. La memoria necesita:

- Decodificar la dirección.
- Acceder a la celda de memoria.
- Cargar el dato en el registro de salida.

Ese proceso toma tiempo fijo (nanosegundos). El dummy le da ese tiempo al chip.

**La ganancia real viene de la frecuencia de reloj.**

Cálculo para leer **N = 1000 bytes**:

READ normal a 50 MHz:

$$
T_{read} = \frac{8 + 24 + 8000}{50\times 10^6} = \frac{8032}{50\times 10^6} = 160.6\ \mu s
$$

FAST READ a 104 MHz:

$$
T_{fast} = \frac{8 + 24 + 8 + 8000}{104\times 10^6} = \frac{8040}{104\times 10^6} = 77.3\ \mu s
$$

**FAST READ es ~2 veces más rápido** aunque tenga 8 ciclos extra. El dummy cuesta muy poco y a cambio se duplica la frecuencia.

> **Regla mental:** el byte dummy es la **cuota de entrada** para poder correr el reloj al doble.

### 8.2 ¿Solo se pueden leer 8 bits por comando?

**No.** Cada byte son 8 bits, pero al mantener CS bajo la Flash sigue entregando bytes y **avanza sola la dirección** (modo burst / continuous read).

### 8.3 ¿Toca enviar la trama de nuevo para leer más bits?

**No.** Solo se manda **una vez** comando + dirección. Después se sigue dando reloj y los datos salen consecutivos.

Ejemplo: leer un `uint32_t` desde `0x000100`:

```
CS baja
→ 03h
→ 00h 01h 00h   (dirección)
→ recibes byte0, byte1, byte2, byte3
→ unes: dato = (byte0<<24)|(byte1<<16)|(byte2<<8)|byte3
CS sube
```

### 8.4 ¿Qué comandos van antes o después?

- **Para leer:** no se necesita comando previo.
- **Para escribir o borrar:** primero `06h` (Write Enable).
- **Después de escribir:** revisar con `05h` (Read Status Register) si el chip terminó.

### 8.5 ¿Qué diferencia hay con I2S?

En **I2S** la frecuencia de muestreo sí importa (44.2 kHz, 48 kHz). Si se manda el audio más rápido o más lento, se escucha mal. En **SPI** la velocidad no afecta el dato, solo el tiempo que tarda.

> Cita de clase: *"Es la gran diferencia de protocolos SPI frente a todos los demás: el tiempo acá sí importa [en I2S], en los demás no."*

---

## 9. Usos prácticos

### 9.1 Almacenar imágenes o audio del juego

La Flash guarda el fondo de pantalla, sprites o el archivo de audio. La FPGA lee por SPI y luego manda a la pantalla o al DAC.

### 9.2 Leer varios bytes rápido

Usar **FAST READ** en lugar de **READ** cuando se leen bloques grandes. Para leer una sola posición pequeña, la diferencia es mínima.

### 9.3 Escritura

Antes de escribir:

1. Enviar `06h` (Write Enable).
2. Enviar `02h` (Page Program) con dirección y hasta 256 bytes.
3. Esperar con `05h` hasta que WIP = 0.

### 9.4 Borrado

- `20h` → sector de 4 KB.
- `D8h` → bloque de 64 KB.
- `C7h` → toda la memoria.

**Cuidado:** en Flash solo se borra por sectores o bloques, no bit a bit.

### 9.5 Modo burst

Si vas a leer 512 bytes consecutivos, manda la dirección **una sola vez** y sigue dando reloj.

---

## 10. Diagramas incluidos en este directorio

| Archivo | Descripción |
|---------|-------------|
| `read_03h.png` | Trama del comando READ |
| `fast_read_0bh.png` | Trama del comando FAST READ |
| `conexion_fpga_flash.png` | Diagrama físico FPGA ↔ Flash |
| `comparacion_tiempos.png` | Gráfica READ vs FAST READ |
| `comandos_spi_flash.png` | Tabla resumen de comandos |

> Los diagramas se generan con [WaveDrom](https://wavedrom.com) siguiendo la recomendación del profesor de usar herramientas para hacer las formas de onda.

---

## 11. Dependencias y enlaces

- Issue [#1](https://github.com/noNintendo2026/spi_flash_ctrl/issues/1)
- Issue [#2](https://github.com/noNintendo2026/spi_flash_ctrl/issues/2)
- Issue [#3](https://github.com/noNintendo2026/spi_flash_ctrl/issues/3)
- Wiki del curso (actualizar aquí)
- Datasheet de la Flash usada (agregar número de parte específico)

---

## 12. Criterios de aceptación

- [ ] Revisión por @camahechag-bars.
- [ ] Revisión por @jcanizalez16.
- [ ] Revisión por @edfrancos.
- [ ] Toda la información de los issues #1, #2 y #3 está incluida.
- [ ] Diagramas PNG en este directorio.
- [ ] Enlaces a los archivos pertinentes funcionando.

---

## 13. Referencias

- Clase de SPI Flash (transcripción del curso, 2026-09-29).
- Datasheet típico de Flash SPI (ejemplo: W25Q128JV, Winbond).
- Documentación de WaveDrom.

---

**Última actualización:** 2026-10-06  
**Autores:** @camahechag-bars, @jcanizalez16, @edfrancos