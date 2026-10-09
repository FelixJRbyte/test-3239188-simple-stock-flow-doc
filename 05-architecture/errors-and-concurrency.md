# Errores, transacciones y concurrencia

> **[derivado]** El modelo no define el cuerpo de error ni los códigos (viven en
> `api-contract.md`, fuera del modelo, §12). Aquí se clasifican los fallos que el
> modelo **hace posibles** y se separa lo **confirmado** de lo **propuesto**.
> No se afirma que nada de esto esté implementado ni probado.

## 1. Clasificación de errores

| Clase | Ejemplos (con fuente) | Origen | ¿Es error del usuario? |
|---|---|---|---|
| **Violación de regla de dominio** | Retirar más stock del disponible (§2.2); venta sin líneas (§2.3); producto repetido en la venta (§2.3); `Money` negativo; `Quantity` ≤ 0; rol inválido (§2.5); `DateRange` con fin anterior al inicio (§1) | Dominio (C#) | Sí |
| **Autenticación o autorización** | Sin token → 401; `seller` → 403 en el alta de usuarios (§13 D-3, reportado, no verificado) | Borde de aplicación | Sí |
| **Conflicto de concurrencia** | Dos operaciones sobre el mismo producto (testigo `xmin`) | Adaptador de persistencia | No: reintentable (propuesta) |
| **Restricción del motor que salta** | `ck_product_stock_non_negative`, `RESTRICT` de FK, índices únicos | Motor | **No.** ADR-002: «si la restricción salta, algo escribió fuera del adaptador» |
| **Fallo de infraestructura** | Motor no disponible; puerto de hash no responde; almacenamiento de imágenes falla | Adaptadores | No |
| **Dato no hallado** | Producto o venta inexistente o dado de baja (§2.2) | Aplicación | Sí |

**Regla confirmada (§7):** ningún mensaje de error, log ni proyección contiene `password_hash`.

## 2. Operación que abarca Product y Sale (`RegisterSale`)

**Confirmado (§2.3):**
- `Sale.AddItem` llama a `Product.Withdraw` **antes** de añadir la línea; descontar stock y añadir la línea son **una sola operación**.
- Una venta requiere al menos una línea (`EnsureConfirmable`) y no repite producto.
- Q3 (lectura por lote de ids, activos) precede a la escritura de stock y es el punto de contención de D-04.
- Una venta es inmutable una vez registrada.

**Tensión de diseño `[derivado]`:** la operación escribe en **dos agregados** (Catálogo y Ventas), lo que normalmente se evita en DDD. El modelo la mantiene a propósito. La arquitectura lo acepta, pero el modelo **no define** el límite transaccional de toda la venta.

**Propuestas pendientes de aprobación:**
- **P-T1** Todas las líneas de una venta y los descuentos de stock se confirman en **una sola transacción de base de datos**: o se registra todo o nada.
- **P-T2** El almacenamiento de imágenes queda **fuera** de esa transacción (coherente con §7.1).

## 3. Concurrencia con `xmin`

**Confirmado:**
- Concurrencia optimista con `xmin` como testigo, expuesto como propiedad sombra (D-04, T-10, ADR-002 `[reconstruido]`).
- `ck_product_stock_non_negative` es la última barrera (§2.2, §4, verificada en la instantánea de §10.2).
- Si esa restricción salta, se trata como fallo de integridad, no como error de uso.

**No definido por el modelo:** qué ocurre al detectar un conflicto de `xmin`: tipo de error, si se reintenta, cuántas veces y con qué espera.

**Propuestas pendientes de aprobación:**

| # | Propuesta | Motivo |
|---|---|---|
| P-C1 | Un conflicto de `xmin` aborta la operación completa y devuelve un error de conflicto reintentable | Evita ventas parciales |
| P-C2 | Reintento automático **acotado**; el máximo (N) y la espera **los fija el propietario** | El modelo no da cifras; no se inventan |
| P-C3 | Agotados los reintentos, se informa conflicto al usuario sin descontar stock | Coherente con P-T1 |
| P-C4 | Un `CHECK` de stock que salta tras agotar la ruta normal se registra como incidente de integridad | Aplica el criterio de ADR-002 |

## 4. Modos de fallo y comportamiento esperado

| Situación | Resultado esperado | Estado |
|---|---|---|
| Retirar más stock del disponible | Operación rechazada; stock sin cambios | **Confirmado** (§2.2) |
| Segunda línea de la venta falla | Ninguna línea queda registrada ni descontada | **Propuesta** P-T1 |
| Dos retiradas simultáneas sobre el mismo producto | El stock final no es negativo; una se rechaza o se reintenta | Parte **confirmada** (stock ≥ 0); el resto **propuesta** P-C1 a P-C3 |
| Motor caído a mitad de la venta | Ninguna venta parcial confirmada | **Propuesta** P-T1 |
| Fallo al borrar el binario de imagen | Binario huérfano e inofensivo; la clave ya está anulada | **Confirmado** (§7.1); retención del huérfano es H-2, abierto |
| Fallo tras anular `image_key` y antes del borrado | Igual que arriba | **Confirmado** (§7.1) |

## 5. Límites y ausencias
- Sin números concretos de reintentos, tiempos de espera ni límites de tamaño: el modelo no los da.
- Sin cuerpo ni códigos de error de la API (fuera de alcance, §12).
- Ninguna de las propuestas está aprobada, implementada ni probada. Las pruebas asociadas están en [testing-strategy.md](../04-requirements/testing-strategy.md) (TS-06, TS-07) y **no se han ejecutado**.
