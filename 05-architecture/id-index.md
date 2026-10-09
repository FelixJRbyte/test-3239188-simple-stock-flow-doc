# Índice de identificadores (D, DP, T, A, H, FK, Q)

> **Aviso de trazabilidad.** `spec/data-model.md` cita estos identificadores, pero
> los documentos que los definen **no se entregan** (`plan.md` §1, `tasks.md`,
> `HANDOFF-TECNICO.md`, `spec.md`). **Ninguna definición de aquí es un registro
> original.** Cada una es una reconstrucción a partir del uso en el modelo:
> - **[reconstruido]**: el modelo la nombra o la describe de forma que permite inferir su sentido.
> - **[derivado]**: se deduce del contexto; el sentido exacto puede variar.
> - **sin evidencia**: el modelo no permite inferirla; no se inventa.

## D-xx — decisiones técnicas (originales en `plan.md` §1, no entregado)

| ID | Significado reconstruido | Evidencia en el modelo | Marca |
|---|---|---|---|
| D-01 | Sin evidencia | — | sin evidencia |
| D-02 | Sin evidencia | — | sin evidencia |
| D-03 | Propiedades sombra: algo (p. ej. `deleted_at`) existe en el motor sin propiedad en el agregado | §3 (`deleted_at`) | [derivado] |
| D-04 | Concurrencia optimista con `xmin` como testigo; Q3 es el punto de contención | §3 (`xmin`), §6.1 | [reconstruido] |
| D-05 | Sistema monomoneda; ninguna columna de moneda | §1, §3, §12 | [reconstruido] |
| D-06 | El reporte se calcula en el motor por un puerto de lectura y no se persiste; la categoría se congela sin FK | §1, §3, §6.1 | [reconstruido] |
| D-07 | Los objetos de valor no tienen tabla; viven en la fila de su dueño | §2 | [reconstruido] |
| D-08 | La imagen del producto es una clave opaca de almacenamiento externo | §1, §2.2, §7.1 | [reconstruido] |
| D-09 | El hash de clave lo produce un puerto; el dominio nunca ve la clave en claro | §2.5, §9.2 | [reconstruido] |
| D-10 | Cinco categorías fijas, sembradas y sin mantenimiento. Además se cita en §9.2 junto a D-09 para la creación del admin inicial por el arranque | §1, §9.1, §9.2 | [derivado]; el doble uso es una duda abierta |

## DP-xx — decisiones del propietario

| ID | Significado reconstruido | Evidencia | Marca |
|---|---|---|---|
| DP-01 | Criterio de desempate para el nombre de producto en el reporte; H-1 rechaza el criterio «el más reciente» que hoy usa | §11.1 | [derivado] |
| DP-02 | El reporte no se desglosa por vendedor | §6.3, §7.1 | [reconstruido] |
| DP-03 | Producto: nombre, precio, stock, categoría e imagen, y nada más | §1, §8 | [reconstruido] |
| DP-04 | Nadie otorga el rol `admin` en ejecución; un administrador da de alta vendedores y el despliegue provisiona `admin` desde el entorno | §11 (H-3), §13 D-3 | [reconstruido] |

## T-xx — tareas (originales en `tasks.md`, no entregado)

| ID | Significado reconstruido | Estado según el modelo | Marca |
|---|---|---|---|
| T-02 | Esquema inicial, corrección de acento y renombrado a singular | Aplicada (§3.2) | [reconstruido] |
| T-05 | Guarda explícita de moneda en `Sale.AddItem` | Pendiente («deuda barata», §2.3) | [reconstruido] |
| T-09 | Baja lógica: columna `deleted_at` y filtro global | **Saldada** según §13 D-1 (2026-09-20). §6.3 y §7.1 aún la llaman pendiente (DISC-08) | [reconstruido] |
| T-10 | `xmin` como propiedad sombra y `CHECK` de stock no negativo | Aplicada (§3, §3.2) | [reconstruido] |
| T-11 | `sale_item.category_name` congelado | **Estado en conflicto** (DISC-03) | [reconstruido] |
| T-12 | `sold_by` → `sold_by_username` y `sold_by_user_id` con FK-4 | **Pendiente** (§13, cierre) | [reconstruido] |
| T-13 | Índices de acceso (parcial de `product`, trigramas) y `pg_trgm` | Pendiente; el compuesto único solapa con T-20 (DISC-06) | [reconstruido] |
| T-20 | Llevar invariantes «solo dominio» al motor | **Parcial**: `sale_id NOT NULL`, índice único y FK-3 según §13 D-2; los cinco `CHECK` sin confirmar (DISC-05) | [reconstruido] |
| T-01, T-03, T-04, T-06 a T-08, T-14 a T-19 | Sin evidencia | — | sin evidencia |

## A-x — defectos medidos

| ID | Significado reconstruido | Estado | Marca |
|---|---|---|---|
| A-1 | El alta de usuarios era anónima | **Cerrado** según §13 D-3 y §11 H-3 (sin token → 401; `seller` → 403; no verificado aquí). §9.2 aún dice que está rota (DISC-08) | [reconstruido] |
| A-7 | Defecto medido del desempate del nombre de producto en el reporte (`HANDOFF-TECNICO.md` §6.1, no entregado) | Abierto por instrucción del propietario (§11.1) | [derivado] |
| A-2 a A-6 | Sin evidencia | — | sin evidencia |

## H-x — huecos con dueño (§11)

| ID | Significado | Estado | Marca |
|---|---|---|---|
| H-1 | Qué `category_name` gana en el reporte tras una recategorización | **Cerrado** (2026-09-20): se agrupa por el valor congelado; puede dar más de una fila por producto (§11.1) | [reconstruido] |
| H-2 | Retención del binario de imagen huérfano; sin proceso de limpieza | **Abierto** (propietario; operativo) | [reconstruido] |
| H-3 | Quién otorga el rol `admin` | **Cerrado** (2026-09-20) por DP-04 | [reconstruido] |

## Otros identificadores que usa el modelo

| ID | Significado | Dónde está definido |
|---|---|---|
| FK-1 a FK-4 | Cuatro claves foráneas de la política de §5 | Modelo §5 (definición **original**) |
| Q1 a Q10 | Patrones de acceso | Modelo §6.1 (definición **original**) |
| ADR-001 a ADR-004 | Decisiones estructurales | Originales no entregados; ver `decisions/records/` ([reconstruido]) |
| CA-06.1 | Criterio de aceptación de `spec.md` («una fila por producto», según §11.1) | `spec.md`, no entregado. Su texto no se reproduce (DISC-09) |
| NFR-xx, US-xx | Requisitos e historias de **este repositorio** | `04-requirements/` |
| TS-xx | Escenarios de prueba documentales de **este repositorio** | `04-requirements/testing-strategy.md` |
| DISC-xx | Discrepancias del modelo registradas por este repositorio | [model-discrepancies.md](model-discrepancies.md) |
