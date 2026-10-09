# Dónde vive cada regla

> Fuente directa: `spec/data-model.md` §4 y §2. **Este es el
> corazón de la arquitectura**: ninguna regla queda sin dueño
> declarado, y «lo que no está aplicado se declara pendiente;
> no se promete».

## Las tres marcas

| Marca | Significa | Riesgo |
|---|---|---|
| **motor** | Existe en Postgres hoy; un `INSERT` manual la respeta o falla | — |
| **solo dominio** | La garantiza C# y nada más | Un `psql`, una migración o un servicio futuro la **saltan sin ruido** |
| **pendiente (T-xx)** | No existe todavía | No se promete |

## Inventario completo (§4)

| Regla | Objeto en el motor | Dónde vive hoy |
|---|---|---|
| Clave primaria de las 5 tablas | `PK_category`, `PK_product`, `PK_sale`, `PK_sale_item`, `PK_user` | **motor** |
| `category.name` único | `IX_category_name` (índice único) | **motor** |
| `user.username` único | `IX_user_username` (índice único) | **motor** |
| `product.category_id` → `category.id`, `ON DELETE RESTRICT` | `FK_product_category_category_id` (FK-1) | **motor** |
| `sale_item.sale_id` → `sale.id`, `ON DELETE CASCADE` | `FK_sale_item_sale_sale_id` (FK-2) | **motor** |
| `product.stock >= 0` | `ck_product_stock_non_negative` | **motor** — última barrera de ADR-002 |
| `product.price > 0` | — | **solo dominio** · `Product.ChangePrice` · baja al motor en **T-20** |
| `sale_item.quantity > 0` | — | **solo dominio** · constructor de `Quantity` · **T-20** |
| `category.name` no vacío | — | **solo dominio** · `Category.Rename` · **T-20** |
| `user.role` en `('admin','seller')` | — | **solo dominio** · `Roles.IsValid` · **T-20** |
| `user.username` en minúsculas | — | **solo dominio** · `User.NormalizeUsername` · **T-20** |
| `sale_item.sale_id NOT NULL` | `sale_item.sale_id` | **motor** (T-20) — prerrequisito del compuesto único |
| Único `(sale_id, product_id)` con `INCLUDE (quantity, unit_price)` | `IX_sale_item_sale_id_product_id` | **motor** (T-20/T-13) · hoy no puede cumplir su función mientras `sale_id` admita nulos |
| `sale_item.product_id` → `product.id`, `RESTRICT` | `FK_sale_item_product_product_id` (FK-3) | **motor** (T-20) — barrera de última instancia de ADR-003; **nunca se implementó y hoy `sale_item` no tiene ninguna FK hacia el catálogo** |
| `sale.sold_by_user_id` → `user.id`, `RESTRICT` | — | **pendiente (T-12)** — la autoría de una venta no puede quedar huérfana |
| Índices de acceso | ver §6.2 | Tres existen, tres faltan (T-13) |

**La deuda concreta:** bajar al motor las cinco invariantes
*solo dominio* es **T-20** — cinco `CHECK` y un índice. No
cambia ni una línea de dominio; cambia que dejan de depender
de que todo el mundo pase por el adaptador (§4).

## Reglas que **no** viven en ningún sitio (por diseño)

| Regla | Por qué no existe |
|---|---|
| `created_at` / `updated_at` + disparador | §8: sin requisito, contradice «sin `DEFAULT`», ningún puerto las leería |
| Moneda en ninguna tabla | D-05: monomoneda por construcción |
| FK de `sale_item.category_name` | Sin clave foránea **a propósito**: renombrar la categoría reescribiría el histórico, que ADR-004 prohíbe (§3) |
| Índice de `user.password_hash` | **Nunca se indexa** — no es rendimiento, es privacidad (§6.3, §7) |
| Índice de `sale.sold_by_user_id` | Ningún patrón lo usa; y la consulta que lo justificaría cruza datos personales que el negocio decidió no exponer (§6.3, DP-02) |
| Unicidad insensible a acentos en `category.name` | Aceptada **a sabiendas**: sin CRUD de categorías nadie puede provocar la colisión (§4.1) |

## Índices: los que hay y los que faltan (§6.2)

| Índice | Sirve a | Estado |
|---|---|---|
| `product (category_id, name)` parcial sobre activos | Q1 | **falta (T-13)** — sustituye a `IX_product_category_id` e `IX_product_name`, que hoy existen sueltos: **se sustituyen, no se suman** |
| `product (name)` con trigramas, parcial sobre activos | Q1 | **falta (T-13)** — ningún árbol B sirve un comodín a la izquierda; el primero que se cae si se objeta la extensión |
| `sale_item (sale_id, product_id)` único, `INCLUDE (quantity, unit_price)` | Unicidad, Q6, Q9 | **falta (T-13)** — exige antes `sale_id NOT NULL` |
| `sale (sold_at)` | Q7, Q9 | **existe** — `IX_sale_sold_at` ascendente, y está bien así (el `DESC` era cosmético) |
| `sale_item (product_id)` | Q9 y verificación de FK-3 | **existe** — `IX_sale_item_product_id` |
| `category (name)` único · `user (username)` único | Integridad, Q4, Q10 | **existen** — `IX_category_name`, `IX_user_username` |
| `sale_item (sale_id)` suelta | — | **existe y sobra** — se borra en la misma migración del compuesto único (parte de T-13) |

**`pg_trgm`:** no está instalada (solo `plpgsql`, §10.3). La
instala **la propia migración EF** que crea el índice de trigramas
— la misma migración, no otra; y no puede ir en `db/init/` porque
ADR-001 reserva todo el DDL a las migraciones. Es *trusted* en
Postgres 16: la puede instalar el propietario de la base (§6.2).
