# Visión del producto — Simple Stock Flow

## Declaración de visión **[derivado]**

> Para negocios que necesitan registrar ventas con confianza,
> Simple Stock Flow es un sistema de catálogo y ventas que **garantiza las
> reglas de negocio en los datos mismos** — stock nunca negativo, precio
> congelado en el momento de la venta, ventas inmutables — sin la complejidad
> de multi-moneda, clientes ni auditoría que no necesitan.

## Propuesta de valor

| Valor | Evidencia en el modelo |
|---|---|
| **Reglas que no se pueden saltar** | Cada invariante tiene dueño declarado: **motor**, **solo dominio** o **pendiente T-xx** (§ «Cómo se lee», §4). Las cinco invariantes «solo dominio» dependen aún de pasar por el adaptador |
| **Verdad histórica** | Precio, nombre y categoría **congelados** en la línea (§2.4); el histórico de períodos cerrados no se reescribe nunca (la categoría congelada depende de T-11, en conflicto: DISC-03) |
| **Esquema mínimo y honesto** | Cinco tablas, sin `DEFAULT`, sin excedente (§2, §3); «si algo no sale del modelo, es supuesto». El número exacto de columnas está sin resolver (DISC-01) |
| **Privacidad pequeña por diseño** | No hay dato de cliente final (§7); `password_hash` jamás se indexa, loguea ni expone (§7) |
| **Consultas que el motor sostiene** | Diez patrones de acceso reales, cada índice justificado por una consulta (§6.1, §6.2). **Objetivo de diseño; no se ha medido** |

## Diferenciadores (derivados de las decisiones del modelo)

1. **Congelamiento deliberado, no desnormalización descuidada** — el precio
   vendido es un hecho de la venta (§1), lo que permite reprecificar el
   catálogo sin tocar el histórico.
2. **Deuda declarada, no oculta** — lo que no está aplicado se declara pendiente
   (§ «Cómo se lee»). Hoy siguen sin confirmar los cinco `CHECK` de T-20 (§4, DISC-05).
3. **Barreras que fallan ruidosamente** — FK-3 existe para que un borrado
   manual **falle** en vez de corromper el histórico (§5, ADR-003). Está
   **implementada según §13 D-2** y **no verificada** (DISC-04).
4. **Alcance que se niega a crecer solo** — DP-03 cierra los atributos del
   producto; §8 cierra la auditoría; D-05 cierra la moneda.

## Frontera del producto

- **Dentro:** catálogo (con imágenes por clave opaca), ventas inmutables,
  reporte agregado por producto, identidad de operadores (§1, §2, §6.1).
- **Fuera:** multi-moneda, clientes, pagos, auditoría de cambios, CRUD de
  categorías, analítica por vendedor (`01-context/scope.md`).

## Métricas de éxito **[supuesto]**

El modelo **no define métricas de producto**. El equipo podría proponer:

| Métrica candidata | Vínculo con el modelo |
|---|---|
| Cero ventas con stock negativo | `ck_product_stock_non_negative` (§2.2) — violación = algo escribió fuera del adaptador (ADR-002) |
| Cero líneas de venta sin precio congelado | Invariante de construcción de `Sale.AddItem` (§2.4) |
| Cero filas `product` con `price <= 0` | Hueco conocido: **solo dominio** hasta que se confirme el `CHECK` de T-20 (§2.2, DISC-05) |

## Hoja de ruta **[derivado]** — con el estado que reporta el modelo

> «Según §13» = el modelo lo afirma (2026-09-20); este repositorio **no lo verificó**.
> El detalle de cada identificador está en [id-index.md](../05-architecture/id-index.md).

| Etapa | Qué | Estado | Origen |
|---|---|---|---|
| — | Baja lógica: columna `deleted_at` + filtro global | **Saldada según §13 D-1** | T-09, ADR-003 |
| — | `sale_id NOT NULL`, único `(sale_id, product_id)` con `INCLUDE`, FK-3 | **Saldadas según §13 D-2** | T-20 |
| — | Defecto A-1 (alta de usuarios anónima) | **Cerrado según §13 D-3** | §9.2, §13 |
| — | Decisión DP-04 (nadie otorga `admin` en ejecución) | **Decidida** | §11 (H-3) |
| — | Decisión H-1 (reporte agrupa por el valor congelado) | **Decidida** | §11.1 |
| 1 | Cinco `CHECK` (`price > 0`, `quantity > 0`, `category.name` no vacío, `role`, `username` en minúsculas) | **Sin confirmar** (DISC-05) | T-20, §4 |
| 2 | `category_name` congelado en la línea | **Estado en conflicto** (DISC-03) | T-11, §2.4, §3 |
| 3 | Autoría por usuario: `sold_by` → `sold_by_username` + `sold_by_user_id` (FK-4) | **Pendiente** | T-12, §3, §5 |
| 4 | Índice parcial de `product`, índice de trigramas y `pg_trgm` | **Pendiente** | T-13, §6.2 |
| 5 | Guarda explícita de moneda en `Sale.AddItem` | **Pendiente** | T-05, §2.3 |
| 6 | Reescribir CA-06.1 («una fila por producto y etiqueta congelada», propuesta de §11.1) | **Decisión pendiente del propietario** (`spec.md` no se entrega) | H-1, DISC-09 |
| 7 | Retirar Q8 del puerto (ventas por rango sin paginar no tiene consumidor) | **Propuesta** del modelo | §6.1 |
| 8 | Política del binario de imagen huérfano | **Abierto** | H-2, §11 |
| 9 | Defecto A-7 en el desempate del nombre de producto | **Abierto** por instrucción del propietario | DP-01, §11.1 |

**[supuesto]** Versiones futuras (pagos, clientes, multi-tienda) **no**
tienen respaldo en el modelo: son decisiones de producto pendientes, no
deuda técnica.
