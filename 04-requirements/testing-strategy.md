# Estrategia de pruebas (documental)

> **Esta estrategia describe pruebas; NO las ejecuta.** No existe código ni motor de
> base de datos en este repositorio. **Ningún escenario de aquí se ha ejecutado** y
> ninguno puede citarse como evidencia de que el sistema funcione. Todos los estados
> son *«No ejecutada»*.

## 1. Alcance y niveles

| Nivel | Qué comprueba | Qué necesita | Disponible aquí |
|---|---|---|---|
| **Dominio** | Invariantes de `Product`, `Sale`, `User` y objetos de valor | Código C# | No |
| **Persistencia** | Restricciones, índices y FK en el motor | PostgreSQL 16 con las migraciones aplicadas | No |
| **Aplicación / API** | Casos de uso, autorización, errores | API en ejecución | No |
| **Verificación del modelo** | Que el documento no mienta (consultas de §10) | Acceso `psql` al motor | No |
| **Rendimiento** | Planes de ejecución de Q1 a Q10 | Motor con datos de volumen conocido | No |

## 2. Convenciones
- **Estado** de toda prueba: *No ejecutada*. Quien la ejecute registra fecha, entorno y resultado aparte.
- Un resultado esperado marcado **[pendiente de confirmación]** o **[propuesta]** no puede darse por correcto o incorrecto hasta que el propietario decida.
- Las pruebas que dependen de un estado **«según §13»** o **«en conflicto»** no deben ejecutarse como si ese estado estuviera verificado.

## 3. Escenarios

| ID | Escenario | Precondiciones | Pasos | Resultado esperado | Trazabilidad | Estado |
|---|---|---|---|---|---|---|
| TS-01 | Retirar más stock del disponible | Producto con stock 5 | Retirar 6 | La operación falla; stock sigue en 5 | US-02, §2.2 | No ejecutada |
| TS-02 | Barrera de stock en el motor | Entorno de pruebas con acceso `psql`; producto con stock 5 | `UPDATE` directo a stock −1 | El motor rechaza (`ck_product_stock_non_negative`) | US-02, NFR-01, NFR-02, §4 | No ejecutada |
| TS-03 | Registro de venta con descuento | Producto con stock 10; vendedor autenticado | Vender 4 | Venta con línea congelada; stock 6 | US-05, §2.3, §2.4 | No ejecutada |
| TS-04 | Venta sin líneas | Vendedor autenticado | Confirmar venta vacía | Rechazada | US-05, `EnsureConfirmable` | No ejecutada |
| TS-05 | Producto repetido en la venta | Venta con línea de P | Añadir otra línea de P | Rechazada por dominio; el índice único del motor es **según §13 D-2** | US-05, §2.3 | No ejecutada |
| TS-06 | Venta multilínea atómica | Dos productos; el segundo sin stock suficiente | Registrar la venta de ambos | No queda ninguna línea ni descuento | US-05 · **[propuesta]** P-T1 | No ejecutada |
| TS-07 | Conflicto de concurrencia con `xmin` | Producto con stock 1; dos operaciones simultáneas | Ambas retiran 1 | Stock final ≥ 0; solo una se confirma. Error del perdedor y reintento: **[pendiente de confirmación]** | US-02, NFR-10, NFR-11, ADR-002 | No ejecutada |
| TS-08 | Precio congelado | Venta registrada a 10,00 | Cambiar el precio a 12,00; consultar la venta | La línea sigue en 10,00 | US-02, US-06, §1, §2.4 | No ejecutada |
| TS-09 | Reporte agrupado por categoría congelada (H-1) | Ventas de P como «Herramientas» (sept.) y «Ferretería» (oct.) | Reporte del rango completo | Dos filas de P | US-07, H-1, §11.1 | No ejecutada |
| TS-10 | Reporte cerrado estable | Reporte de septiembre leído | Registrar venta nueva en octubre; releer septiembre | Sin cambios | US-07, §11.1 | No ejecutada |
| TS-11 | Rango inválido | — | Reporte con fin anterior al inicio | Rechazado | US-07, §1 | No ejecutada |
| TS-12 | Sin desglose por vendedor | Ventas de varios vendedores | Reporte | Sin campo ni agrupación por vendedor | US-07, DP-02 | No ejecutada |
| TS-13 | Alta de usuarios: autenticación y rol | API en ejecución | Alta sin token; con `seller`; con `admin` | 401; 403; permitido (solo `seller`) | US-08, NFR-20, §13 D-3, DP-04 | No ejecutada |
| TS-14 | `admin` no se otorga en ejecución | Administrador autenticado | Alta con rol `admin` | No se otorga. Código y cuerpo: **[pendiente de confirmación]** | US-08, DP-04 | No ejecutada |
| TS-15 | Normalización de usuario | Usuario `ana` existente | Alta de `"Ana "` | Rechazada por duplicado | US-08, §2.5 | No ejecutada |
| TS-16 | Login sin filtrar secretos | Usuario existente | Login correcto e incorrecto; revisar respuestas y logs | Nunca aparece `password_hash` | US-09, NFR-16, §7 | No ejecutada |
| TS-17 | Baja lógica | Producto con ventas | Dar de baja; buscar; consultar reporte | No aparece en búsquedas; sus ventas siguen | US-11, US-04, §7.1 | No ejecutada |
| TS-18 | FK-3 en el motor | Entorno con acceso `psql`; producto con ventas | `DELETE` directo del producto | Falla por FK-3. **Solo ejecutable si FK-3 se verifica antes** | US-11, NFR-09, §13 D-2 | No ejecutada |
| TS-19 | Verificación del modelo contra el motor | Acceso `psql` | Ejecutar las consultas de §10.1 a §10.4 | Comparar con el modelo y registrar las diferencias. **No usar 21/8/12 como valores esperados** sin confirmar la cifra vigente (DISC-01, DISC-07) | NFR-26, §10 | No ejecutada |
| TS-20 | Plan de ejecución de Q9 | Motor con volumen de datos conocido | `EXPLAIN` de Q9 | **Objetivo** (NFR-13), no hecho comprobado: sin acceso a la tabla `sale_item` | NFR-12, NFR-13, §6.2 | No ejecutada |
| TS-21 | Imagen: orden de borrado | Producto con imagen | Reemplazar la imagen; forzar fallo del borrado | `image_key` confirmada antes del borrado; binario huérfano, sin clave rota | US-03, §7.1 | No ejecutada |
| TS-22 | Categorías de solo lectura y semilla | Base migrada | Listar; intentar mutar | Cinco categorías; mutación inexistente | US-10, §9.1 | No ejecutada |

## 4. Lo que no cubre esta estrategia
- Los cinco `CHECK` de T-20 (`price > 0`, `quantity > 0`, `category.name` no vacío, `role`, `username` en minúsculas): su estado no está confirmado (DISC-05). Si existen, deben añadirse escenarios que intenten violarlos por `psql`.
- Sold-by y FK-4 (T-12) y `pg_trgm` (T-13): pendientes; sus pruebas se definen cuando se implementen.
- Cobertura de código, datos de volumen y pruebas de carga: el modelo no fija metas (ver supuesto de NFR).
