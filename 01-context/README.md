# 01 — Contexto del proyecto

> **Qué es esto:** el «qué» y el «para quién» del sistema. Todo aquí se
> reconstruye **hacia atrás** desde el modelo de datos. La tabla de stack de
> `overview.md` es **solo una referencia técnica derivada del modelo**, no un
> requisito de producto.

## Regla de trazabilidad del reto

Cada afirmación cita su origen en [`spec/data-model.md`](../spec/data-model.md)
(por ejemplo «§2.3», «FK-2», «D-05», «T-20», «ADR-003»). Lo que **no** sale del
modelo está marcado como **[supuesto]**; lo deducido del contexto, como
**[derivado]**. El significado de los identificadores está en
[`05-architecture/id-index.md`](../05-architecture/id-index.md) y las
contradicciones del modelo, en
[`05-architecture/model-discrepancies.md`](../05-architecture/model-discrepancies.md).

## Archivos

| Archivo | Contenido | Estado |
|---|---|---|
| `overview.md` ⭐ | Descripción ejecutiva: qué es, qué problema resuelve, quiénes lo usan, estado actual | ✅ lleno |
| `scope.md` ⭐ | Fronteras: qué se construye y qué **no** | ✅ lleno |

La visión del producto vive en `03-product/vision.md`; el glosario, en
`02-domain/glossary.md` (es glosario de dominio, §1 del modelo).

## Correlaciones

| Si cambias esto... | Revisa también | Por qué |
|---|---|---|
| Alcance del catálogo | `02-domain/entities-and-rules.md` (Product, §2.2) | Producto tiene nombre, precio, stock, categoría e imagen, **y nada más** (DP-03) |
| Inmutabilidad de ventas | `04-requirements/user-stories.md` | No existe puerto de edición ni borrado de ventas (§2.3) |
| Roles (admin/seller) | `04-requirements/` (US-08, NFR-20) y `05-architecture/security-and-authorization.md` | Conjunto cerrado de dos (§2.5); DP-04: nadie otorga `admin` en ejecución |
| Monomoneda | `05-architecture/overview.md` | Sin columna de moneda en ninguna tabla (D-05, §3) |
