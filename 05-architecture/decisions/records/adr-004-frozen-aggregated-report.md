# ADR-004 — Reporte agregado y congelado

> **[reconstruido]** de las citas en §1, §2.4, §3, §6.1, §6.2, §11.1 y D-06.
> El ADR original (`adr/adr-004-reporte-agregado-y-congelado.md`) **no se
> entrega**. No hay aprobación, fecha ni autor originales disponibles, y no se
> inventan.

## Contexto

El negocio necesita saber qué se vendió por producto en un período. El catálogo,
en cambio, cambia: precios, nombres y etiquetas de categoría se renombran y
reprecifican.

## Decisión

- **Congelamiento:** cada línea guarda copia del nombre, precio y categoría del
  instante de la venta (§2.4). No es desnormalización: son **hechos propios de
  la venta** (§1).
- **`sale_item.category_name` va sin clave foránea, a propósito:** si la
  tuviera, renombrar la categoría reescribiría el histórico, justo lo que este
  ADR prohíbe (§3). Mismo ancho que `category.name`; ambas se mueven juntas.
- **El reporte no se persiste:** se calcula **en el motor** por un puerto de
  lectura (§1, D-06); agrega por producto sobre un rango, ordenado por importe
  descendente (§6.1, Q9).
- **Decisión H-1 (cerrada el 2026-09-20 por el propietario, §11.1):** ante una
  recategorización dentro del rango, el reporte **no elige un ganador**:
  **agrupa por el valor congelado**, es decir por
  `product_id, product_name, category_name`. Si una categoría se llamaba
  «Herramientas» en septiembre y «Ferretería» en octubre, el reporte muestra
  **dos filas**. Criterio rector del propietario: *un reporte cerrado no debe
  cambiar nunca*.
- **El reporte no se desglosa por vendedor** (DP-02).
- **Optimización deliberada:** índice único `(sale_id, product_id)` con
  `INCLUDE (quantity, unit_price)` para que la agregación no toque la tabla
  (§6.2).

## Consecuencias

- **Puede haber más de una fila por producto** cuando hubo recategorización
  dentro del rango.
- **Tensión con CA-06.1 de `spec.md`:** §11.1 afirma que CA-06.1 dice «una fila
  por producto». `spec.md` **no se entrega**, así que **no se reproduce su
  texto**. El propio modelo propone reescribirlo como «una fila por producto y
  etiqueta congelada», pero lo deja como **decisión pendiente del propietario**
  (DISC-09).
- Con las columnas incluidas, el reporte puede ser un recorrido solo-índice
  (salvedad de §6.2: en una tabla que solo recibe inserciones el mantenimiento
  automático se dispara poco; mitigación operativa, no de diseño). **No se
  verificó** en este repositorio.
- **Defecto abierto A-7 / DP-01:** el nombre de producto usa hoy un desempate
  que H-1 rechaza; por instrucción del propietario no se corrige en esa tanda
  (§11.1; DISC-10). Su detalle vive en `HANDOFF-TECNICO.md`, no entregado.
- Q8 (rango sin paginar) no tiene consumidor si el reporte agrega en el motor:
  conviene retirarlo del puerto (§6.1).

## Estado

- **Decisión:** vigente. H-1 **cerrado** (§11.1).
- **T-11** (`sale_item.category_name`): **estado en conflicto** (§2.4 dice motor;
  §3 dice pendiente). El reporte depende de esta columna (DISC-03).
- **Índice compuesto con `INCLUDE`:** **implementado según §13 D-2 (T-20)**, no
  verificado; §6.2 aún lo asigna a T-13 (DISC-06).
- **CA-06.1:** decisión pendiente del propietario.
- La versión anterior decía «Requiere T-11, T-13 y T-20» sin estado; se
  reemplaza por el estado anterior.

## Referencias

- `spec/data-model.md` §1, §2.4, §3, §6.1, §6.2, §11.1, §13.
- [model-discrepancies.md](../../model-discrepancies.md) DISC-03, DISC-06, DISC-09 y DISC-10.
- [id-index.md](../../id-index.md) (H-1, DP-01, A-7, T-11, T-13).
