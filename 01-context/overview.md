# Descripción general del sistema — Simple Stock Flow

## Qué es

Un sistema web para **gestionar un catálogo de productos y registrar ventas**
con reglas estrictas y un esquema mínimo. Cinco entidades, cinco tablas, sin
excedente (§2): `category`, `product`, `sale`, `sale_item`, `user` (§0, §3).

**Rasgos definitorios (todos del modelo):**

- **Monomoneda por construcción**: ninguna tabla tiene columna de moneda (D-05, §3).
- **Sin datos de cliente**: la venta registra al **operador interno**, no al comprador.
  No existe dato personal de cliente final (§1, §7).
- **Stock nunca negativo**: invariante de dominio *y* barrera en el motor
  (`ck_product_stock_non_negative`, §2.2, §4).
- **Ventas inmutables**: una vez registrada no se edita ni se borra (§2.3).
- **Precio y nombre congelados** en la línea de venta: son hechos de la venta,
  no lecturas tardías del catálogo (§1 «Nombre congelado», §2.4).
- **Sin columnas de auditoría**: `sold_at` es el único instante de negocio del
  sistema; `deleted_at` (T-09, saldada según §13 D-1) es la única transición de
  estado que se rastrea (§8).
- **Esquema mínimo**: cinco tablas y sin `DEFAULT` en ninguna columna (§3, §10.1).
  **El número exacto de columnas está sin resolver:** 21 en la instantánea del
  2026-09-19 (§10.1) y 22 en §3, §10 y §12 (DISC-01).

## Problema que resuelve

**[derivado]** del modelo: el sistema cierra cuatro fallas concretas —
**stock que puede quedar negativo**, **precios que cambian y reescriben el
histórico**, **un instante de venta poco claro** y **complejidad innecesaria**
(multi-moneda, clientes, auditoría)—, detalladas en
[problem.md](../03-product/problem.md). Se resuelven con el modelo como única verdad:
`stock >= 0` garantizado en el motor, precio congelado en `sale_item.unit_price`
(§2.4) y `sold_at` como el único instante de negocio (§8).

**[supuesto]** El perfil del negocio al que se destina (sector, herramientas que
usa hoy) **no figura en el modelo**. Las cinco categorías sembradas
(§9.1) sugieren un comercio de herramientas y suministros, pero es una inferencia.

## Quiénes lo usan

| Rol | Qué hace | Origen |
|---|---|---|
| **Vendedor (`seller`)** | Registra productos, ajusta stock y registra ventas **[derivado]** | `user.role` ∈ {admin, seller} (§2.5) |
| **Administrador (`admin`)** | Da de alta vendedores; el rol `admin` lo provisiona el despliegue y nadie lo otorga en ejecución (DP-04) | §2.5, §11 (H-3), §13 D-3 |

El reparto exacto de operaciones por rol, salvo el alta de usuarios, **no está en el
modelo** ([security-and-authorization.md](../05-architecture/security-and-authorization.md)).
**No hay tercer rol ni entidad cliente** (§2.5, §1). Las categorías son datos
semilla de **solo lectura**: ningún puerto las crea, renombra ni borra (§2.1, D-10).

## Stack y forma del sistema

> **[derivado]** Referencia técnica extraída del modelo para situar al lector;
> **no es un requisito de producto**.

| Capa | Tecnología | Origen |
|---|---|---|
| Dominio y aplicación | **C#** (las invariantes «solo dominio» las garantiza C#, §2) | §2 |
| Persistencia | **PostgreSQL 16.14**, base `simple_stock_flow`, esquema `sales`, servidor en UTC | Encabezado, §3 |
| Migraciones | **EF Core** — el esquema lo poseen las migraciones y nada más (ADR-001, §3.2) | §3.2 |
| Infraestructura | Docker Compose (repo `simple-stock-flow-infra`, §10) | §10 |
| Almacenamiento de imágenes | Externo; `image_key` es clave opaca, no ruta ni bytes (D-08) | §1, §2.2 |

> **[supuesto]** La versión concreta de .NET no aparece en el modelo; el modelo
> solo dice C#. El informe de ventas se calcula **en el motor** por un puerto de
> lectura (D-06, §1) — no hay capa de reportes propia.

## Estado actual

> Estados según [id-index.md](../05-architecture/id-index.md). «Según §13» significa
> que el modelo lo afirma y este repositorio **no lo verificó**.

- Modelo de datos **verificado contra el motor por última vez el 2026-09-19** (§10),
  antes de los cambios que registra §13 (2026-09-20) (DISC-07).
- Esquema: cuatro migraciones aplicadas en la instantánea (§3.2). Los cambios
  posteriores implican más migraciones que el modelo no lista. Las cifras de
  columnas, restricciones e índices de la instantánea (21, 8, 12) **no se
  presentan como vigentes**.
- Tabla `user` con **exactamente una fila** en la instantánea (administrador inicial, §9.2).
- **Saldado según §13:** T-09 (`deleted_at` y filtro global), partes de T-20
  (`sale_id NOT NULL`, índice único compuesto, FK-3) y A-1 (alta anónima).
- **Pendiente:** T-12 (`sold_by_user_id` y FK-4), T-13 (índice parcial de `product`,
  trigramas y `pg_trgm`), T-05 (guarda de moneda).
- **Sin confirmar:** los cinco `CHECK` de T-20 (DISC-05) y T-11 (DISC-03).
- **Abierto:** H-2 (binario de imagen huérfano) y A-7 (desempate del nombre de
  producto en el reporte).
