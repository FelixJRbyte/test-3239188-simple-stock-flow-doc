# Cierre — comprobar que la arquitectura cuadra

> **Paso 6 del reto:** volver a la arquitectura y comprobar que cuadra con todo lo
> anterior y con el modelo. **Versión revisada tras la auditoría técnica.**
> La versión anterior marcaba ✅ elementos pendientes y afirmaba una equivalencia
> exacta con el modelo que la evidencia no respalda.

## Cómo se lee este cierre

Cada regla o restricción se clasifica en **tres niveles**, que no son lo mismo:

| Nivel | Significa | Fuente aceptada |
|---|---|---|
| **Declarada** | Está en la política o en el diseño del modelo | §2 a §9 |
| **Implementada (según el modelo)** | El modelo afirma que ya existe en el motor | §13 y §11, 2026-09-20 |
| **Verificada** | Se comprobó contra el motor | **Solo** la instantánea de §10, 2026-09-19. Este repositorio no tiene acceso al motor y no la repitió |

Leyenda: ✅ coincide con la evidencia · ⏳ pendiente (no se da por completo) · ⚠️ contradicción del modelo, ver [model-discrepancies.md](model-discrepancies.md).

**Cifras históricas.** La instantánea de §10 del 2026-09-19 dio 21 columnas, 8 restricciones y 12 índices. **No se presentan como estado vigente**: son anteriores a los cambios de §13 (DISC-07). Este cierre no calcula una cifra vigente porque no hay evidencia para hacerlo.

## 1. ¿Cada tabla tiene dueño en el dominio?

| Tabla | Dueño | ¿Cuadra? |
|---|---|---|
| `category` | Entidad de referencia, repositorio de solo lectura (§2.1) | ✅ |
| `product` | Raíz de agregado Catálogo (§2.2) | ✅ |
| `sale` | Raíz de agregado Ventas (§2.3) | ✅ |
| `sale_item` | Entidad interna de `Sale`, constructor `internal` (§2.4) | ✅ |
| `user` | Raíz de agregado Identidad (§2.5) | ✅ |

**Sin excedente:** no hay tabla de reporte, auditoría, contadores ni de objetos de valor (§2, D-07). ✅

## 2. ¿Cada regla tiene dueño declarado?
El inventario completo con el estado de cada regla está en [where-rules-live.md](where-rules-live.md). Resumen:

| Grupo | Estado |
|---|---|
| Claves primarias, `category.name` único, `user.username` único, FK-1, FK-2, `stock >= 0` | **Verificadas** en la instantánea del 2026-09-19 ✅ |
| `deleted_at` + filtro global (T-09) | **Implementada según el modelo** (§13 D-1); no verificada ✅/⚠️ (DISC-08) |
| `sale_id NOT NULL`, índice único `(sale_id, product_id)`, FK-3 (T-20) | **Implementadas según el modelo** (§13 D-2); no verificadas ⚠️ (DISC-04, DISC-06) |
| Cinco `CHECK` («solo dominio», T-20) | **Declaradas; ejecución sin confirmar** ⏳ (DISC-05) |
| `sale_item.category_name` (T-11) | **Estado en conflicto** ⚠️ (DISC-03) |
| `sale.sold_by_user_id` y FK-4 (T-12) | **Pendiente** ⏳ (confirmado por §13) |
| Índice parcial de `product`, trigramas y `pg_trgm` (T-13) | **Pendiente** ⏳ |

## 3. ¿Las claves foráneas cuadran con la política?

| FK | Política (§5) | Declarada | Implementada (según el modelo) | Verificada |
|---|---|---|---|---|
| FK-1 `product.category_id` | `RESTRICT` | Sí | Sí | **Sí** (2026-09-19) ✅ |
| FK-2 `sale_item.sale_id` | `CASCADE` | Sí | Sí | **Sí** (2026-09-19) ✅ |
| FK-3 `sale_item.product_id` | `RESTRICT`, barrera de última instancia | Sí | **Sí**, según §13 D-2 (2026-09-20) | **No**: la instantánea es anterior; §3, §4 y §5 del modelo aún dicen que no existe ⚠️ (DISC-04) |
| FK-4 `sale.sold_by_user_id` | `RESTRICT` | Sí | **No** (T-12 pendiente) | No ⏳ |

Las cuatro llevan `ON UPDATE NO ACTION` por diseño (§5): las PK son UUID generados por la aplicación y no cambian. Es una decisión **declarada**, verificada solo para FK-1 y FK-2.

## 4. ¿Los índices cuadran con los patrones de acceso?
- Q1 a Q10 (§6.1) tienen un índice previsto en §6.2, con los descartes justificados en §6.3. Esa correspondencia es de **diseño** ✅.
- **Verificados (2026-09-19):** `IX_sale_sold_at`, `IX_sale_item_product_id`, `IX_category_name`, `IX_user_username`, `IX_product_category_id`, `IX_product_name` y `IX_sale_item_sale_id`.
- **Implementado según el modelo, sin verificar:** el índice único compuesto `(sale_id, product_id)` con `INCLUDE (quantity, unit_price)` (§2.3, §4, §13 D-2). §6.2 aún lo llama «falta (T-13)» ⚠️ (DISC-06).
- **Pendientes:** índice parcial `product (category_id, name)` y trigramas `product (name)`, con `pg_trgm` (T-13) ⏳.
- **Sin comprobar:** si `IX_sale_item_sale_id` se eliminó (§6.2 lo exige en la migración del compuesto).
- **No se afirma** que los índices sirvan las consultas: no se ejecutó ningún plan de ejecución ni prueba de rendimiento.

## 5. ¿Las historias de usuario cuadran con las operaciones?
- US-01 a US-11 trazan a operaciones de §2 y patrones de §6.1. ✅
- **US-07** recoge ahora H-1 (agrupación por `category_name` congelado). La tensión con CA-06.1 queda como decisión pendiente del propietario ⚠️ (DISC-09).
- **US-08** quedó alineada con DP-04: el administrador da de alta vendedores y no otorga `admin`.
- Las historias rechazadas siguen trazando a §2.3, D-10, §1, D-05 y DP-02. ✅

## 6. ¿La arquitectura cuadra con las decisiones transversales?

| Decisión | Dónde se refleja | Estado |
|---|---|---|
| Monomoneda (D-05) | Ninguna columna de moneda (§3). La guarda explícita en `Sale.AddItem` es T-05, **pendiente** | ✅ / ⏳ |
| Sin auditoría (§8) | No hay `created_at` ni `updated_at`; `sold_at` es el único instante | ✅ |
| Privacidad (§7) | `password_hash` nunca indexado; NFR-16 a NFR-21; ver [security-and-authorization.md](security-and-authorization.md) | ✅ (diseño) |
| Semilla (§9) | Categorías en la migración inicial; administrador inicial creado por el arranque | ✅ (diseño) |
| Migraciones (ADR-001) | Todo DDL en migraciones; `pg_trgm` en la misma migración del índice (§6.2) | ✅ (diseño) |
| Rol `admin` (DP-04) | Provisionado por el despliegue; nadie lo otorga en ejecución | ✅ (decisión cerrada; sin verificar en código) |
| Reporte (H-1) | Agrupa por valor congelado; puede dar más de una fila por producto | ✅ (decisión cerrada); CA-06.1 pendiente |

## 7. ¿Qué queda abierto, con dueño?

| Pendiente | Tarea / ID | Dueño | Estado |
|---|---|---|---|
| Confirmar los cinco `CHECK` (`price > 0`, `quantity > 0`, `category.name` no vacío, `role`, `username` en minúsculas) | T-20 | Propietario | ⏳ DISC-05 |
| `category_name` congelado | T-11 | Propietario | ⚠️ DISC-03 |
| `sold_by` → `sold_by_username` + `sold_by_user_id` (FK-4) | T-12 | Propietario | ⏳ |
| Índice parcial de `product`, trigramas y `pg_trgm` | T-13 | Propietario | ⏳ |
| Guarda explícita de moneda en `Sale.AddItem` | T-05 | Propietario | ⏳ |
| Reescribir CA-06.1 en `spec.md` | H-1 | Propietario | ⏳ DISC-09 |
| Retención del binario de imagen huérfano | H-2 | Propietario | Abierto |
| Defecto A-7 (desempate del nombre de producto en el reporte) | DP-01 | Propietario | Abierto por instrucción (§11.1) |
| Repetir las consultas de §10 contra el motor | — | Equipo con acceso al motor | ⏳ DISC-07 |
| Contradicciones DISC-01 a DISC-10 | — | Propietario del modelo | Ver [model-discrepancies.md](model-discrepancies.md) |
| Cuerpo de error, tipo de conflicto `xmin` y política de reintento | Propuestas | Propietario | Ver [errors-and-concurrency.md](errors-and-concurrency.md) |
| Licencia del repositorio | — | Equipo | **No decidida**; no se añadió ninguna |

**Cerrados por el modelo (no se reabren aquí):** T-09 (§13 D-1), A-1 (§13 D-3), H-1 y H-3 (§11).

## 8. Conclusión

**La documentación 01–05 sigue la estructura, las reglas y las marcas del modelo, pero no se puede afirmar que describa exactamente el mismo sistema.** Los motivos:
1. El modelo se contradice en puntos que esta documentación no puede resolver (DISC-01, DISC-02, DISC-03, DISC-05 y DISC-06).
2. Algunas piezas figuran como implementadas solo porque §13 lo dice, y este repositorio no pudo verificarlas contra el motor (FK-3, `sale_id NOT NULL`, índice compuesto, `deleted_at`).
3. La instantánea verificada es de 2026-09-19 y es anterior a esos cambios.
4. Varios documentos citados por el modelo no se entregan, así que no se pudo contrastar CA-06.1, ni las tareas T-xx, ni los registros D-xx y DP-xx (ver [id-index.md](id-index.md)).

Lo único que la arquitectura añade sin respaldo directo del modelo es la **forma hexagonal** y las propuestas de seguridad, errores y despliegue, marcadas como **[derivado]** o como propuesta pendiente de aprobación.

> **Verificación contra el motor:** no realizada en este repositorio. La última comprobación disponible (§10, 2026-09-19) no refleja los cambios del 2026-09-20. Si el motor difiere del modelo, **gana el motor** y el documento está roto (§10).
