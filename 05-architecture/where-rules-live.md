# Dónde vive cada regla

> Fuente directa: `spec/data-model.md` §4 y §2. **Este es el
> corazón de la arquitectura**: ninguna regla queda sin dueño
> declarado, y «lo que no está aplicado se declara pendiente;
> no se promete».
>
> **Revisado tras la auditoría.** Cada regla tiene ahora **un solo estado** en
> todas las tablas de este documento. Donde el modelo se contradice, el estado es
> **«en conflicto»** y remite a [model-discrepancies.md](model-discrepancies.md).

## Las marcas

| Marca | Significa | Riesgo |
|---|---|---|
| **motor** | Existe en Postgres; un `INSERT` manual la respeta o falla | — |
| **solo dominio** | La garantiza C# y nada más | Un `psql`, una migración o un servicio futuro la **saltan sin ruido** |
| **pendiente (T-xx)** | No existe todavía | No se promete |
| **en conflicto** | El modelo da estados contradictorios | Requiere decisión humana |

**Sufijo de evidencia** (solo para reglas **motor**):
- **verificada**: aparece en la instantánea de §10 del 2026-09-19.
- **según §13**: el modelo afirma que existe desde el 2026-09-20; **este repositorio no lo verificó**.

## Quién es quién: T-13 y T-20
- **T-20** baja invariantes al motor. Según §13 D-2 puso `sale_id NOT NULL`, el índice único `(sale_id, product_id)` con `INCLUDE (quantity, unit_price)` y FK-3. **Los cinco `CHECK` no se mencionan en §13 y su estado no está confirmado** (DISC-05).
- **T-13** crea los índices de acceso. Queda pendiente el índice parcial de `product`, el de trigramas y `pg_trgm`. El compuesto único figura en ambas tareas (DISC-06) y aquí se clasifica **una sola vez**, bajo T-20, según §13.

## Inventario completo (§4)

| Regla | Objeto en el motor | Estado único |
|---|---|---|
| Clave primaria de las 5 tablas | `PK_category`, `PK_product`, `PK_sale`, `PK_sale_item`, `PK_user` | **motor** (verificada) |
| `category.name` único | `IX_category_name` (índice único) | **motor** (verificada) |
| `user.username` único | `IX_user_username` (índice único) | **motor** (verificada) |
| `product.category_id` → `category.id`, `RESTRICT` (FK-1) | `FK_product_category_category_id` | **motor** (verificada) |
| `sale_item.sale_id` → `sale.id`, `CASCADE` (FK-2) | `FK_sale_item_sale_sale_id` | **motor** (verificada) |
| `product.stock >= 0` | `ck_product_stock_non_negative` | **motor** (verificada) — última barrera de ADR-002 |
| Baja lógica: `deleted_at` + filtro global | Columna `product.deleted_at` | **motor** (según §13 D-1, T-09). §6.3 y §7.1 aún la llaman pendiente (DISC-08) |
| `sale_item.sale_id NOT NULL` | `sale_item.sale_id` | **motor** (según §13 D-2, T-20). §3 del modelo aún lo muestra nulable (DISC-04) |
| Único `(sale_id, product_id)` con `INCLUDE (quantity, unit_price)` | `IX_sale_item_sale_id_product_id` | **motor** (según §13 D-2, T-20). §6.2 aún lo llama «falta (T-13)» (DISC-06) |
| `sale_item.product_id` → `product.id`, `RESTRICT` (FK-3) | `FK_sale_item_product_product_id` | **motor** (según §13 D-2, T-20). §3, §4 y §5 del modelo aún dicen que no existe (DISC-04) |
| `product.price > 0` | — | **solo dominio** (`Product.ChangePrice`); `CHECK` de T-20 sin confirmar (DISC-05) |
| `sale_item.quantity > 0` | — | **solo dominio** (`Quantity`); `CHECK` de T-20 sin confirmar (DISC-05) |
| `category.name` no vacío | — | **solo dominio** (`Category.Rename`); `CHECK` de T-20 sin confirmar (DISC-05) |
| `user.role` en `('admin','seller')` | — | **solo dominio** (`Roles.IsValid`); `CHECK` de T-20 sin confirmar (DISC-05) |
| `user.username` en minúsculas | — | **solo dominio** (`User.NormalizeUsername`); `CHECK` de T-20 sin confirmar (DISC-05) |
| `sale_item.category_name NOT NULL` (congelado, sin FK) | `sale_item.category_name` | **en conflicto** (T-11): §2.4 dice motor, §3 dice pendiente (DISC-03) |
| `sale.sold_by_user_id` → `user.id`, `RESTRICT` (FK-4) | — | **pendiente (T-12)** |
| Índice parcial `product (category_id, name)` sobre activos | — | **pendiente (T-13)**; sustituye a `IX_product_category_id` e `IX_product_name` |
| Índice de trigramas `product (name)` parcial | — | **pendiente (T-13)** |
| `sale (sold_at)` | `IX_sale_sold_at` | **motor** (verificada) |
| `sale_item (product_id)` | `IX_sale_item_product_id` | **motor** (verificada) |
| `sale_item (sale_id)` suelta | `IX_sale_item_sale_id` | **motor** en la instantánea; §6.2 exige borrarla con el compuesto. **Su retiro no está confirmado** (DISC-06) |
| Retirar más stock del disponible falla | — | **solo dominio** (`Product.Withdraw`); no expresable en un `CHECK` |
| Venta con al menos una línea | — | **solo dominio** (`Sale.EnsureConfirmable`) |
| Venta inmutable | — | **solo dominio** (por ausencia de operación) |

**Cuenta honesta.** No se da aquí un total de restricciones ni de índices vigentes. La instantánea (2026-09-19) dio 8 restricciones y 12 índices; los cambios posteriores del §13 no se verificaron (DISC-07).

**La deuda concreta.** Mientras no se confirme lo contrario, las cinco invariantes «solo dominio» dependen de que todo el mundo pase por el adaptador (§4).

## Reglas que **no** viven en ningún sitio (por diseño)

| Regla | Por qué no existe |
|---|---|
| `created_at` / `updated_at` + disparador | §8: sin requisito, contradice «sin `DEFAULT`», ningún puerto las leería |
| Moneda en ninguna tabla | D-05: monomoneda por construcción |
| FK de `sale_item.category_name` | Sin clave foránea **a propósito**: renombrar la categoría reescribiría el histórico, que ADR-004 prohíbe (§3) |
| Índice de `user.password_hash` | **Nunca se indexa** — no es rendimiento, es privacidad (§6.3, §7) |
| Índice de `sale.sold_by_user_id` | Ningún patrón lo usa; la consulta que lo justificaría cruza datos personales (§6.3, DP-02) |
| Unicidad insensible a acentos en `category.name` | Aceptada **a sabiendas**: sin CRUD de categorías nadie puede provocar la colisión (§4.1) |

## Índices: resumen por estado

| Índice | Sirve a | Estado único |
|---|---|---|
| `product (category_id, name)` parcial sobre activos | Q1 | **pendiente (T-13)**; depende de la columna de baja (T-09) |
| `product (name)` con trigramas, parcial sobre activos | Q1 | **pendiente (T-13)** — el primero que se cae si se objeta la extensión |
| `sale_item (sale_id, product_id)` único `INCLUDE (quantity, unit_price)` | Unicidad, Q6, Q9 | **motor** (según §13 D-2, T-20); solape con T-13 en DISC-06 |
| `sale (sold_at)` | Q7, Q9 | **motor** (verificada) — ascendente; el `DESC` sería cosmético |
| `sale_item (product_id)` | Q9 y verificación de FK-3 | **motor** (verificada) |
| `category (name)` único · `user (username)` único | Integridad, Q4, Q10 | **motor** (verificada) |
| `sale_item (sale_id)` suelta | — | Ver fila del inventario; su retiro no está confirmado |

**`pg_trgm`:** no estaba instalada en la instantánea (solo `plpgsql`, §10.3). La instalará **la propia migración EF** que cree el índice de trigramas (T-13, pendiente) — la misma migración, no otra; no puede ir en `db/init/` porque ADR-001 reserva todo el DDL a las migraciones. Es *trusted* en Postgres 16 (§6.2).
