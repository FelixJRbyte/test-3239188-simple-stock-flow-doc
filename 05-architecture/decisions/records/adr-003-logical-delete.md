# ADR-003 — Baja lógica, nunca borrado físico

> **[reconstruido]** de las citas en §2.2, §3, §5, §6.3, §7.1.

## Contexto

Un producto puede dejar de venderse, pero sus líneas de
venta y su presencia en reportes históricos **no pueden
desaparecer** (§7.1).

## Decisión

- **Baja lógica** mediante columna `deleted_at`
  (`timestamptz`, nulable, sin defecto — verificada en
  `information_schema`, §3) y **filtro global** que la aplica.
- Es **propiedad sombra**: no existe como propiedad del
  agregado (§3, D-03).
- Nula mientras el producto está activo: así sirve de
  predicado a los índices parciales (§3).
- **Nunca borrado físico** (§7.1).
- **FK-3** (`sale_item.product_id` → `product.id`,
  `RESTRICT`) es la **barrera de última instancia**: un
  `DELETE` manual o un cambio de código futuro debe **fallar
  ruidosamente** en vez de corromper el histórico (§5).

## Contradicción cerrada por el modelo

Este ADR **razona sobre FK-3 como si existiera** y **nunca
se implementó**: hasta hoy ningún documento lo decía. Queda
dicho, con marca y con tarea: **T-20** (§5).

## Estado

Parcial: la decisión está tomada; `deleted_at` y FK-3 son
**pendientes (T-09 y T-20)** respectivamente.
