# Entidades, reglas e invariantes

> Fuente: `spec/data-model.md` §2 (y §3, §4, §5 para dónde vive cada regla).
> **Cinco entidades, cinco tablas, sin excedente** (§2): no hay tabla de
> reporte, ni de auditoría, ni de contadores, ni para objetos de valor (D-07).
>
> **Revisado tras la auditoría:** los estados de este documento siguen el criterio de
> [where-rules-live.md](../05-architecture/where-rules-live.md). «Según §13» = el modelo lo
> afirma el 2026-09-20 y este repositorio **no lo verificó**. Contradicciones del
> modelo: [model-discrepancies.md](../05-architecture/model-discrepancies.md).

## Convención de nombres (§0)

- Entidad: `PascalCase`, inglés, **singular** → `SaleItem`.
- Atributo: `snake_case`, inglés, **singular** → `unit_price`.
- Tabla: singular (`category`, `product`, `sale`, `sale_item`, `user`);
  el **esquema** sigue llamándose `sales` → forma cualificada del agregado
  de ventas: `sales.sale` (§0).
- Las colecciones de código (`Sale.Items`, `DbSet<Product> Products`) **siguen
  en plural**: nombran conjuntos de objetos. Traducir tabla↔colección es
  responsabilidad del adaptador de persistencia (§0).

## Marca de cada regla (corazón del documento, § «Cómo se lee»)

| Marca | Significa |
|---|---|
| **motor** | Existe en Postgres; un `INSERT` manual la respeta o falla |
| **solo dominio** | La garantiza C# y nada más; un `INSERT` por `psql` la salta sin ruido |
| **pendiente (T-xx)** | No existe todavía; la pone esa tarea de `tasks.md` (no entregado) |
| **en conflicto** | El modelo da estados contradictorios; ver DISC-xx |

> **Sobre T-20.** El modelo la describe como «cinco `CHECK` y un índice» (§4), pero §13 D-2
> solo le atribuye `sale_id NOT NULL`, el índice único compuesto y FK-3. **Los cinco `CHECK` siguen
> como «solo dominio; ejecución de T-20 sin confirmar»** (DISC-05).

---

## 1. `Category` — entidad de referencia (§2.1)

| Invariante | Quién la cumple | Marca |
|---|---|---|
| Nombre obligatorio, no vacío, guardado recortado | `Category.Rename` | **solo dominio** · `CHECK` de T-20 sin confirmar |
| Nombre único | Índice único `IX_category_name` | **motor** (verificada) |

**No es raíz de agregado y no tiene ciclo de vida.** Su repositorio es de
**solo lectura**: ningún puerto crea, renombra ni borra categorías. Las cinco
filas nacen en la migración inicial (§9.1): *General*, *Herramientas*,
*Electricidad*, *Fontanería*, *Pinturas* — con UUID literales para que las
pruebas los referencien.

**Acentos y mayúsculas:** la unicidad es sensible a ambos, **aceptado por
escrito** — nadie puede provocar la colisión por la interfaz porque no hay CRUD
de categorías. **Condición de revisión: si algún día se abre el mantenimiento
de categorías, esta decisión se revisa antes de escribir ese CRUD** (§4.1).

## 2. `Product` — raíz de agregado (catálogo) (§2.2)

| Invariante | Quién la cumple | Marca |
|---|---|---|
| Nombre obligatorio, no vacío, recortado | `Product.Rename` | **solo dominio** (`NOT NULL` sí está en el motor; *no vacío* no) |
| `price > 0` | `Product.ChangePrice` | **solo dominio** · `CHECK` de T-20 sin confirmar |
| `stock >= 0` tras cualquier operación | `Product.Withdraw` / `Product.Restock` | **motor** (verificada) — `ck_product_stock_non_negative`, última barrera de ADR-002 |
| Retirar más stock del disponible falla | `Product.Withdraw` | **solo dominio** — regla de proceso, no expresable en un `CHECK` |
| Categoría obligatoria y existente | `Product.SetCategory` + `FK_product_category_category_id` | **motor** (FK-1, verificada) |
| `image_key` ausente ⇒ `NULL`, nunca `""` | `Product.AttachImage` normaliza blanco a `null` | **solo dominio** · no hay regla equivalente pendiente: el `NULL` basta |
| Nunca se borra físicamente: baja lógica | Propiedad sombra `deleted_at` + filtro global | **motor** (según §13 D-1, T-09; ADR-003) |

**`Money` admite importe cero y esto importa** (§2.2): su constructor rechaza
solo negativos, así que `new Money(0)` es válido. La única guarda de `price > 0`
es `Product.ChangePrice`: **un producto a precio 0 insertado por `psql` pasa**
mientras el `CHECK` de T-20 no se confirme.

**Regla de redondeo en `Money`, no en la columna** (§2.2): `Money` redondea a
**2 decimales con `MidpointRounding.AwayFromZero`**; la columna es `numeric(18,2)`.
Coinciden por construcción. **Si una cambia, la otra cambia en la misma migración.**

## 3. `Sale` — raíz de agregado (ventas) (§2.3)

| Invariante | Quién la cumple | Marca |
|---|---|---|
| Registra quién la realiza; obligatorio y no vacío | Constructor de `Sale` | **solo dominio** (`NOT NULL` sí está en el motor) |
| **Al menos una línea** para poder confirmarse | `Sale.EnsureConfirmable` | **solo dominio** — no expresable en un `CHECK` |
| **Un producto no se repite** en la misma venta | `Sale.AddItem` rechaza el duplicado | **solo dominio** *y* **motor** según §13 D-2: índice único `(sale_id, product_id)` con `INCLUDE (quantity, unit_price)` (T-20) |
| Descontar stock y añadir la línea son **una sola operación** | `Sale.AddItem` llama a `Product.Withdraw` antes de añadir | **solo dominio** — la regla que da sentido al agregado |
| Inmutable una vez registrada | **No existe puerto de edición ni de borrado** | **solo dominio** (por ausencia de operación) |

**La venta no conoce la moneda** (§2.3): el total se calcula sumando subtotales
y `Money` exige la misma moneda al sumar; el mapeo reconstruye siempre la moneda
por defecto. La guarda explícita en `Sale.AddItem` es deuda barata con tarea: T-05 (pendiente).

## 4. `SaleItem` — entidad interna del agregado `Sale` (§2.4)

| Invariante | Quién la cumple | Marca |
|---|---|---|
| Producto obligatorio | Constructor de `SaleItem` + `NOT NULL` | **motor** · `NOT NULL` y FK-3 `FK_sale_item_product_product_id` con `RESTRICT` **según §13 D-2** (T-20; DISC-04) |
| `quantity > 0` | Constructor de `Quantity` | **solo dominio** · `CHECK` de T-20 sin confirmar |
| Nombre y precio **congelados** en el instante de la venta | `Sale.AddItem` copia de `Product` | **solo dominio**, por construcción |
| Nombre de **categoría congelado** | Constructor de `SaleItem` + `NOT NULL` | **en conflicto** (T-11): §2.4 dice motor, §3 dice pendiente (DISC-03). `sale_item.category_name`, **sin clave foránea a propósito** (D-06, ADR-004) |
| **No existe fuera de su venta** | `FK_sale_item_sale_sale_id ON DELETE CASCADE` | **motor**: la cascada (FK-2, verificada) y el `sale_id NOT NULL` que la completa **según §13 D-2** (T-20) |

**No se construye desde fuera:** su constructor es `internal` y solo `Sale.AddItem`
lo invoca — no hay forma legítima de fabricar una línea suelta (§2.4).

## 5. `User` — raíz de agregado (identidad) (§2.5)

| Invariante | Quién la cumple | Marca |
|---|---|---|
| Nombre de usuario obligatorio y **único** | Constructor + índice único `IX_user_username` | **motor** (unicidad, verificada) |
| Nombre de usuario **en minúsculas y recortado** | `User.NormalizeUsername` | **solo dominio** · `CHECK` de T-20 sin confirmar |
| Hash de clave obligatorio y no vacío | Constructor de `User` | **solo dominio** (`NOT NULL` sí está en el motor) |
| `role` en `('admin','seller')` | `Roles.IsValid` | **solo dominio** · `CHECK` de T-20 sin confirmar. **DP-04:** desde la aplicación no se otorga `admin` |
| El dominio **nunca ve la clave en claro** | El hash lo produce un puerto (D-09) | Por diseño del hexágono |

**Por qué la normalización es invariante y no comodidad** (§2.5): una búsqueda
que se saltara `NormalizeUsername` dejaría registrar `"Ana "` como cuenta nueva
que **nunca podría iniciar sesión**: el agregado la guardaría como `ana` y
chocaría con la existente.

---

## Objetos de valor (§1, §2, D-07)

| Objeto | Regla | Vive en |
|---|---|---|
| `Money` | Rechaza negativos; **admite cero**; redondea a 2 decimales (`AwayFromZero`) | `product.price`, `sale_item.unit_price` |
| `Quantity` | Estrictamente positivo | `sale_item.quantity` |
| `DateRange` | El fin no puede ser anterior al inicio | Capa de aplicación; **sin tabla** |

Los objetos de valor **no tienen identidad ni tabla**: viven dentro de la fila
de su dueño (D-07, §2).

## Política de claves foráneas (§5) — cuatro relaciones lógicas

**Tres niveles, que no son lo mismo:**
- **Relación lógica**: existe en el dominio, con o sin clave foránea.
- **FK declarada**: está en la política de §5.
- **FK implementada / verificada**: existe en el motor. *Verificada* solo si consta en la instantánea de §10.2 (2026-09-19).

| # | FK | `ON DELETE` | Declarada | Implementada (según el modelo) | Verificada | Por qué |
|---|---|---|---|---|---|---|
| FK-1 | `product.category_id` → `category.id` | `RESTRICT` | Sí | Sí | **Sí** | Una categoría con productos no se elimina |
| FK-2 | `sale_item.sale_id` → `sale.id` | `CASCADE` | Sí | Sí | **Sí** | Composición pura; en la práctica nunca se dispara (no se borran ventas) |
| FK-3 | `sale_item.product_id` → `product.id` | `RESTRICT` | Sí | **Sí, según §13 D-2** (T-20) | **No** (instantánea anterior; DISC-04) | Barrera de última instancia: un `DELETE` manual debe **fallar ruidosamente** (ADR-003) |
| FK-4 | `sale.sold_by_user_id` → `user.id` | `RESTRICT` | Sí | **No** (T-12 pendiente) | No | La autoría de una venta es dato contable: un usuario con ventas no se elimina |

**Resumen coherente con la tabla:** de las cuatro relaciones, **cuatro están declaradas**;
**tres están implementadas según el modelo** (FK-1, FK-2, FK-3); **dos están verificadas** (FK-1,
FK-2); **una está pendiente** (FK-4). La relación N:M `sale` ↔ `product` no es una quinta clave: se
resuelve por `sale_item` (FK-2 y FK-3).

`ON UPDATE NO ACTION` en las cuatro, **y es una decisión**: las PK son UUID
generados por la aplicación y jamás cambian; un `CASCADE` en `UPDATE` sería
maquinaria muerta (§5).

**N:M: exactamente una** — `sale` ↔ `product`, resuelta por `sale_item`, que
porta datos propios (`quantity`, `unit_price`, `product_name`, `category_name`).
**No se introduce ninguna otra tabla puente.** `user` ↔ `role` **no** es N:M:
es un valor único por usuario dentro de un conjunto cerrado de dos (§5).

## Deuda concreta del modelo (§4)

Las cinco invariantes *solo dominio* (`price > 0`, `quantity > 0`, `category.name` no vacío,
`role`, `username` en minúsculas) siguen siendo deuda declarada: su `CHECK` figura
como T-20 pero **su ejecución no está confirmada** (DISC-05). **No cambia ni una línea de
dominio**; lo que cambia es que dejan de depender de que todo el mundo pase
por el adaptador (§4).
