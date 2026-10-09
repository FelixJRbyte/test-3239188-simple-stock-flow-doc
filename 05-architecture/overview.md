# Arquitectura general — Simple Stock Flow

> **[derivado]** El modelo no contiene un documento de
> arquitectura (`architecture.md` se cita pero no se entrega).
> Lo que sigue reconstruye la forma del sistema **desde las
> decisiones del modelo**, citando cada una.

## Estilo: hexágono (Puertos y Adaptadores)

**Evidencia:** el modelo separa de forma consistente *qué
garantiza el dominio* (C#) de *cómo se persiste* (motor) y
de *qué produce la infraestructura* (hash, imagen, reporte)
— §2, §2.5, §1, §0.

```
                    ┌──────────────────────────────────────┐
                    │           Casos de uso                │
                    │  RegistrarProducto · RegistrarVenta  │
                    │  AjustarStock · RegistrarUsuario …  │
                    └──────▲───────────────▲───────────────┘
                           │               │
              puertos (interfaz del dominio)
                           │               │
        ┌──────────────────┴───┐     ┌─────┴─────────────────┐
        │   DOMINIO (C#)        │     │   INFRAESTRUCTURA      │
        │  Agregados:           │     │  Adaptador EF (DDL:    │
        │   Product, Sale,      │◄────│   solo migraciones,    │
        │   User; SaleItem      │     │   ADR-001)             │
        │  Referencia: Category │     │  Puerto de hash (D-09) │
        │  VO: Money, Quantity  │     │  Almacén de imágenes   │
        │  Reglas con marca     │     │   (D-08, binario externo)│
        └───────────────────────┘     │  Puerto de lectura     │
                                      │   (reporte en motor,  │
                                      │   D-06)                │
                                      └────────────────────────┘
                                            │
                                      ┌─────┴──────┐
                                      │ PostgreSQL  │
                                      │ sales (UTC) │
                                      └────────────┘
```

## Agregados y raíces (§2)

| Agregado | Raíz | Internos | Regla de consistencia (toda en el agregado) |
|---|---|---|---|
| **Catálogo** | `Product` | — | `price > 0`, `stock >= 0`, categoría obligatoria, `image_key` normalizado (§2.2) |
| **Ventas** | `Sale` | `SaleItem` | ≥ 1 línea, sin producto repetido, **stock descontado y línea añadidos en una sola operación** (§2.3) |
| **Identidad** | `User` | — | username único y normalizado, rol válido, hash obligatorio (§2.5) |
| **Referencia** | — (`Category` no es raíz) | — | Solo lectura; cinco filas sembradas (§2.1, §9.1) |

**Dónde viven los objetos de valor:** dentro de la fila de su
dueño, sin identidad ni tabla (D-07, §2): `Money` en
`product.price` y `sale_item.unit_price`; `Quantity` en
`sale_item.quantity`; `DateRange` en la capa de aplicación.

## Transacción que cruza agregados (§2.3)

`RegistrarVenta` invoca `Product.Withdraw` **antes** de añadir
la línea: **una sola operación** que abarca el agregado Catálogo
y el agregado Ventas. Es la regla que da sentido al agregado
(§2.3) y define el límite transaccional del caso de uso.

## Concurrencia: optimista con `xmin` (§3, D-04, ADR-002)

- `xmin` es columna **de sistema** del motor (no del esquema,
  no cuenta entre las 21): Postgres la incrementa en cada
  `UPDATE` (§3).
- Se expone como **propiedad sombra** (T-10) y es el testigo
  de concurrencia de D-04.
- **Criterio de ADR-002:** si la restricción `stock >= 0`
  salta, algo escribió fuera del adaptador (§ «Cómo se lee»).

## Esquema y despliegue

- Esquema `sales` de la base `simple_stock_flow`, PostgreSQL
  16.14, servidor en **UTC** (encabezado, §3).
- El DDL lo poseen **solo las migraciones EF** (§3.2, ADR-001);
  el historial vive en `public."__EFMigrationsHistory"`, fuera
  del esquema `sales` (§3.2).
- Infraestructura levanta el motor con Docker Compose
  (`simple-stock-flow-infra`, §10): **levanta el motor, no
  define el esquema** (§6.2).

## Decisiones estructurales citadas (ver `decisions/records/`)

| ADR | Decisión | Cita |
|---|---|---|
| ADR-001 | El esquema lo poseen las migraciones EF y nada más | §3.2, §6.2 |
| ADR-002 | Concurrencia optimista; `stock >= 0` como última barrera | §2.2, §4 |
| ADR-003 | Baja lógica (`deleted_at`), nunca borrado físico; FK-3 como barrera de última instancia | §2.2, §5, §7.1 |
| ADR-004 | Reporte agregado y congelado: categoría congelada **sin FK**; el reporte agrega en el motor | §2.4, §3, §6.2, D-06 |
