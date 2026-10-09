# 04 — Requisitos

> **Qué es esto:** historias de usuario y requisitos no funcionales
> que el modelo de datos **hace necesarios**. Cada historia traza a
> la operación o invariante que la justifica.

| Archivo | Contenido | Estado |
|---|---|---|
| `user-stories.md` ⭐ | Historias de usuario con criterios y escenarios de aceptación (Dado/Cuando/Entonces) | ✅ revisado |
| `non-functional-requirements.md` ⭐ | NFRs (datos, concurrencia, privacidad, rendimiento), con estado actual separado del objetivo | ✅ revisado |
| `testing-strategy.md` | Estrategia **documental** de pruebas (TS-01 a TS-22); **ninguna ejecutada** | ✅ documental |

## Método

Una historia existe solo si el modelo la soporta: una operación de
dominio (§2), un patrón de acceso (§6.1) o una política (§5, §7, §9).
Lo que el modelo no soporta **no es requisito** (DP-03 aplica al
alcance completo). Lo que el modelo no define va marcado **[supuesto]** o
**[pendiente de confirmación]**.

## Trazabilidad inversa

| Requisito del modelo | Historia |
|---|---|
| `Product` raíz de agregado (§2.2) | US-01…US-04, US-11 |
| `Sale`/`SaleItem` (§2.3, §2.4) | US-05…US-07 |
| `User` (§2.5) | US-08…US-09 |
| `Category` solo lectura (§2.1) | US-10 |
| Patrones Q1–Q10 (§6.1) | US-04, US-06, US-07, US-09, US-10 |
| A-1 y DP-04 (§9.2, §11, §13 D-3) | US-08 |
| H-1 (§11.1) | US-07 |

Para el significado de cada identificador, ver
[`../05-architecture/id-index.md`](../05-architecture/id-index.md).
