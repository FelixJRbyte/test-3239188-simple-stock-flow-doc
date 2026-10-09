# Alcance del sistema — Simple Stock Flow

Todo trazado a `spec/data-model.md`. Lo que no sale del modelo está marcado **[supuesto]**.
Estados de T-xx, A-x y H-x: [id-index.md](../05-architecture/id-index.md).

## Dentro del alcance (qué se construye)

| # | Capacidad | Origen en el modelo |
|---|---|---|
| 1 | **Catálogo de productos**: crear con nombre, precio, stock y categoría obligatoria; imagen opcional | §2.2, §3 `product` |
| 2 | **Categorías fijas**: cinco, sembradas, de solo lectura, sin CRUD | §2.1, §9.1, D-10 |
| 3 | **Mantenimiento de producto**: renombrar, cambiar precio, retirar y reponer stock | §2.2 (`Product.Rename`, `ChangePrice`, `Withdraw`, `Restock`) |
| 4 | **Baja lógica de producto** (`deleted_at`); nunca borrado físico. T-09 saldada según §13 D-1 | §2.2, §7.1, ADR-003, §13 |
| 5 | **Imagen de producto**: clave opaca en almacenamiento externo; el binario sí se borra al reemplazar o dar de baja | §2.2, §7.1, D-08 |
| 6 | **Registrar venta** con al menos una línea; descontar stock y añadir la línea son **una sola operación** | §2.3 |
| 7 | **Congelar** nombre, precio y categoría en cada línea (la categoría depende de T-11, estado en conflicto: DISC-03) | §2.4, D-06 |
| 8 | **Consultar ventas** por rango de fechas y venta con sus líneas | §6.1 (Q6, Q7) |
| 9 | **Reporte agregado de ventas** por producto sobre un rango; se calcula en el motor por un puerto de lectura; **no se desglosa por vendedor**; **agrupa por el valor congelado** (H-1), por lo que puede haber más de una fila por producto | §1, §6.1 (Q9), D-06, DP-02, §11.1 |
| 10 | **Identidad**: autenticación de operadores internos, roles `admin`/`seller`, hash de clave por puerto | §2.5, §9.2, D-09 |
| 11 | **Alta de vendedores por un administrador autenticado.** El rol `admin` **no** se otorga en ejecución: lo provisiona el despliegue desde el entorno (DP-04). El defecto A-1 (alta anónima) figura **cerrado** en §13 D-3, no verificado aquí | §2.5, §9.2, §11 (H-3), §13 D-3 |

## Fuera del alcance (qué **no** se construye, y por qué)

| # | No se hace | Origen |
|---|---|---|
| 1 | Multi-moneda o columna de moneda | D-05, §3 (regla transversal 1) |
| 2 | Entidad cliente/comprador o dato personal de comprador | §1, §7 («no existe dato personal de cliente final») |
| 3 | Columnas de auditoría `created_at`/`updated_at` ni disparador | §8 (decisión cerrada) |
| 4 | Borrado físico de productos o de ventas | §7.1, §2.3 |
| 5 | CRUD de categorías (renombrar, crear, borrar) | §2.1, D-10; condición de revisión en §4.1 |
| 6 | Reporte por vendedor / analítica por operador | DP-02, §6.3, §7.1 |
| 7 | Pagos, tarjetas o medios de pago | §7 («no hay pagos ni tarjetas») |
| 8 | Versionado ni histórico de hash de contraseña | §7.1 |
| 9 | Anonimización para analítica externa | §7.1 (omisión consciente: no hay analítica externa) |
| 10 | Tablas de reporte, auditoría, contadores o de objetos de valor | §2 («sin excedente»), D-07 |
| 11 | Otorgar el rol `admin` desde la aplicación | DP-04, §11 (H-3) |

**[supuesto]** Funcionalidades comunes en otros sistemas (notificaciones, multi-
tienda, código de barras, importación masiva) no aparecen en el modelo: quedan
fuera por ausencia de requisito, igual que DP-03 cierra los atributos del producto.

## Supuestos del alcance

| # | Supuesto | Consecuencia si cambia |
|---|---|---|
| 1 | Las cinco categorías son fijas y de solo lectura (D-10) | Si se abre mantenimiento, la decisión de unicidad sensible a acentos se revisa **antes** de ese CRUD (§4.1) |
| 2 | El servidor de base de datos corre en UTC (§3) | Una columna de fecha nueva hereda `timestamptz` (regla transversal 2, §3) |
| 3 | El esquema lo poseen solo las migraciones EF (ADR-001, §3.2) | Cualquier DDL manual rompe la gobernanza del esquema |
| 4 | `sold_at` es el único instante de negocio (§8) | Un requisito real de auditoría vuelve al propietario y se resuelve con bitácora, no con columnas (§8) |

## Restricciones

| Tipo | Restricción | Origen |
|---|---|---|
| **Técnica** | C# + PostgreSQL 16 + EF Core; Docker Compose para infra | §2, §3, §3.2, §10 |
| **De datos** | Sin `DEFAULT`; tablas en singular; esquema `sales`. El número de columnas está sin resolver (21 en la instantánea 2026-09-19 / 22 en §3, §10, §12; DISC-01) | §3, §0 |
| **De privacidad** | `password_hash` jamás en logs/respuestas/índices; `username` y `sold_by` de acceso restringido | §7 |
| **De plazo** | Entrega del reto: 8 de octubre de 2026, 10:50 p. m. (hora Colombia) — vale el último commit antes de esa hora | README del reto |

**[supuesto]** Normativa aplicable (protección de datos personales): el modelo
clasifica atributos (§7) pero no cita ley alguna. Si el equipo decide mencionar
Ley 1581 de 2012 (Colombia), es un supuesto del equipo, no del modelo.

**Licencia del repositorio:** **no decidida**. No hay información en el repositorio
sobre qué licencia corresponde; **se omitió a propósito** y debe decidirla el equipo.
