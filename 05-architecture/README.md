# 05 — Arquitectura

> **Qué es esto:** el estilo y las piezas del sistema que el
> modelo implica: agregados, puertos, adaptadores y **dónde
> vive cada regla**. Revisado tras la auditoría técnica.

| Archivo | Contenido | Estado |
|---|---|---|
| `overview.md` ⭐ | Estilo (hexagonal), capas, componentes, agregados | ✅ [derivado] |
| `ports-and-adapters.md` ⭐ | Puertos del dominio y sus adaptadores | ✅ [derivado] |
| `where-rules-live.md` ⭐ | Motor vs. dominio vs. pendiente vs. en conflicto: cada regla con un estado único | ✅ |
| `security-and-authorization.md` | Autenticación y autorización: confirmado frente a propuesto | ✅ [derivado]; propuestas sin aprobar |
| `errors-and-concurrency.md` | Clasificación de errores, transacción multi-agregado, `xmin`, reintentos | ✅ [derivado]; propuestas sin aprobar |
| `deployment-and-configuration.md` | Despliegue y configuración respaldados por el modelo | ✅ [derivado]; límites explícitos |
| `id-index.md` | Significado reconstruido de D, DP, T, A, H, FK, Q | ✅ [reconstruido] |
| `model-discrepancies.md` | Contradicciones del modelo (DISC-01 a DISC-10) y su estado | ✅ |
| `decisions/records/` | ADR-001…ADR-004 reconstruidos de las citas del modelo | ✅ [reconstruido] |
| `closure.md` ⭐ | **Cierre del reto**, revisado: distingue declarada, implementada y verificada | ✅ con pendientes explícitos |

## Evidencia del estilo en el modelo

El modelo **no dice «hexagonal»**, pero lo implica de forma
consistente (por tanto, **[derivado]**):

- Las invariantes las garantiza **C#** (§2) → capa de dominio.
- «El hash lo produce un **puerto**» (§2.5, D-09) → puerto
  de infraestructura.
- El reporte «se calcula en el **motor** por un **puerto de
  lectura**» (§1, D-06) → puerto de lectura con SQL en el motor.
- «Derivados de los **puertos**» (§6.1) → los patrones de acceso
  nacen de la interfaz del dominio.
- «Traducir tabla↔colección es responsabilidad del **adaptador
  de persistencia**» (§0) → adaptador EF.
- El esquema lo poseen las migraciones (§3.2, ADR-001) → la
  infraestructura define el DDL, no la aplicación.
