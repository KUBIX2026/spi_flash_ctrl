# Planificación y Seguimiento — SPI-Flash Controller (Equipo D)

Documento operativo del equipo. Define enlaces, convenciones de trabajo
y el roadmap hacia la propuesta de arquitectura. La especificación técnica
vive en [`docs/01_specification/`](../docs/01_specification/).

## 1. Enlaces

| Recurso | Enlace |
|---|---|
| Organización GitHub | [KUBIX2026](https://github.com/KUBIX2026) |
| Repositorio de este equipo | [spi_flash_ctrl](https://github.com/KUBIX2026/spi_flash_ctrl) |
| Repositorio del curso (upstream) | [digital_UN](https://github.com/cicamargoba/digital_UN) |
| Plantilla de issues | [`template/.github/ISSUE_TEMPLATE/tarea.yml`](https://github.com/cicamargoba/digital_UN/tree/main/template/.github/ISSUE_TEMPLATE) |
| Rúbrica de evaluación | [`rubrica_evaluacion_proyecto.md`](https://github.com/cicamargoba/digital_UN/blob/main/rubrica_evaluacion_proyecto.md) |

## 2. Convenciones Git

### Ramas

| Rama | Uso |
|---|---|
| `main` | Estable. Solo recibe merges vía Pull Request. |
| `reorganzation` | Reorganización estructural del repo (temporal). |
| `feat/<modulo>` | Nueva funcionalidad. |
| `fix/<bug>` | Corrección. |
| `docs/<tema>` | Solo documentación. |

### Commits (Conventional Commits)

Formato: `<tipo>(<alcance>): <descripción>`

| Tipo | Uso |
|---|---|
| `feat` | Nueva funcionalidad RTL o firmware |
| `fix` | Corrección de bug |
| `docs` | Documentación |
| `rtl` | Cambios exclusivos de RTL |
| `firmware` | Cambios exclusivos de firmware |
| `test` | Testbench o tests de firmware |
| `chore` | Tareas de mantenimiento |

Ejemplos reales de este repo:

- `docs(spec): add SPI flash command set table`
- `rtl(spi_master): implement shift register`
- `firmware(driver): add write_enable() helper`

## 3. Gestión de tareas

Cada tarea se crea como *issue* en este repositorio y se agrega al
GitHub Project de la organización. Campos obligatorios según el PBL:

| Campo | Descripción |
|---|---|
| Tarea | Actividad concreta |
| Responsable | Estudiante asignado |
| Fecha de inicio | Prevista |
| Fecha de entrega | Prevista |
| Dependencias | Tareas o módulos previos |
| Estado | Pendiente / En desarrollo / Bloqueada / Terminada |
| Evidencia | Enlace a commit, simulación o documento |

## 4. Roadmap del equipo D (SPI-Flash)

| Fase | Entregable | Archivo(s) | Estado |
|---|---|---|---|
| 0 | Reorganización del repo | `README.md`, `planificacion/`, `rtl/Makefile` | ✅ |
| 1 | Requisitos funcionales | `docs/01_specification/requirements.md` | 🔄 |
| 2 | Especificación del protocolo SPI | `docs/01_specification/spi_protocol.md` | 🔄 |
| 3 | Especificación de registros CSR | `docs/01_specification/csr_specification.md` | ⏳ |
| 4 | Flowchart de operación | `docs/01_specification/operation_flow.md` + SVG | ⏳ |
| 5 | **Propuesta de arquitectura** | `docs/02_architecture.md` + `diagrams/architecture/` | ⏳ |
| 6 | ASM / FSM | `docs/03_design.md` + `diagrams/design/` | ⏳ |
| 7 | RTL + testbench | `rtl/` | ⏳ |
| 8 | Verificación | `docs/04_verification.md` + `simulation/` | ⏳ |
| 9 | Integración SoC | `docs/05_integration.md` + `integration/` | ⏳ |

## 5. Checkpoints del curso

| # | Checkpoint | Fases del roadmap |
|---|---|---|
| 1 | Flowchart + especificación CSR | Fases 1–4 |
| 2 | ASM del datapath y control | Fase 6 |
| 3 | RTL simulado (iverilog / GTKWave) | Fases 7–8 |
| 4 | Integración en hardware | Fase 9 |
| 5 | Demo final | — |

## 6. Convenciones de evidencia

Toda tarea terminada debe enlazar al menos uno de los siguientes:

- commit o Pull Request con el código
- captura de GTKWave o archivo `rtl/waves/*.gtkw`
- documento en `docs/`
- diagrama en `diagrams/` o SVG exportado

---

*Última actualización: 2026-10-XX*