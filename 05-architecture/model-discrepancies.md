# Registro de discrepancias del modelo de datos

> **Qué es esto:** contradicciones **dentro de** `spec/data-model.md` y entre el
> modelo y los documentos derivados (01–05). **No se corrigió el modelo**
> (§12 lo declara firmado). Este registro dice dónde está cada contradicción,
> qué documentos derivados deben leerse con cautela y qué decisión falta.
>
> **Criterio de lectura adoptado en los documentos derivados** *(decisión de este
> repositorio, no del modelo; revisable)*: cuando §13 o §11 (ambos del 2026-09-20)
> contradicen una marca de §2 a §9, los documentos derivados citan §13/§11 como
> *«implementada según el modelo»* y **no** como *«verificada»*. Solo la
> instantánea de §10 (2026-09-19) cuenta como verificada contra el motor, y es
> anterior a esos cambios. Este repositorio no tiene acceso al motor.

Los números de línea se refieren a `spec/data-model.md` tal como se leyó en esta revisión.

## Resumen

| ID | Tema | Estado | Documentos derivados a leer con cautela |
|---|---|---|---|
| DISC-01 | 22 columnas declaradas frente a 21 filas devueltas | **Pendiente de confirmación** (hay una hipótesis aritmética) | 01 overview, 01 scope, 03 vision, NFR-26, closure, ADR-001 |
| DISC-02 | `category_name` y `xmin` en la tabla `product` de §3 | **Pendiente de confirmación** | Ninguno lo reproduce; cuidado al transcribir §3 |
| DISC-03 | Estado de `sale_item.category_name` respecto a T-11 | **Pendiente de confirmación** | entities-and-rules, where-rules-live, ADR-004, closure |
| DISC-04 | ¿Existe la FK `sale_item.product_id → product.id` (FK-3)? | **Resuelto como criterio de lectura** (§13 D-2), sin verificación | entities-and-rules, where-rules-live, closure, ADR-003, domain-map, NFR-09 |
| DISC-05 | Alcance real de T-20: cinco `CHECK` frente a `sale_id`/índice/FK-3 | **Pendiente de confirmación** | entities-and-rules, where-rules-live, vision, closure |
| DISC-06 | Solape T-13 / T-20 en el índice único compuesto y en `IX_sale_item_sale_id` | **Pendiente de confirmación** | where-rules-live, NFR-12 a NFR-14, ADR-004, closure |
| DISC-07 | Instantánea §10 (2026-09-19) anterior a los cambios de §13 | Hecho constatado | closure, NFR-26, ADR-001, overview |
| DISC-08 | Referencias residuales a T-09 y A-1 como pendientes (§6.3, §7.1, §9.2) | **Resuelto como criterio de lectura** (§13 D-1 y D-3) | overview, scope, vision, US-08, NFR-20 |
| DISC-09 | H-1 frente a CA-06.1 | **Decisión pendiente del propietario** (lo declara §11.1) | US-07, ADR-004, ports-and-adapters |
| DISC-10 | DP-01 y defecto A-7 del nombre de producto en el reporte | **Abierto por instrucción del propietario** (§11.1) | US-07, ADR-004 |

---

## DISC-01 — 22 columnas frente a 21 filas
- **Dice 22:** §3 (título, línea 208), §10 (línea 628, «siguen siendo 22 columnas»), §12 (línea 829).
- **Dice 21:** §3 (líneas 238 y 316, «no cuenta entre las 21», «devuelve 21»), §10.1 (título y resultado, 21 filas, 2026-09-19), §8 (el ancla de la línea 539 es `#3-modelo-físico--las-21-columnas`, que no coincide con el título «22»).
- **Hipótesis aritmética `[derivado]`, no confirmada:** 21 filas de la instantánea + `product.deleted_at` (T-09, 2026-09-20) = 22. En ese caso la cifra 22 excluiría las columnas aún pendientes (`sale.sold_by_user_id`, y `sale_item.category_name` según DISC-03). Pero el modelo no lo dice y §3 lista 25 filas de columnas, incluida `product.category_name` (DISC-02).
- **Decisión pendiente:** confirmar la cifra vigente y corregir §3, §8, §10 y §12.
- **Efecto en este repositorio:** los documentos derivados ya no afirman una cifra sola: citan «21 (instantánea 2026-09-19) / 22 (§3, §10, §12)».

## DISC-02 — `category_name` y `xmin` en la tabla `product` de §3
- **`category_name`** aparece como fila de `product` (línea 236) con el texto «etiqueta congelada en el instante de la venta», que describe a `sale_item` (§2.4, §6, §11.1). Choca con §1 y DP-03 («Producto: nombre, precio, stock, categoría e imagen, y nada más») y no aparece en `product` en §10.1.
  - Parece una fila mal ubicada, pero el modelo no lo confirma.
- **`xmin`** (línea 238) sí pertenece conceptualmente a `product` (testigo de concurrencia de D-04, T-10), pero el propio modelo aclara que **no es una columna declarada** y que no cuenta en el total. No es una contradicción de fondo; es una fila de otra naturaleza dentro de una tabla de columnas.
- **Decisión pendiente:** confirmar si `product.category_name` es un error de edición. Hasta entonces, los documentos derivados **no** atribuyen `category_name` a `product`.

## DISC-03 — Estado de `sale_item.category_name` (T-11)
- «**motor** (T-11)» en §2.4 (línea 186) y en la fila de `product` de §3 (línea 236).
- «**pendiente (T-11)**» en §3, tabla `sale_item` (línea 262) y en §10.1 (líneas 681-682).
- §11.1 (línea 810) trata a T-11 como trabajo futuro («T-11 hereda el `GROUP BY`»). §13 no lo lista entre las deudas saldadas.
- **Decisión pendiente:** confirmar si T-11 está hecha. Los derivados lo registran como **«estado en conflicto»**.

## DISC-04 — FK de `sale_item` hacia `product` (FK-3)
- **No existe:** §3 (línea 257, «Sin clave foránea hoy»), §4 (línea 342, «nunca se implementó»), §5 (línea 365, «hoy existen dos»), §10.2 (8 restricciones, sin FK-3, 2026-09-19).
- **Existe:** §2.4 (línea 183), §4/§5 (columna «Estado»: «motor (T-20)»), §13 D-2 (2026-09-20: «existe `FK_sale_item_product_product_id` con `RESTRICT`»).
- **Criterio de lectura adoptado:** §13 D-2 es la entrada más reciente y dice haber sido medida contra el motor. Los derivados distinguen: **declarada** (política §5), **implementada según el modelo** (§13 D-2) y **verificada** (no: la instantánea es anterior).
- **Decisión pendiente:** repetir las consultas de §10 contra el motor y corregir §3, §4 y §5.

## DISC-05 — Alcance real de T-20
- §4 (línea 346) describe T-20 como «cinco `CHECK` y un índice».
- §13 D-2 le atribuye `sale_id NOT NULL`, el índice único compuesto y FK-3, y no menciona los cinco `CHECK`.
- §2.2, §2.4, §2.5 y §4 siguen marcando esas invariantes como «**solo dominio** · T-20».
- **No hay evidencia** de que los cinco `CHECK` (`price > 0`, `quantity > 0`, `category.name` no vacío, `role`, `username` en minúsculas) existan en el motor.
- **Decisión pendiente:** confirmar el estado de esos cinco `CHECK`. Los derivados los mantienen como *«solo dominio; ejecución de T-20 sin confirmar»*.

## DISC-06 — Solape T-13 / T-20
- El índice único `(sale_id, product_id)` con `INCLUDE (quantity, unit_price)` figura como «falta (T-13)» en §6.2 (línea 439) y como «motor (T-20)» en §2.3 (línea 171), §4 (línea 341) y §13 D-2.
- §6.2 dice que `IX_sale_item_sale_id` «se borra en la misma migración» del compuesto. No hay evidencia de que se haya borrado.
- Siguen faltando, sin conflicto, el índice parcial `product (category_id, name)` y el de trigramas, junto con `pg_trgm` (T-13).
- **Decisión pendiente:** confirmar cuál de las dos tareas creó el compuesto y si se retiró el índice suelto.

## DISC-07 — Instantánea de §10 anterior a §13
- §10.1, §10.2 y §10.3 son del 2026-09-19 (21 columnas, 8 restricciones, 12 índices). §13 y §11 son del 2026-09-20.
- §3.2 lista cuatro migraciones; los cambios de T-09, T-11 y T-20 que cita §13 implican migraciones posteriores que el modelo no lista.
- **Consecuencia:** las cifras 8 y 12 son históricas. Este repositorio ya no las presenta como vigentes.

## DISC-08 — Referencias residuales a estados ya cerrados
- T-09 como «pendiente»: §6.3 (línea 465) y §7.1 (línea 511), mientras §2.2, §3 y §13 D-1 lo dan por hecho.
- A-1 («hoy está rota»): §9.2 (líneas 604-606), mientras §13 D-3 y §11 H-3 dicen que A-1 está cerrado.
- **Criterio de lectura:** prevalecen §13 D-1 y D-3.

## DISC-09 — H-1 frente a CA-06.1
- §11.1 (líneas 806-808) declara que `spec.md` CA-06.1 dice «una fila por producto» y que la decisión H-1 puede producir más de una por producto.
- **`spec.md` no se entrega:** el texto exacto de CA-06.1 no está disponible y no se inventa.
- **Decisión pendiente del propietario:** reescribir CA-06.1 («una fila por producto y etiqueta congelada», propuesta del propio §11.1).

## DISC-10 — DP-01 y defecto A-7
- §11.1 (líneas 814-817) indica que el nombre de producto usa hoy el desempate que H-1 rechaza (defecto A-7, `HANDOFF-TECNICO.md` §6.1, **que no se entrega**) y que no se corrige «en esta tanda».
- Los documentos derivados lo registran como defecto abierto. No se conoce su detalle.

---

## Límites de este registro
- Se construyó solo a partir de lo que `spec/data-model.md` dice de sí mismo y de sus documentos referenciados. Éstos no existen en el repositorio: `constitution.md`, `spec.md`, `plan.md`, `tasks.md`, `architecture.md`, `adr/`, `api-contract.md`, `traspaso/HANDOFF-TECNICO.md` y `verify.sh`.
- No se ejecutó ninguna consulta contra un motor de base de datos.
