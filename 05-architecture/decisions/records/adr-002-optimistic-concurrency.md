# ADR-002 — Concurrencia optimista y `stock >= 0` como última barrera

> **[reconstruido]** de las citas en §2.2, §4 y
> «Cómo se lee este documento».

## Contexto

La operación `Product.Withdraw` lee stock (Q3, el punto de
contención, §6.1) y lo modifica. Dos operaciones simultáneas
podrían dejar el stock negativo si la guarda vive solo en C#.

## Decisión

- **Concurrencia optimista** con testigo `xmin` (columna de
  sistema, expuesta como propiedad sombra, T-10; §3, D-04).
- **`stock >= 0` se aplica en el motor** con
  `ck_product_stock_non_negative` — la **última barrera**
  (§2.2, §4).
- **Criterio general para toda invariante expresable en el
  motor:** *si la restricción salta, algo escribió fuera del
  adaptador* (§ «Cómo se lee», ADR-002).

## Consecuencias

- Las invariantes que hoy son «solo dominio» son deuda
  declarada: T-20 las baja al motor con cinco `CHECK` y un
  índice, sin cambiar el dominio (§4).
- Un `INSERT` por `psql` que viole `stock >= 0` **falla**;
  las que aún son «solo dominio» **pasan sin ruido** — ese
  es exactamente el riesgo que T-20 cierra (§ «Cómo se lee»).

## Estado

Vigente. `ck_product_stock_non_negative` verificada en el
motor (§10.2).
