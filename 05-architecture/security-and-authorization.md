# Autenticación y autorización

> **[derivado]** El modelo no contiene un documento de seguridad ni el contrato de
> la API (`api-contract.md` queda fuera, §12). Este documento reúne lo que el modelo
> **sí** dice y separa lo confirmado de lo que es **propuesta pendiente de aprobación**.
> **No se fija ninguna tecnología** (formato de token, algoritmo de hash, proveedor).

## 1. Decisiones confirmadas por el modelo

| # | Decisión | Fuente |
|---|---|---|
| S-1 | Los usuarios son **operadores internos**; no hay entidad cliente | §1, §2.5 |
| S-2 | Roles: conjunto cerrado `admin` / `seller` | §2.5 |
| S-3 | El hash de clave lo produce y verifica el puerto `IPasswordHashPort`; el dominio nunca ve la clave en claro | §2.5, D-09 |
| S-4 | `password_hash` jamás en logs, respuestas, proyecciones ni mensajes de error; nunca se indexa | §7 |
| S-5 | `username` es dato personal de acceso restringido: no va en respuestas anónimas ni endpoints públicos | §7 |
| S-6 | `role` es confidencial interno | §7 |
| S-7 | El administrador inicial lo crea el arranque con credenciales de entorno; no se siembra desde SQL | §9.2 |
| S-8 | **DP-04:** nadie otorga el rol `admin` en ejecución; un administrador da de alta vendedores y el despliegue provisiona `admin` desde el entorno | §11 (H-3) |
| S-9 | **A-1 (alta anónima) está cerrado:** sin token → 401, con `seller` → 403 | §13 D-3 |
| S-10 | El inicio de sesión busca por nombre con igualdad exacta (Q10) | §6.1 |

> **Estado de S-9:** §13 D-3 lo da por medido el 2026-09-20. **Este repositorio no lo verificó** (no hay API ni pruebas). §9.2 aún lo describe como roto (DISC-08).

## 2. Lo que el modelo no define
- Mecanismo concreto de autenticación (formato de credencial, caducidad, renovación).
- Quién puede ejecutar cada operación distinta del alta de usuarios.
- Política de bloqueo ante intentos fallidos, rotación de claves o recuperación de acceso.
- Algoritmo de hash (se elige en el adaptador del puerto).

## 3. Matriz de permisos

| Operación | `anónimo` | `seller` | `admin` | Estado |
|---|---|---|---|---|
| Alta de usuario `seller` | denegado (401) | denegado (403) | permitido | **Confirmado** (§13 D-3 y DP-04), no verificado aquí |
| Alta de usuario `admin` | denegado | denegado | denegado | **Confirmado** (DP-04); código y cuerpo de error no definidos |
| Registrar producto, ajustar stock, imagen, baja lógica | — | permitido | permitido | **[supuesto]** inferido de los roles descritos en el README del reto y del modelo; no figura en el modelo |
| Registrar y consultar ventas | — | permitido | permitido | **[supuesto]** |
| Ver ventas por rango y reporte agregado | — | por decidir | permitido | **[supuesto]**; el modelo no dice quién ve los reportes |
| Listar categorías | — | permitido | permitido | **[supuesto]** |

Toda celda `[supuesto]` requiere aprobación humana.

## 4. Propuestas pendientes de aprobación
- **P-S1** La autorización se evalúa en el borde de aplicación (API), no en el dominio, porque §9.2 sitúa la política de otorgar `admin` en la API.
- **P-S2** Toda operación distinta del inicio de sesión exige identidad autenticada.
- **P-S3** Los mensajes de error de autenticación no distinguen entre usuario inexistente y clave incorrecta, para no revelar qué usuarios existen (coherente con S-5).
- **P-S4** Los eventos o logs que incluyan `username` o `sold_by` deben tratarse como dato personal (§7, [domain-events.md](../02-domain/domain-events.md)).

## 5. Riesgos conocidos
- `role` y `username` solo se normalizan y validan en el dominio (§4, T-20). Un `INSERT` manual puede crear un rol inválido (DISC-05).
- Si el administrador inicial se provisiona con credenciales de entorno, su gestión (rotación, almacenamiento seguro) depende del despliegue y no está definida ([deployment-and-configuration.md](deployment-and-configuration.md)).
