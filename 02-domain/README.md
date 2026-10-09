# 02 — Dominio del problema

> **Qué es esto:** el modelo mental del negocio, antes de la tecnología.
> Metodología DDD: lenguaje ubicuo, entidades, objetos de valor, agregados,
> eventos de dominio.

## Estado

| Archivo | Contenido | Estado |
|---|---|---|
| `glossary.md` ⭐ | Glosario del dominio (negocio ↔ técnico) | ✅ desde §1 del modelo |
| `domain-map.md` ⭐ | Contextos acotados y mapa de contextos | ✅ interpretación del modelo |
| `entities-and-rules.md` ⭐ | Las 5 entidades, objetos de valor, agregados e invariantes | ✅ desde §2 |
| `domain-events.md` ⭐ | Eventos de dominio | ⚠️ [derivado] (el modelo no los define; no hay requisito de mensajería) |

## Conceptos clave

- **Entidad:** identidad propia (`Sale` por su `id`, sin importar su estado).
- **Objeto de valor:** sin identidad; vive dentro de la fila de su dueño (`Money`,
  `Quantity`) — D-07, §2.
- **Agregado:** unidad de consistencia. Raíces: `Product`, `Sale`, `User`.
  `SaleItem` vive **dentro** del agregado `Sale` (§2.4).
- **Entidad de referencia:** `Category` — no es raíz de agregado ni tiene ciclo
  de vida (§2.1).
- **Regla y dónde vive:** toda invariante tiene marca — **motor**, **solo dominio**
  o **pendiente (T-xx)** (§ «Cómo se lee este documento»). Esa marca es el
  corazón de este dominio.

## Correlaciones

| Este archivo alimenta... | Por qué |
|---|---|
| `05-architecture/` | Agregados → componentes; invariantes → dónde vive cada regla |
| `04-requirements/` | Invariantes → criterios de aceptación |
| `01-context/` | Lenguaje ubicuo → descripción general |
