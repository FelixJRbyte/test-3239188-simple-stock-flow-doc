# Cierre — comprobar que la arquitectura cuadra

> **Paso 6 del reto:** volver a la arquitectura y comprobar
> que cuadra con todo lo anterior y con el modelo.
> Cada chequeo responde **sí/no + cita**.

## 1. ¿Cada tabla tiene dueño en el dominio?

| Tabla | Dueño | ¿Cuadra? |
|---|---|---|
| `category` | Entidad de referencia, repositorio de solo lectura (§2.1) | ✅ |
| `product` | Raíz de agregado Catálogo (§2.2) | ✅ |
| `sale` | Raíz de agregado Ventas (§2.3) | ✅ |
| `sale_item` | Entidad interna de `Sale`, constructor `internal` (§2.4) | ✅ |
| `user` | Raíz de agregado Identidad (§2.5) | ✅ |

**Sin excedente:** no hay tabla de reporte, auditoría,
contadores ni de objetos de valor (§2, D-07). ✅

## 2. ¿Cada regla tiene dueño declarado?

- 8 restricciones en el motor (§10.2) ↔ inventario de
  `where-rules-live.md` (§4 del modelo). ✅
- 5 invariantes «solo dominio» con tarea **T-20** (§4). ✅
- Ninguna regla sin marca: el modelo exige las tres marcas
  y este entregable las respeta («Cómo se lee»). ✅

## 3. ¿Las claves foráneas cuadran con la política?

| FK | Política (§5) | Estado |
|---|---|---|
| FK-1 `product.category_id` | `RESTRICT`, `NO ACTION` | ✅ motor |
| FK-2 `sale_item.sale_id` | `CASCADE`, `NO ACTION` | ✅ motor |
| FK-3 `sale_item.product_id` | `RESTRICT` — barrera de última instancia | ⏳ T-20 (declarada, no implementada) |
| FK-4 `sale.sold_by_user_id` | `RESTRICT` — autoría no huérfana | ⏳ T-12 (pendiente) |

Las cuatro con `ON UPDATE NO ACTION` por diseño: las PK son
UUID generados por la aplicación y jamás cambian (§5). ✅

## 4. ¿Los índices cuadran con los patrones de acceso?

- Q1–Q10 (§6.1) ↔ inventario de índices (§6.2): cada índice
  existe por una consulta concreta; cada descarte está
  justificado (§6.3). ✅
- Tres faltan (T-13), incluido el `INCLUDE` que hace el
  reporte Q9 un recorrido solo-índice. ⏳ con tarea. ✅

## 5. ¿Las historias de usuario cuadran con las operaciones?

- US-01…US-11 (`04-requirements/user-stories.md`) trazan a
  operaciones de §2 y patrones de §6.1. ✅
- Las historias **rechazadas** (editar venta, crear categoría,
  registrar cliente, otra moneda, ventas por vendedor) trazan a
  §2.3, D-10, §1, D-05 y DP-02 respectivamente. ✅

## 6. ¿La arquitectura cuadra con las decisiones transversales?

| Decisión | Dónde se refleja | ¿Cuadra? |
|---|---|---|
| Monomoneda (D-05) | Ninguna columna de moneda en el modelo físico (§3); `Money` sin moneda (§2.3) | ✅ |
| Sin auditoría (§8) | No hay `created_at`/`updated_at` en el modelo físico; `sold_at` único instante | ✅ |
| Privacidad (§7) | `password_hash` nunca indexado; `username`/`sold_by` restringidos; NFR-16…NFR-21 | ✅ |
| Semilla (§9) | Categorías en la migración inicial con UUID literales; administrador inicial creado por el arranque, no por la base | ✅ |
| Migraciones (ADR-001) | Todo DDL en migraciones; `pg_trgm` en la misma migración del índice (§6.2) | ✅ |

## 7. ¿Qué queda abierto, con dueño?

| Pendiente | Tarea | Sección |
|---|---|---|
| 5 `CHECK` + 1 índice al motor | **T-20** | §4 |
| `deleted_at` + filtro global | **T-09** | §2.2, §3 |
| `category_name` congelado | **T-11** | §3 |
| `sold_by` → `sold_by_username` + `sold_by_user_id` (FK-4) | **T-12** | §3, §5 |
| 3 índices + `pg_trgm` | **T-13** | §6.2 |
| Alta de usuarios anónima (defecto A-1) | **política de autorización en la API** | §9.2 |
| Guarda explícita de moneda en `Sale.AddItem` | **T-05** (deuda barata) | §2.3 |

## 8. Conclusión

**El sistema descrito por la documentación (01–05) y el
descrito por el modelo de datos son el mismo sistema:**
cinco agregados/entidades, tres marcas por regla, cuatro
FK previstas (dos hoy), diez patrones de acceso, y una
lista cerrada de deuda con tarea. Lo único que la
arquitectura añade sin respaldo directo del modelo es la
**forma hexagonal** (puertos y adaptadores) — marcada como
**[derivado]** en cada archivo donde aparece, porque el
modelo la implica (§0, §1, §2.5, §6.1) pero no la nombra.

> **Verificación final contra el motor (§10):** 21 columnas,
> 8 restricciones, 12 índices. Si el motor difiere de este
> documento, **gana el motor** y el documento está roto.
