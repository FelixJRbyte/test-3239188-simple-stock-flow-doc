# Requisitos no funcionales — Simple Stock Flow

> Cada NFR nace de una decisión del modelo. **[supuesto]** marca
> lo que el modelo no dice.
>
> **Revisión posterior a la auditoría.** Los requisitos de rendimiento distinguen
> ahora el **estado actual** (lo que el modelo afirma) del **objetivo deseado**.
> **No se ha ejecutado ninguna prueba de rendimiento** y no se afirma que los
> índices funcionen. Escenarios en [testing-strategy.md](testing-strategy.md).

## Datos e integridad

| # | Requisito | Origen |
|---|---|---|
| NFR-01 | `stock >= 0` se cumple **siempre**: dominio + motor (`ck_product_stock_non_negative`, verificada en la instantánea de §10.2) | §2.2, §4, ADR-002 |
| NFR-02 | Si la restricción de stock salta, **algo escribió fuera del adaptador** — criterio de concurrencia de ADR-002 | § «Cómo se lee», ADR-002 |
| NFR-03 | Ninguna columna tiene `DEFAULT`: los valores los pone el dominio, nunca el motor | §3 |
| NFR-04 | Sin columna de moneda en ninguna tabla; sistema monomoneda | D-05, §3 |
| NFR-05 | Todas las marcas de tiempo son `timestamptz`; servidor en UTC | §3 (regla transversal 2) |
| NFR-06 | El esquema lo poseen **solo las migraciones EF**: ningún DDL manual | §3.2, ADR-001 |
| NFR-07 | Ventas y líneas: retención **indefinida**, nunca se borran ni se editan | §7.1 |
| NFR-08 | Productos: baja lógica, **nunca** borrado físico | §7.1, ADR-003 |
| NFR-09 | Un borrado físico de producto referenciado por una línea debe **fallar ruidosamente** (FK-3 `RESTRICT`). **Estado:** FK-3 declarada; **implementada según §13 D-2**, no verificada; §3, §4 y §5 del modelo aún la dan por inexistente (DISC-04) | §5, ADR-003 |

## Concurrencia

| # | Requisito | Origen |
|---|---|---|
| NFR-10 | Concurrencia **optimista** en el stock: testigo `xmin` (columna de sistema, expuesta como propiedad sombra). **No definido:** el tipo de error y la política de reintento ante un conflicto (propuestas en [errors-and-concurrency.md](../05-architecture/errors-and-concurrency.md)) | §3 (`product.xmin`), D-04, ADR-002 |
| NFR-11 | La lectura previa a escritura de stock (Q3, lote de ids) es el **punto de contención** y debe soportarlo. **[supuesto]** el modelo no fija una meta medible (ver Supuestos) | §6.1 |

## Rendimiento

| # | Requisito | Origen |
|---|---|---|
| NFR-12 | **Objetivo:** cada uno de los diez patrones de acceso (Q1–Q10) está servido por un índice previsto y cada índice existe por una consulta concreta. **Estado actual:** *Verificados* (2026-09-19): `IX_sale_sold_at`, `IX_sale_item_product_id`, `IX_category_name`, `IX_user_username`, `IX_product_category_id`, `IX_product_name`, `IX_sale_item_sale_id`. *Según §13 D-2, sin verificar:* índice único compuesto de `sale_item`. *Pendientes (T-13):* índice parcial de `product` y trigramas. **No se ha ejecutado ningún plan de ejecución**: «servido por índices» es una intención de diseño, no un hecho comprobado | §6.1, §6.2, §13 D-2 |
| NFR-13 | **Objetivo:** el reporte agregado (Q9, el **más costoso**) no toca la tabla `sale_item`, gracias al índice único `(sale_id, product_id)` con `INCLUDE (quantity, unit_price)`. **Estado actual:** ese índice figura como implementado según §13 D-2 (T-20), pero §6.2 lo llama «falta (T-13)» (DISC-06). Ninguna prueba lo ha verificado. El propio modelo advierte que, en una tabla que solo recibe inserciones, el recorrido solo-índice depende del mantenimiento automático (§6.2) | §6.2, §13 D-2 |
| NFR-14 | **Objetivo:** la búsqueda por texto parcial (Q1) usa trigramas, porque ningún árbol B sirve un comodín a la izquierda. **Estado actual:** **pendiente (T-13)**; `pg_trgm` no estaba instalada en la instantánea y la instalará la propia migración del índice | §6.2 |
| NFR-15 | Un índice de más encarece cada escritura para siempre: los descartados están justificados uno por uno | §6.3 |

## Privacidad y seguridad

| # | Requisito | Origen |
|---|---|---|
| NFR-16 | `password_hash`: **jamás** en logs, respuestas, proyecciones ni mensajes de error; **nunca se indexa**; su única lectura legítima es verificar, por el puerto de hash | §7 |
| NFR-17 | `username` y `sold_by`/`sold_by_username`: **dato personal**, acceso restringido; admisible en auditoría; no en respuestas anónimas ni endpoints públicos | §7 |
| NFR-18 | `role`: confidencial interno (revela el nivel de privilegio); no es público | §7 |
| NFR-19 | El dominio **nunca ve la clave en claro**; el hash lo produce un puerto (D-09) | §2.5 |
| NFR-20 | El alta de usuarios **no puede ser anónima** y **nadie otorga el rol `admin` en ejecución** (lo provisiona el despliegue, DP-04). **Estado:** el defecto A-1 está **cerrado según §13 D-3** (sin token → 401; `seller` → 403), no verificado aquí; §9.2 aún lo da por roto (DISC-08) | §9.2, §11 (H-3), §13 D-3 |
| NFR-21 | No existe dato personal de cliente final: la venta registra al **operador interno**; la superficie de privacidad es deliberadamente pequeña y conviene no ampliarla sin requisito | §7 |

## Auditoría y retención

| # | Requisito | Origen |
|---|---|---|
| NFR-22 | `sold_at` es **el único instante de negocio** del sistema | §8 |
| NFR-23 | El sistema **no lleva** columnas de auditoría (`created_at`/`updated_at`) ni disparadores — decisión cerrada con cuatro motivos | §8 |
| NFR-24 | Si aparece un requisito real de auditoría («¿quién cambió este precio y cuándo?»), vuelve al **propietario** y se resuelve con bitácora de cambios — decisión de alcance, no de esquema | §8 |

## Calidad del modelo

| # | Requisito | Origen |
|---|---|---|
| NFR-25 | Toda regla lleva marca — **motor**, **solo dominio** o **pendiente (T-xx)**; «lo que no está aplicado se declara pendiente; no se promete». Este repositorio añade la marca **en conflicto** para contradicciones sin resolver | § «Cómo se lee» |
| NFR-26 | El documento es verificable contra el motor con las consultas de §10. **Estado:** la última verificación (2026-09-19) dio 21 columnas, 8 restricciones y 12 índices; **es anterior a los cambios de §13** y la cifra de columnas es contradictoria (21 frente a 22, DISC-01, DISC-07). No hay una verificación vigente. Si el motor difiere, **gana el motor** y el documento está roto | §10 |

## Supuestos

**[supuesto]** Metas de latencia, disponibilidad (SLA), volumen de
transacciones, despliegue multi-entorno y copias de seguridad **no
aparecen en el modelo** — solo se infiere un alcance pequeño
(una fila en `user`, servidor UTC único, §9.2 y §3). El equipo debe
fijarlos antes de `05-architecture/` si el reto lo requiere. Ver
[deployment-and-configuration.md](../05-architecture/deployment-and-configuration.md).
