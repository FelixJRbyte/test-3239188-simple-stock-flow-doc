# ADR-001 — El esquema lo poseen las migraciones EF

> **[reconstruido]** El modelo cita este ADR (§3.2, §6.2)
> pero el archivo original (`adr/`) **no se entrega**.
> El contenido se reconstruye de las citas.

## Contexto

El esquema de la base puede definirse por varios caminos:
migraciones de EF, scripts SQL en `db/init/`, o DDL manual.

## Decisión

**El esquema lo poseen las migraciones de EF y nada más.**
Todo DDL —incluido `CREATE EXTENSION pg_trgm`— va en la
migración que lo necesita, en la misma migración (§6.2).

## Consecuencias

- `db/init/` y la infraestructura **levantan el motor, no
  definen el esquema** (§6.2).
- El historial de migraciones vive en
  `public."__EFMigrationsHistory"` — fuera del esquema
  `sales`, por lo que la consulta de columnas devuelve 21
  y no más (§3.2).
- Cuatro migraciones aplicadas: `InitialSchema`,
  `StockNonNegative`, `AccentSeedCategoryNames`,
  `RenameTablesToSingular` (§3.2).
- Renombrar tablas fue **una migración**, no un retoque de
  documento: mismo camino que todo el DDL (§3.2).

## Estado

Vigente. Sin excepciones declaradas.
