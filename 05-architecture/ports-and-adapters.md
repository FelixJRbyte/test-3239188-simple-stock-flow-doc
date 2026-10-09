# Puertos y adaptadores

> **[derivado]** El modelo menciona puertos (§2.5 D-09, §1 D-06,
> §6.1) y adaptadores (§0), pero no los lista. Esta es la lista
> que el modelo **implica**, con la evidencia de cada uno.

## Puertos de dominio (escritura)

| Puerto | Operaciones | Evidencia |
|---|---|---|
| `IProductRepository` | Crear, `Rename`, `ChangePrice`, `Withdraw`, `Restock`, `AttachImage`, baja lógica; lecturas Q1, Q2, Q3 | §2.2 (operaciones), §6.1 |
| `ICategoryRepository` | **Solo lectura**: Q4 (listado), Q5 (por id). Ningún puerto crea, renombra ni borra | §2.1, §6.1 |
| `ISaleRepository` | Registrar venta con líneas (transacción con `Product.Withdraw`); Q6 (venta con líneas), Q7 (por rango) | §2.3, §6.1 |
| `IUserRepository` | Crear usuario; Q10 (por nombre, igualdad exacta) | §2.5, §6.1 |

## Puertos de infraestructura

| Puerto | Responsabilidad | Evidencia |
|---|---|---|
| `IPasswordHashPort` | Producir y verificar el hash. **El dominio nunca ve la clave en claro**; su única lectura legítima es verificar | §2.5, §7, D-09 |
| `IImageStoragePort` | Guardar, resolver y **eliminar** el binario. La base solo guarda `image_key` (clave opaca) | §2.2, §7.1, D-08 |
| `ISalesReportReadPort` | Reporte agregado por producto sobre un rango (Q9). **Se calcula en el motor**; no se persiste | §1, §6.1, D-06 |

## Adaptadores

| Adaptador | Detalle del modelo |
|---|---|
| **EF Core** (persistencia) | Traducir tabla (singular) ↔ colección C# (plural) es su responsabilidad (§0). Las propiedades sombra son suyas: `deleted_at` (T-09), `xmin` (T-10), `sale_id` (defecto de `HasForeignKey` sin `IsRequired()`, §3) |
| **Hash** | Implementa `IPasswordHashPort`; el administrador inicial se crea en el arranque con credenciales de entorno — **no** se siembra desde SQL (§9.2) |
| **Almacenamiento de imágenes** | Externo. Orden de borrado: anular `image_key` → confirmar → borrar binario (§7.1). **No** participa en la transacción de la base; no se promete atomicidad |
| **Reporte (motor)** | SQL en el motor; con T-13 el índice único `INCLUDE (quantity, unit_price)` permite agregar **sin tocar la tabla** (§6.2) |

## Reglas de frontera

1. **El caso de uso `RegistrarVenta` es transaccional** sobre dos
   agregados: `Product.Withdraw` antes de `Sale.AddItem` (§2.3).
2. **El dominio no habla con el motor directamente**: toda regla
   «solo dominio» depende de que todo el mundo pase por el
   adaptador — por eso T-20 las baja al motor (§4).
3. **El puerto de lectura no expone datos personales**: el reporte
   agrega por producto, no por operador (DP-02, §7.1).
4. **Q8 (rango sin paginar) no tiene consumidor** si el reporte
   agrega en el motor: conviene retirarlo del puerto en vez de
   dejarlo como trampa (§6.1).
