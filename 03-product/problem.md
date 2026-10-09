# Problema que resuelve — Simple Stock Flow

## El problema

**[derivado]** Un negocio que lleva **catálogo y ventas** con herramientas
genéricas, sin un modelo que garantice sus reglas, puede enfrentar cuatro fallas
concretas que el modelo de datos cierra explícitamente. Estas fallas se
**deducen de las decisiones de diseño del modelo**, no de un diagnóstico
documentado del cliente:

| # | Falla | Cómo la cierra el modelo |
|---|---|---|
| 1 | El **stock puede quedar negativo** porque nada lo impide | Invariante `stock >= 0` en el dominio **y en el motor** (`ck_product_stock_non_negative`, §2.2, §4); retirar más de lo disponible falla (§2.2) |
| 2 | **Cambiar el precio reescribe el histórico**: ya no se sabe qué se cobró | Precio y nombre **congelados** en `sale_item` al momento de la venta (§1, §2.4); son hechos de la venta, no lecturas tardías |
| 3 | **No hay un instante claro de la venta**: muchas marcas de tiempo y ninguna significa «se vendió» | `sold_at` es **el único instante de negocio del sistema**; no hay `created_at`/`updated_at` (§8) |
| 4 | Sistemas contables completos exigen **multi-moneda, clientes y auditoría** que este negocio no necesita | Monomoneda por construcción (D-05); **sin entidad cliente** (§1, §7); sin columnas de auditoría (§8); producto con «nombre, precio, stock, categoría e imagen, **y nada más**» (DP-03) |

> **Corrección de la revisión.** La versión anterior hablaba de «un comerciante que
> vende productos de ferretería/construcción» y de «hojas de cálculo». **Ninguna de
> esas dos afirmaciones figura en el modelo.** Se infirió el sector a partir de las
> cinco categorías sembradas (§9.1) y no hay evidencia sobre las herramientas que
> el negocio usa hoy. Se retiraron como hecho y se dejan como supuesto abajo.

## Para quién

- **Vendedor (`seller`)**: registra productos, ajusta stock, registra ventas **[derivado]** (§2.5).
- **Administrador (`admin`)**: da de alta vendedores; el rol `admin` lo provisiona el despliegue (DP-04, §11 H-3).
- **No es para:** clientes finales (no existe su dato, §7), ni contadores
  que necesiten multi-moneda o asientos (fuera de alcance, `01-context/scope.md`).

## Consecuencias de no resolverlo

**[derivado]** Stock negativo no detectable, reportes de ventas históricos
incorrectos (por precios reescritos), imposibilidad de conciliar «qué se
cobró» en un período cerrado, y complejidad operativa innecesaria que eleva
el costo de adopción para un comercio pequeño.

## Qué **no** es el problema

Igual de importante (del modelo): no es un problema de pagos (§7: no hay
pagos ni tarjetas), ni de analítica por vendedor (DP-02 prohíbe el desglose
por operador), ni de gestión de categorías (D-10: cinco fijas, sin
mantenimiento), ni de auditoría de cambios al catálogo (§8: cerrado por
ausencia de requisito).

## Supuestos

**[supuesto]** El perfil del negocio (sector, tamaño, volumen de transacciones,
cantidad de operadores, herramientas que usa hoy) **no aparece en el modelo**. Las
decisiones de diseño —un solo servidor UTC, índices para diez patrones de acceso
(§6.1), tabla `user` con una sola fila en la instantánea (§9.2)— sugieren un
alcance **pequeño y monotenant**, pero el modelo no lo afirma.
