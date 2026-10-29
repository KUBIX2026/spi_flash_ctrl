# Protocolo SPI

## 1. Explicación general del protocolo

### 1.1 ¿Qué es SPI?

SPI (**Serial Peripheral Interface**) es un protocolo de comunicación **sincrónico, serie y full-duplex**. Un dispositivo **maestro** (en nuestro caso la FPGA) controla a uno o varios **esclavos** (en este caso, la memoria Flash).

### 1.2 Señales del bus

El protocolo cuenta con cuatro señales principales de conexión:

| Señal       | Nombre completo            | Función                               |
| ----------- | -------------------------- | ------------------------------------- |
| `SCLK`      | Serial Clock               | Reloj generado por el maestro         |
| `MOSI`      | Master Out Slave In        | Datos del maestro hacia el esclavo    |
| `MISO`      | Master In Slave Out        | Datos del esclavo hacia el maestro    |
| `CS` / `SS` | Chip Select / Slave Select | Activo en bajo. Selecciona el esclavo |

### 1.3 Características principales

* **Protocolo sincrónico:** los datos se leen en los flancos del reloj. La configuración del flanco de lectura y el estado base del reloj determinan los cuatro modos de operación del protocolo.
* **Comunicación full-duplex:** el protocolo soporta comunicación bidireccional simultánea mediante las líneas independientes `MOSI` y `MISO`.
* **Generación del reloj:** la señal de reloj es generada por el maestro.
* **Estabilidad de los datos:** cuando ocurre el flanco utilizado para leer un dato, la información debe encontrarse estable.
* **Velocidad de transmisión:** la frecuencia utilizada debe respetar los tiempos de operación establecidos para la memoria Flash.

### 1.4 Modos SPI (CPOL / CPHA)

La comunicación SPI puede utilizar cuatro configuraciones determinadas por **CPOL** y **CPHA**:

| Modo | CPOL | CPHA | Flanco de muestreo |
| ---- | ---- | ---- | ------------------ |
| 0    | 0    | 0    | Subida             |
| 1    | 0    | 1    | Bajada             |
| 2    | 1    | 0    | Bajada             |
| 3    | 1    | 1    | Subida             |

En memorias Flash se utiliza normalmente **Modo 0** o **Modo 3**, dependiendo de la memoria específica.

El dato cambia en un flanco del reloj y se lee en el otro. Por lo tanto, en el momento del flanco de lectura, el dato debe permanecer estable.

### 1.5 Estructura general de la comunicación

Toda transacción con la Flash sigue, de manera general, el siguiente patrón:

```text
CS baja → [Comando] → [Dirección] → [Dummy, si aplica] → [Datos] → CS sube
```

Los elementos principales de la transacción son:

* **Comando:** 1 byte que indica la operación que se desea realizar.
* **Dirección:** 3 bytes (24 bits) que indican la posición de memoria.
* **Dummy:** 0 o más bytes de relleno utilizados por algunos comandos.
* **Datos:** información que se transmite o recibe.

La secuencia general de la comunicación es:

1. `CS` baja al inicio de la transacción.
2. El maestro genera el reloj.
3. Se transmite el comando.
4. Se transmite la dirección cuando la operación la requiere.
5. Se transmiten ciclos dummy cuando el comando los requiere.
6. Se transmiten o reciben los datos.
7. `CS` sube al finalizar la transacción.


### 1.6 Estructura interna de la memoria 

Como informacion extra, es necesario comprender como se divide la estructura interna de la memoria flash, ya que esto es importante para entender la funcionalidad de las direcciones que se envia en multiples comandos.



![Estructura de memoria](cores/spi_flash/diagramas/Estructura_memoria.png)


Típicamente, estas memorias se dividen en 3 niveles, tal como se muestra en la imagen anterior. La memoria completa tiene una capacidad de 16 MiB, la cual se divide en 256 bloques, cada uno de 64 KB. Cada bloque se divide en 16 sectores, cada uno de 4 KB. Cada sector se vuelve a dividir en 16 paginas, cada una de 256 Bytes y estas ultimas se dividen en los ya mencionados 256 Bytes. Una imagen mas representativa se muestra a continuación.

![Estructura detallada de memoria](cores/spi_flash/diagramas/Estructura_detallada_de_memoria.png)


Las direcciones de memoria que se envian desde el maestro hacia el esclavo se dividen en 4 partes (esto se refiere únicamente a la distribución de la información, las direcciones se envian de forma continua bit tras bit).

![Estrcutura de la direccion enviada](cores/spi_flash/diagramas/Estructura_direcciones.png)




---

# 2. Comandos de estado y control
Estos comandos gestionan los registros de estado internos de la memoria Flash, que permiten consultar el estado del dispositivo y configurar determinados parámetros de funcionamiento.

### 2.2.1 Write Enable — WREN (0x06)

**Propósito.** Establecer en `1` el bit **Write Enable Latch (WEL)** de la memoria Flash. Los comandos `PAGE_PROGRAM`, `SECTOR_ERASE`, `BLOCK_ERASE`, `CHIP_ERASE` y `WRSR` se ignoran si `WEL` no está establecido en `1`. Por tanto, `WREN` es un paso obligatorio antes de ejecutar cualquier operación de escritura o borrado.

**Secuencia de comunicación.**

1. `CS` pasa a nivel bajo.
2. El maestro transmite el byte `06h` por `MOSI`, comenzando por el bit más significativo.
3. `CS` pasa a nivel alto.

Durante toda la transacción, la memoria Flash mantiene la salida `MISO` en alta impedancia. No se devuelve ningún dato.

**Tabla de tramas.**

| Fase | `MOSI` (FPGA → Flash) | `MISO` (Flash → FPGA) |
|---|---|---|
| Código de operación | `06h` | Alta impedancia (sin conducción) |

**Diagrama de tiempos.**

![Forma de onda de WREN (06h)](diagrams/protocol/waveform_wren.svg)

*Figura 1. WREN (`06h`). `CS` delimita la transacción, `CLK` permanece en nivel bajo durante el reposo (Modo 0), `MOSI` transmite `06h` = `0000 0110` y `MISO` permanece en alta impedancia.*

**Referencia en la hoja de datos.** Sección 8.2.1, Figura 5.

**Nota de diseño.** Se trata de una transacción de un solo byte, sin datos de retorno. Desde la interfaz CSR, puede ejecutarse configurando `CMD=WREN`, activando `START` y esperando a que se indique `DONE`. No requiere una FIFO.

### 2.2.2 Write Disable — WRDI (0x04)

**Propósito.** Establecer en `0` el bit `WEL`. Después de ejecutar `WRDI`, la memoria Flash rechaza los comandos de escritura y borrado hasta que se emita un nuevo `WREN`. No es necesario ejecutarlo después de una operación `PAGE_PROGRAM` normal, ya que la propia memoria borra `WEL` al completar la operación. Sin embargo, puede utilizarse como medida de seguridad después de una condición de error.

**Secuencia de comunicación.**

1. `CS` pasa a nivel bajo.
2. El maestro transmite `04h` por `MOSI`.
3. `CS` pasa a nivel alto.

**Tabla de tramas.**

| Fase | `MOSI` (FPGA → Flash) | `MISO` (Flash → FPGA) |
|---|---|---|
| Código de operación | `04h` | Alta impedancia (sin conducción) |

**Diagrama de tiempos.**

![Forma de onda de WRDI (04h)](diagrams/protocol/waveform_wrdi.svg)

*Figura 2. WRDI (`04h`). Presenta la misma estructura temporal que WREN, pero utiliza el código de operación `04h` = `0000 0100`.*

**Referencia en la hoja de datos.** Sección 8.2.3, Figura 7.

**Nota de diseño.** Su estructura es idéntica a la de WREN. Puede reutilizarse la máquina de estados finitos (FSM) de transacciones de un solo byte, cambiando únicamente el código de operación.

### 2.2.3 Read Status Register 1 — RDSR1 (0x05)

**Propósito.** Leer el **registro de estado 1** (*Status Register 1*) de 8 bits, que contiene los dos indicadores que necesita consultar el firmware: `BUSY/WIP` (Write in Progress) y `WEL`. Este comando permite determinar si una operación de programación o borrado iniciada previamente ha finalizado.

**Secuencia de comunicación.**

1. `CS` pasa a nivel bajo.
2. El maestro transmite `05h` por `MOSI`.
3. A partir de los siguientes ciclos de reloj, la memoria Flash transmite los bits `S7…S0` por `MISO`.
4. El maestro puede continuar generando ciclos de reloj para recibir repetidamente el byte de estado.
5. `CS` pasa a nivel alto para finalizar la transacción.

**Tabla de tramas.**

| Fase | `MOSI` (FPGA → Flash) | `MISO` (Flash → FPGA) |
|---|---|---|
| Código de operación | `05h` | Alta impedancia (sin conducción) |
| Datos (repetibles) | No relevante | `S7…S0`, seguido de `S7…S0`, etc. |

**Diagrama de tiempos.**

![Forma de onda de RDSR1 (05h)](diagrams/protocol/waveform_rdsr1.svg)

*Figura 3. RDSR1 (`05h`). Después de recibir el código de operación, la memoria Flash transmite el registro de estado por `MISO` durante los ciclos de reloj posteriores.*

**Referencia en la hoja de datos.** Sección 8.2.4, Figura 8a.

**Definición de los bits del registro de estado 1 (bits relevantes).**

| Bit | Nombre | Significado | Representación en el controlador |
|---|---|---|---|
| `S0` | BUSY / WIP | Vale `1` mientras se ejecuta una operación interna de programación o borrado. | `WIP` en `STATUS` |
| `S1` | WEL | Vale `1` cuando se ha ejecutado WREN y el permiso de escritura todavía no se ha consumido. | `WEL` en `STATUS` |
| `S2..S7` | BP/TB/SEC/SRP0 | Configuración de protección contra escritura. | Ignorados |

**Nota de diseño.** RDSR1 es el comando utilizado para el **sondeo** (*polling*). El firmware lo ejecuta repetidamente hasta que `WIP = 0`. La lógica RTL no implementa este bucle: expone transacciones individuales y permite que el firmware gestione las consultas sucesivas.

### 2.2.4 Read Status Register 2 — RDSR2 (0x35)

**Propósito.** Leer el **registro de estado 2** (*Status Register 2*) de 8 bits, que contiene los indicadores `QE` (*Quad Enable*), `CMP` (*Complement Protect*), `SRL` (*Status Register Lock*), `LB1..LB3` (bits de bloqueo de la región OTP) y `SUS` (estado de suspensión). En este proyecto, `SUS` y `QE` tienen interés informativo; los demás bits se ignoran. Se incluye RDSR2 para completar el conjunto de comandos y facilitar una posible habilitación futura del modo Quad SPI.

**Secuencia de comunicación.**

1. `CS` pasa a nivel bajo.
2. El maestro transmite `35h` por `MOSI`.
3. A partir de los siguientes ciclos de reloj, la memoria Flash transmite los bits `S15…S8` por `MISO`.
4. El maestro puede continuar generando ciclos de reloj para recibir repetidamente el byte de estado.
5. `CS` pasa a nivel alto para finalizar la transacción.

**Tabla de tramas.**

| Fase | `MOSI` (FPGA → Flash) | `MISO` (Flash → FPGA) |
|---|---|---|
| Código de operación | `35h` | Alta impedancia (sin conducción) |
| Datos (repetibles) | No relevante | `S15…S8`, seguido de `S15…S8`, etc. |

**Diagrama de tiempos.**

![Forma de onda de RDSR2 (35h)](diagrams/protocol/waveform_rdsr2.svg)

*Figura 4. RDSR2 (`35h`). Presenta la misma estructura temporal que RDSR1. Solo cambia el código de operación (`35h` = `0011 0101`) y el byte devuelto corresponde al registro de estado 2.*

**Referencia en la hoja de datos.** Sección 8.2.4, Figura 8a (misma familia de instrucciones que RDSR1).

**Definición de los bits relevantes del registro de estado 2.**

| Bit | Nombre | Significado | Uso en el proyecto |
|---|---|---|---|
| `S9` | QE | Habilitación del modo Quad (debe valer `1` para utilizarlo). | No utilizado; solo se emplea SPI estándar. |
| `S14` | CMP | Protección complementaria. | Ignorado. |
| `S15` | SUS | Estado de suspensión. | Solo lectura; informativo. |

**Nota de diseño.** Utiliza la misma FSM que RDSR1; únicamente cambia el código de operación. El registro `DATA` expone el byte recibido al firmware.

### 2.2.5 Write Status Register — WRSR (0x01)

**Propósito.** Escribir el byte del registro de estado 1. En este proyecto, WRSR se incluye para completar el conjunto de comandos y permitir futuras ampliaciones, ya que no se configuran los bits de protección contra escritura. También podría resultar útil en una futura migración a Quad SPI, si fuera necesario habilitar `QE`. Este comando requiere ejecutar previamente `WREN`.

**Secuencia de comunicación.**

1. `CS` pasa a nivel bajo.
2. El maestro transmite `01h` por `MOSI`.
3. El maestro transmite por `MOSI` el nuevo byte de estado.
4. `CS` pasa a nivel alto, una vez completada la transmisión del byte.

**Tabla de tramas.**

| Fase | `MOSI` (FPGA → Flash) | `MISO` (Flash → FPGA) |
|---|---|---|
| Código de operación | `01h` | Alta impedancia (sin conducción) |
| Datos | `S7…S0` | Alta impedancia (sin conducción) |

**Diagrama de tiempos.**

![Forma de onda de WRSR (01h)](diagrams/protocol/waveform_wrsr.svg)

*Figura 5. WRSR (`01h`). Se transmiten dos bytes por `MOSI`: el código de operación `01h`, seguido del byte de estado que se desea escribir. `MISO` permanece en alta impedancia.*

**Referencia en la hoja de datos.** Sección 8.2.5, Figura 9a.

**Nota de diseño.** Su estructura es similar a la de `PAGE_PROGRAM`, pero no transmite bytes de dirección y requiere exactamente un byte de datos. El controlador puede implementarlo como un caso particular de la FSM de escritura, configurando `CMD=WRSR`, `LEN=1` y `ADDR=0`.



---

# 3. Comandos de lectura

Esta sección presenta los comandos utilizados para obtener información almacenada en la memoria Flash. Se documentará posteriormente la estructura de cada comando y su respectiva forma de onda.

### 3.1 Índice de comandos

| Código | Comando               | Descripción                     |
| ------ | --------------------- | ------------------------------- |
| `0x03`  | READ                  | Lectura de datos                |
| `0x0B`  | FAST READ             | Lectura rápida con ciclos dummy |

<!--| `3Bh`  | FAST READ DUAL OUTPUT | Lectura mediante dos líneas     |
| `BBh`  | FAST READ DUAL I/O    | Lectura Dual I/O                |
| `6Bh`  | FAST READ QUAD OUTPUT | Lectura mediante cuatro líneas  |
| `EBh`  | FAST READ QUAD I/O    | Lectura Quad I/O                |-->

### 3.2 Documentación

Para cada comando se incluirá:

* Secuencia de utilización.
* Comando, dirección y ciclos dummy cuando correspondan.
* Datos obtenidos mediante la lectura.
* Forma de onda correspondiente.

### 3.3 READ (0x03) paso a paso

#### 3.3.1 Secuencia

1. **CS baja.**
2. Enviar `0x03` por MOSI.
3. Enviar **dirección de 24 bits** (3 bytes, MSB primero).
4. La Flash empieza a devolver datos por MISO.
5. **CS sube** al terminar.

#### 3.3.2 Trama (diagrama Mermaid)

```mermaid
sequenceDiagram
    participant FPGA as FPGA (Maestro)
    participant FLASH as Flash (Esclavo)

    Note over FPGA,FLASH: CS baja
    FPGA->>FLASH: 0x03
    FPGA->>FLASH: A23..A0 (24 bits)
    FLASH-->>FPGA: D0
    FLASH-->>FPGA: D1
    FLASH-->>FPGA: D2
    FLASH-->>FPGA: D3...
    Note over FPGA,FLASH: CS sube
```

#### 3.3.3 Tabla de la trama

| Fase | MOSI (FPGA → Flash) | MISO (Flash → FPGA) |
|------|---------------------|---------------------|
| Comando | `0x03` | X (no importa) |
| Dirección | A23..A0 (24 bits) | X (no importa) |
| Datos | (no se usa) | D0, D1, D2, D3… |

#### 3.3.4 ¿Cuánta información devuelve?

**Ilimitada.** Mientras CS esté bajo, la Flash sigue sacando bytes y la dirección interna avanza sola. Al llegar al final, vuelve al inicio (**wrap around**).

<!--> Cita de clase: *"hay que tener cuidado porque ese se sobrecribe… termina a los 64 espacios y vuelve a escribir el primer

### 3.4 FAST READ (0x0B) paso a paso

#### 3.4.1 Secuencia

1. **CS baja.**
2. Enviar `0x03` por MOSI.
3. Enviar **dirección de 24 bits**.
4. Enviar **1 byte dummy** (8 ciclos de reloj, MISO ignorado).
5. La Flash devuelve datos por MISO.
6. **CS sube.**

#### 3.4.2 Trama (diagrama Mermaid)

```mermaid
sequenceDiagram
    participant FPGA as FPGA (Maestro)
    participant FLASH as Flash (Esclavo)

    Note over FPGA,FLASH: CS baja
    FPGA->>FLASH: 0x0B
    FPGA->>FLASH: A23..A0 (24 bits)
    FPGA->>FLASH: Dummy (8 ciclos)
    FLASH-->>FPGA: D0
    FLASH-->>FPGA: D1
    FLASH-->>FPGA: D2...
    Note over FPGA,FLASH: CS sube
```

#### 3.4.3 Tabla de la trama

| Fase | MOSI (FPGA → Flash) | MISO (Flash → FPGA) |
|------|---------------------|---------------------|
| Comando | `0x0B` | X (no importa) |
| Dirección | A23..A0 (24 bits) | X (no importa) |
| Dummy | (nada) | X (8 ciclos vacíos) |
| Datos | (no se usa) | D0, D1, D2, D3… |

### 3.5 Formas de onda
<img width="947" height="275" alt="image" src="https://github.com/user-attachments/assets/5bb560f6-d2a7-4d1f-8647-7cf5705d2ca2" />


### 3.6 Preguntas resueltas

#### 3.6.1 ¿Por qué FAST READ es más rápido si **añade** un byte dummy?

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

#### 3.6.2 ¿Solo se pueden leer 8 bits por comando?

**No.** Cada byte son 8 bits, pero al mantener CS bajo la Flash sigue entregando bytes y **avanza sola la dirección** (modo burst / continuous read).

#### 3.6.3 ¿Toca enviar la trama de nuevo para leer más bits?

**No.** Solo se manda **una vez** comando + dirección. Después se sigue dando reloj y los datos salen consecutivos.

Ejemplo: leer un `uint32_t` desde `0x000100`:

```
CS baja
→ 0x03
→ 00h 01h 00h   (dirección)
→ recibes byte0, byte1, byte2, byte3
→ unes: dato = (byte0<<24)|(byte1<<16)|(byte2<<8)|byte3
CS sube
```

#### 3.6.4 ¿Qué comandos van antes o después?

- **Para leer:** no se necesita comando previo.
<!-- **Para escribir o borrar:** primero `06h` (Write Enable).
- **Después de escribir:** revisar con `05h` (Read Status Register) si el chip terminó.-->

<!--### 8.5 ¿Qué diferencia hay con I2S?

En **I2S** la frecuencia de muestreo sí importa (44.2 kHz, 48 kHz). Si se manda el audio más rápido o más lento, se escucha mal. En **SPI** la velocidad no afecta el dato, solo el tiempo que tarda.

> Cita de clase: *"Es la gran diferencia de protocolos SPI frente a todos los demás: el tiempo acá sí importa [en I2S], en los demás no."*-->
---

# 4. Comandos de escritura y borrado

Esta sección presenta los comandos utilizados para modificar el contenido de la memoria Flash, incluyendo las operaciones de programación y borrado.

### 4.1 Índice de comandos

| Código      | Comando      | Función                  |
| ----------- | ------------ | ------------------------ |
| `0x02`       | Page Program | Programa hasta 256 bytes |
| `0x20`       | Sector Erase | Borra un sector de 4 KB  |
| `0xD8`       | Block Erase  | Borra un bloque de 64 KB |
| `0xC7` | Chip Erase   | Borra toda la memoria    |

La estructura de estas 4 funcionalidades poseen la misma estructura base, se genera la señal de reloj desde el maestro, se cambia el `CS` a `0` para indicarle a la memoria que debe escuchar la información que le llegue, la cual tiene la estructura Opencode->direccion->información (únicamente en el comando `0x02`), y la memoria ejecuta la accion indicada. El uso completo de estos comando se explica mas adelante.

#### 4.2.1 Page Program

El comando `0x02` permite escribir en la memoria hasta un total de 256 Bytes, estos se empezaran a escribir a parir de la direccion dada por el maestro, si los datos enviados hacia el esclavo, llegan al final de una pagina, los siguientes es escribiran al inicio de la misma, y no pasaran a la siguiente pagina.

![Formas de onda para el comando Page program](cores/spi_flash/diagramas/Comando_page_program.png)



#### 4.2.2 Sector Erase

El comando `0x20` permite borrar un sector completo de la memoria, correspondiente a un tamaño de 4 KB. El borrado se realiza a partir de la dirección indicada por el maestro, tomando como referencia el sector al que pertenece dicha dirección.

![Formas de onda para el comando Sector Erase](cores/spi_flash/diagramas/Comando_sector_erase.png)

#### 4.2.3 Block Erase

El comando `0xD8` permite borrar un bloque completo de la memoria, correspondiente a un tamaño de 64 KB. El borrado se realiza a partir de la dirección indicada por el maestro, tomando como referencia el bloque al que pertenece dicha dirección.

![Formas de onda para el comando Block Erase](cores/spi_flash/diagramas/Comando_block_erase.png)

#### 4.2.4 Chip Erase

El comando `0xC7` permite borrar completamente el contenido de la memoria, eliminando los datos almacenados en todos sus sectores y bloques. A diferencia de los comandos anteriores, no requiere una dirección específica para determinar la zona que será borrada.

![Formas de onda para el comando Chip Erase](cores/spi_flash/diagramas/Comando_chip_erase.png)


### 4.3 Uso práctico de los comandos de escritura y borrado

El uso práctico de estos cuatro comandos comparte una estructura general. Antes de ejecutar cualquier operación de escritura o borrado, se debe enviar el comando `0x06` (Write Enable), el cual habilita la memoria para realizar este tipo de operaciones.

Posteriormente, se envía el comando correspondiente a la operación que se desea realizar: `0x02` (Page Program), `0x20` (Sector Erase), `0xD8` (Block Erase) o `0xC7` (Chip Erase).

Una vez iniciada la operación, la memoria necesita un tiempo para completarla. Durante este proceso, se utiliza el comando `0x05` (Read Status Register) para consultar el estado de la memoria. Este comando puede repetirse hasta verificar que la operación haya finalizado. La información del registro de estado es enviada por la memoria hacia el maestro mediante la línea `MISO`. Este proceso se ilustra a continuación.

![Uso práctico de los comandos de escritura y borrado](cores/spi_flash/diagramas/Uso_practico_comandos_escritura_borrado.png)


---

# 5. Diagrama de flujo general del protocolo

Esta sección presenta el **diagrama de flujo general del protocolo SPI para la memoria Flash**, mostrando de manera global la secuencia que sigue una transacción desde la activación de `CS` hasta su finalización.

El diagrama de flujo utilizado para representar esta secuencia se encuentra en los archivos gráficos incluidos en este directorio.
![diagrama de flujo de todos los usos de la memoria flash mediante SPI](cores/spi_flash/diagramas/Diagrama_flujo_uso_practico.png)


---

# 6. Diagramas internos

Basandonos en lo explicado en este README y considerando la arquitectura propuesta por el profesor para la comunicacion entre el RV32I y la memoria flash, este seria el diagrama de entradas y salidas de su controlador.

![Diagrama de entradas y salidas](cores/spi_flash/diagramas/Diagrama_entradas_salidas.png)


# 7. Diagramas de flujo

Basándonos en el diagrama de entradas y salidas, y en las necesidades y limitaciones tanto del RV32I como del protocolo SPI para memorias flash, a continuación se presentan los diagramas de flujo de las acciones disponibles para el RV32I mediante el módulo de control de la memoria flash.

## 7.1 Lectura de información pedida por la CPU

El siguiente diagrama de flujo explica la secuencia de pasos necesarios para realizar una lectura de datos entre 2 direcciones de memoria escogidas por la CPU.

![diagrama de flujo para lectura de información](cores/spi_flash/diagramas/diagrama_flujo_lectura_flash_simple.png)

Para ver los registros de consulta de la CPU y una descripción más detallada de las variables del diagrama, vaya [aquí](cores/spi_flash/diagramas/diagrama_flujo_lectura_flash.pdf) o a [aquí](cores/spi_flash/diagramas/diagrama_flujo_lectura_flash.png) (formato png).

El siguiente diagrama explica la esecuencia para el comando FAST READ (0X0B)
<img width="979" height="3483" alt="685469400-82b0364c-0aa4-4d93-8bfc-f7dcefd1feeb" src="https://github.com/user-attachments/assets/74e33cc5-7207-4251-9670-7ecb750ddd0b" />


## 7.2 Borrado de información pedido por la CPU

El siguiente diagrama de flujo explica la secuencia de pasos necesarios para borrar un sector (4 KB), un bloque (64 KB) o el chip completo. Incluye la habilitación de escritura en la flash (WREN), el envío del comando de borrado y la espera a que la flash termine, con una bandera de error si se agota el tiempo máximo de espera.

![diagrama de flujo para borrado de información](cores/spi_flash/diagramas/diagrama_flujo_borrado_flash_simple.png)

Para ver los registros de consulta de la CPU y una descripción más detallada de las variables del diagrama, vaya [aquí](cores/spi_flash/diagramas/diagrama_flujo_borrado_flash.pdf) o a [aquí](cores/spi_flash/diagramas/diagrama_flujo_borrado_flash.png) (formato png).

# 8. Información extra

### Estructura del repo

- `cores/spi_flash/diagramas/` — diagramas de flujo, bloques y protocolo de comunicación

### Equipo

- Jhon Dairon Canizalez Arias
- Edwin Franco Sanchez
- Carlos Alfonso Mahecha Gonzalez




# Esta plantilla no es de mi propiedad.

Toda la documentación y diseño de esta plantilla se encuentra dentro del siguiente repositorio el cual **no me pertenece** y es propiedad del usuario [@cicamargo](https://github.com/cicamargoba).

[Github de digital_UN](https://github.com/cicamargoba/digital_UN/tree/main/2026_1)


