# Eventos de dominio — Simple Stock Flow

> **Advertencia de trazabilidad:** el modelo de datos (`spec/data-model.md`)
> **no define eventos de dominio ni broker de mensajes**. Lo que sigue está
> **derivado** de las invariantes y operaciones del modelo (§2) y se marca
> como **[derivado]** según la regla 3 del reto. Si el equipo adopta eventos,
> estas son las candidatas que el modelo justifica; no hay requisito de
> infraestructura de mensajería en el alcance.

## Eventos candidatos (derivados de §2)

| Evento | Ocurre cuando | Evidencia en el modelo |
|---|---|---|
| `ProductRegistered` | Se crea un producto (nombre, precio, stock, categoría, imagen opcional) | §2.2 — `Product` es raíz de agregado con ciclo de creación |
| `ProductRenamed` | `Product.Rename` | §2.2 |
| `ProductPriceChanged` | `Product.ChangePrice` (precio > 0) | §2.2 |
| `StockWithdrawn` | `Product.Withdraw` (falla si supera el disponible) | §2.2 |
| `StockRestocked` | `Product.Restock` | §2.2 |
| `ProductImageAttached` | `Product.AttachImage` (blanco → `null`) | §2.2 |
| `ProductLogicallyDeleted` | Baja lógica (`deleted_at`) — nunca borrado físico | §2.2, §7.1, ADR-003, T-09 |
| `SaleRegistered` | `Sale` confirmada: **al menos una línea**, stock descontado y línea añadida en **una sola operación** | §2.3 — `Sale.AddItem` + `EnsureConfirmable` |
| `UserRegistered` | Alta de usuario desde la aplicación: rol `seller` (el rol `admin` lo provisiona el despliegue, DP-04); hash por puerto | §2.5, §9.2 |

## Eventos que **no** existen, y por qué (derivado)

| No hay evento de... | Razón del modelo |
|---|---|
| Edición o borrado de venta | **No existe la operación** (§2.3: inmutabilidad por ausencia de puerto) |
| Cambio de categoría | No hay CRUD de categorías (§2.1, D-10) |
| Creación de línea suelta | `SaleItem` solo se construye dentro de `Sale.AddItem` (§2.4) |

## Notas de diseño (marcadas)

- **Nombre en pasado y singular**, inmutable — convención DDD.
- **[derivado]** `SaleRegistered` es el único evento con consecuencias de
  negocio evidentes (el reporte agrega por producto, §6.1 Q9); los demás
  son candidatos de auditoría, y **el proyecto no lleva auditoría** (§8):
  emitirlos hoy sería alcance inventado, exactamente lo que DP-03 evita.
- **[derivado]** Si se adoptaran, el transporte (broker, outbox) es decisión
  de `05-architecture/`; **no hay nada en el modelo que lo exija**.
- **[derivado]** `SaleRegistered` **no** debe exponer `password_hash` ni
  datos personales más allá de lo necesario: `username`/`sold_by` son dato
  personal de acceso restringido (§7).
