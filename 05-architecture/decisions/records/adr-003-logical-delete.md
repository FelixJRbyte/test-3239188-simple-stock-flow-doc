# ADR-003 — Baja lógica, nunca borrado físico

> **[reconstruido]** de las citas en §2.2, §3, §5, §6.3, §7.1 y §13.
> El ADR original (`adr/adr-003-baja-logica.md`) **no se entrega**. No hay
> aprobación, fecha ni autor originales disponibles, y no se inventan.

## Contexto

Un producto puede dejar de venderse, pero sus líneas de venta y su presencia en
reportes históricos **no pueden desaparecer** (§7.1).

## Decisión

- **Baja lógica** mediante columna `deleted_at` (`timestamptz`, nulable, sin
  defecto) y **filtro global** que la aplica (§2.2, §3).
- Es **propiedad sombra**: no existe como propiedad del agregado (§3, D-03).
- Nula mientras el producto está activo: así sirve de predicado a los índices
  parciales (§3).
- **Nunca borrado físico** (§7.1).
- **FK-3** (`sale_item.product_id` → `product.id`, `RESTRICT`) es la **barrera de
  última instancia**: un `DELETE` manual o un cambio de código futuro debe
  **fallar ruidosamente** en vez de corromper el histórico (§5).

## Consecuencias

- Con la baja lógica, FK-3 nunca debería dispararse en uso normal; existe para
  que lo anormal sea ruidoso.
- Los productos dados de baja desaparecen de las búsquedas (Q1, Q3) pero sus
  líneas de venta permanecen intactas.
- Al dar de baja un producto se elimina también el binario de imagen, siguiendo
  el orden de §7.1.

## Historial de la contradicción sobre FK-3

1. El modelo (§5) señaló que este ADR razonaba sobre FK-3 **como si existiera**
   y que **nunca se implementó** (asignada a T-20).
2. El registro de deuda de §13 D-2 (**2026-09-20**) afirma que FK-3 **existe ya en
   el motor** (`FK_sale_item_product_product_id`, `RESTRICT`).
3. Los apartados §3, §4 y §5 del modelo **conservan** texto del punto 1.
   Esa contradicción **no está resuelta** (DISC-04).

## Estado

- **Decisión:** vigente.
- **`deleted_at` y filtro global (T-09):** **saldada** según §13 D-1 (2026-09-20).
  §6.3 y §7.1 aún la llaman pendiente (DISC-08). **No verificada** en este
  repositorio.
- **FK-3 (T-20):** **implementada según §13 D-2**, **no verificada**. Contradicha
  por §3, §4 y §5 (DISC-04).
- La versión anterior de este ADR decía «Parcial: `deleted_at` y FK-3 pendientes»;
  quedó desactualizada frente a §13.

## Referencias

- `spec/data-model.md` §2.2, §3, §5, §6.3, §7.1, §13 D-1 y D-2.
- [model-discrepancies.md](../../model-discrepancies.md) DISC-04 y DISC-08.
- [id-index.md](../../id-index.md) (T-09, T-20, D-03).
