# ADR-004 — Reporte agregado y congelado

> **[reconstruido]** de las citas en §1, §2.4, §3, §6.1, §6.2, D-06.

## Contexto

El negocio necesita saber qué se vendió por producto en un
período. El catálogo, en cambio, cambia: precios, nombres y
etiquetas de categoría se renombran y reprecifican.

## Decisión

- **Congelamiento:** cada línea guarda copia del nombre,
  precio y categoría del instante de la venta (§2.4). No es
  desnormalización: son **hechos propios de la venta** (§1).
- **`sale_item.category_name` va sin clave foránea, a
  propósito**: si la tuviera, renombrar la categoría
  reescribiría el histórico — justo lo que este ADR prohíbe
  (§3). Mismo ancho que `category.name`; ambas se mueven
  juntas (§3).
- **El reporte no se persiste:** se calcula **en el motor**
  por un puerto de lectura (§1, D-06); agrega por producto
  sobre un rango, ordenado por importe descendente (§6.1, Q9).
- **El reporte no se desglosa por vendedor** (DP-02).
- **Optimización deliberada:** índice único
  `(sale_id, product_id)` con `INCLUDE (quantity, unit_price)`
  para que la agregación **no toque la tabla** — la única
  optimización deliberada del diseño (§6.2).

## Consecuencias

- Con las columnas incluidas, el reporte es un recorrido
  solo-índice (salvedad: en tabla que solo recibe inserciones,
  el mantenimiento automático se dispara poco; mitigación
  operativa, no de diseño, §6.2).
- Q8 (rango sin paginar) **no tiene consumidor** si el
  reporte agrega en el motor: conviene retirarlo del puerto
  (§6.1).

## Estado

Vigente. Requiere T-11 (`category_name`), T-13 (índice
compuesto con `INCLUDE`) y T-20 (`sale_id NOT NULL` previo).
