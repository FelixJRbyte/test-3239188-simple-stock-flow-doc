# Mapa del dominio — Contextos acotados

> **Trazabilidad:** el modelo de datos define **cinco entidades y tres
> agregados** (§2), pero **no define contextos acotados**. La agrupación
> de abajo es una **interpretación [derivada]** del modelo, hecha siguiendo
> DDD: cada agregado raíz es un contexto acotado natural, y `Category`
> es un subdominio de referencia (§2.1).

## 1. Visión general del dominio

Simple Stock Flow gestiona un **catálogo de productos** y el **registro de
ventas** con reglas estrictas y un esquema mínimo:
productos con nombre, precio, stock y categoría (§2.2); ventas inmutables con
líneas de precio congelado (§2.3, §2.4); operadores internos con dos roles
(§2.5). Todo en **una sola moneda** (D-05) y **sin datos de cliente** (§1, §7).

## 2. Contextos acotados [derivado del modelo]

### Catálogo (`Product` — raíz de agregado, §2.2)

| Campo | Valor |
|---|---|
| Responsabilidad | Registrar y mantener productos: nombre, precio > 0, stock ≥ 0, categoría obligatoria, imagen opcional |
| Tabla | `product` (esquema `sales`) |
| Lenguaje ubicuo | Producto, precio, stock, imagen (clave opaca), categoría |
| Frontera | Lo referencia `SaleItem` por identidad de raíz mediante **FK-3** (declarada; implementada según §13 D-2, no verificada): la línea **congela** nombre y precio en lugar de leer tarde (§1, §2.4) |

### Ventas (`Sale` + `SaleItem` — raíz y entidad interna, §2.3, §2.4)

| Campo | Valor |
|---|---|
| Responsabilidad | Registrar el hecho comercial consumado: quién, cuándo y qué. Inmutable |
| Tabla | `sale` + `sale_item` (composición: FK-2 `CASCADE`) |
| Lenguaje ubicuo | Venta, línea, cantidad, precio congelado, total (calculado, §1) |
| Frontera | **Una sola operación**: descontar stock y añadir la línea (§2.3) — cruza al Catálogo invocando `Product.Withdraw` |

### Identidad (`User` — raíz de agregado, §2.5)

| Campo | Valor |
|---|---|
| Responsabilidad | Operadores internos: autenticación y autorización; roles cerrados `admin`/`seller` |
| Tabla | `user` |
| Lenguaje ubicuo | Usuario, nombre de usuario (minúsculas), hash de clave, rol |
| Frontera | La venta registrará autoría por **FK-4** (`sale.sold_by_user_id`, **pendiente T-12**); **no hay entidad cliente** (§1) |

### Referencia (`Category` — entidad de referencia, §2.1)

| Campo | Valor |
|---|---|
| Responsabilidad | Clasificar productos. **Cinco filas fijas, sembradas, solo lectura** (D-10) |
| Tabla | `category` |
| Nota | No es raíz de agregado ni tiene ciclo de vida; su repositorio es de **solo lectura** |

## 3. Mapa de contextos [derivado]

```
        ┌────────────┐
        │  Category   │  referencia, solo lectura (§2.1)
        │  (5 fijas)  │
        └─────▲───────┘
              │ FK-1 RESTRICT (§5)
              │
        ┌─────┴───────┐
        │   Product    │  catálogo (§2.2)
        │  (catálogo)  │
        └─────▲───────┘
              │ FK-3 RESTRICT: sale_item.product_id → product.id
              │ (declarada; implementada según §13 D-2)
              │ + Product.Withdraw (§2.3)
        ┌─────┴───────────────┐        ┌────────────┐
        │        Sale          │◄───────│    User     │  autoría de la venta
        │  ┌────────────────┐  │ FK-4   │ (identidad) │  sale.sold_by_user_id → user.id
        │  │    SaleItem     │  │ RESTRICT│            │  pendiente T-12
        │  │ precio congelado│  │        └────────────┘
        │  └────────────────┘  │
        └──────────────────────┘   FK-2 CASCADE: sale_item.sale_id → sale.id (composición)
```

| Relación | Naturaleza | Origen |
|---|---|---|
| Category → Product | 1:N, cruce de agregado por identidad de raíz (FK-1) | §5 |
| Sale → SaleItem | 1:N, **interna al agregado** (composición, FK-2) | §5 |
| SaleItem → Product | N:1, cruce de agregado; línea apunta a producto existente y **no dado de baja** (FK-3: declarada; implementada según §13 D-2; DISC-04) | §5 |
| Sale → User | N:1, cruce de agregado por identidad; la autoría no puede quedar huérfana (FK-4: **pendiente T-12**) | §5 |

## 4. Clasificación estratégica [derivado]

| Contexto | Tipo DDD | Justificación |
|---|---|---|
| Ventas (`Sale`/`SaleItem`) | **Core** | La razón de ser del sistema: el flujo de venta |
| Catálogo (`Product`) | **Core** | Sin producto no hay venta; cada venta necesita uno (§5) |
| Identidad (`User`) | **Supporting** | Necesaria para el core (autoría, §5 FK-4) pero no diferenciadora |
| Referencia (`Category`) | **Supporting** | Clasificación fija, sembrada, sin mantenimiento (§2.1, D-10) |

> No hay subdominio genérico externo: la imagen vive en almacenamiento
> externo vía clave opaca (D-08) y el hash de clave es un puerto (D-09).

## 5. Decisiones de modelado registradas en el modelo

| Decisión | Alternativa descartada | Razón (con cita) |
|---|---|---|
| `Sale` y `SaleItem` separados | Una sola tabla | Composición explícita FK-2 `CASCADE` e invariante «al menos una línea» (§2.3, §5) |
| Categorías fijas, sin CRUD | CRUD de categorías | D-10: sembrado, sin mantenimiento; abrirlo revisa primero la decisión de unicidad (§4.1) |
| Precio congelado en la línea | Leer precio del catálogo al facturar | El precio vendido es un **hecho de la venta** (§1 «Nombre congelado») |
| Sin columnas de auditoría | `created_at`/`updated_at` + disparador | §8: sin requisito, contradice «sin `DEFAULT`», ningún puerto las leería |
| `category_name` congelado **sin FK** | FK a `category` | Renombrar la categoría reescribiría el histórico — exactamente lo que ADR-004 prohíbe (§3, D-06). Su estado respecto a T-11 está en conflicto (DISC-03) |
| Reporte agrupa por el valor congelado | Elegir «el más reciente» del rango | H-1 (§11.1): un reporte cerrado no debe cambiar nunca; puede haber más de una fila por producto |
| Sin entidad cliente | Tabla `customer` | No existe dato personal de cliente final; la superficie de privacidad es deliberadamente pequeña (§7) |
| Monomoneda | Columna de moneda | D-05: no se reintroduce (§3, regla transversal 1) |
| Nadie otorga `admin` en ejecución | Que el administrador de la app lo otorgue | DP-04 (§11, H-3): lo provisiona el despliegue desde el entorno |

## 6. Cómo se actualiza este mapa

1. Antes de añadir una entidad, verificar si pertenece a un contexto existente
   (el modelo prohíbe el excedente: §2 «cinco entidades, cinco tablas, sin excedente»).
2. Si el lenguaje ubicuo de un contexto cambia, revisar si el contexto debe dividirse.
3. Sincronizar con `05-architecture/` (agregados → componentes) y con `04-requirements/`
   (reglas → criterios de aceptación).
4. **Condición de revisión explícita del modelo:** si se abre el mantenimiento de
   categorías, la decisión de unicidad sensible a acentos se revisa **antes** (§4.1).
