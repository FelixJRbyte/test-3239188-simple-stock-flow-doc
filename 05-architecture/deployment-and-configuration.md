# Despliegue y configuración

> **[derivado]** El modelo no contiene un documento de despliegue. Solo se recogen
> los datos que menciona. **No se introduce ninguna tecnología ni infraestructura
> adicional.**

## 1. Confirmado por el modelo

| Elemento | Dato | Fuente |
|---|---|---|
| Motor | PostgreSQL 16.14 | Encabezado, §10 |
| Base y esquema | Base `simple_stock_flow`, esquema `sales` | §0, §3 |
| Zona horaria del servidor | UTC; todas las marcas de tiempo son `timestamptz` | §3 |
| Infraestructura | Docker Compose, repositorio `simple-stock-flow-infra` (contenedor `simple-stock-flow-db-1`); **levanta el motor y no define el esquema** | §6.2, §10 |
| Propiedad del esquema | Solo las migraciones EF; ningún DDL en `db/init/` ni a mano (ADR-001) | §3.2 |
| `pg_trgm` | La instala la propia migración EF del índice de trigramas; es *trusted* en Postgres 16 | §6.2 |
| Historial de migraciones | `public."__EFMigrationsHistory"`, fuera del esquema `sales` | §3.2 |
| Semilla de categorías | Dentro de `InitialSchema`, con cinco UUID literales | §9.1 |
| Administrador inicial | Lo crea el arranque con credenciales de entorno; no se siembra desde SQL | §9.2, D-09, D-10 |
| Rol `admin` | Lo provisiona el despliegue desde el entorno (DP-04) | §11 (H-3) |
| Imágenes | Almacenamiento externo; la base solo guarda `image_key` | §2.2, D-08 |

## 2. Configuración que el sistema necesita (nombres sin definir)
El modelo implica estos datos de configuración, pero **no fija sus nombres ni su formato**:
- Parámetros de conexión a la base `simple_stock_flow`.
- Credenciales del administrador inicial (entorno).
- Parámetros del almacenamiento externo de imágenes.
- Parámetros del puerto de hash.

Ninguno debe versionarse en el repositorio (principio citado en §9.2: «una credencial no es un valor versionado»).

## 3. Lo que el modelo no respalda (no se documenta como decisión)
- Entornos (desarrollo, pruebas, producción) y su diferenciación.
- Copias de seguridad, recuperación, alta disponibilidad, disponibilidad objetivo (SLA).
- Tecnología concreta de almacenamiento de imágenes, de tokens o de hash.
- Plataforma de alojamiento y gestión de secretos.
- Procedimiento de despliegue de migraciones en producción.

NFR-12 y el modelo mencionan que el mantenimiento automático de la base afecta al recorrido solo-índice (§6.2): es una **mitigación operativa** que el modelo no concreta.

## 4. Propuestas pendientes de aprobación
- **P-D1** Documentar los nombres de las variables de entorno en el repositorio de infraestructura, no aquí.
- **P-D2** Definir un procedimiento de verificación posterior a cada despliegue, basado en las consultas de §10 (columnas, restricciones, índices, extensiones). Ver TS-19 en [testing-strategy.md](../04-requirements/testing-strategy.md).
- **P-D3** Decidir la política de copias de seguridad antes de la primera venta real: el modelo advierte que varias migraciones son «gratis hoy y caras mañana» (§10.4).
