# Historias de usuario — Simple Stock Flow

> Cada historia cita su origen en `spec/data-model.md`.
> Roles posibles (§2.5): **Vendedor** (`seller`) y **Administrador**
> (`admin`). **No hay comprador**: el usuario del sistema es siempre
> un operador interno (§1).
>
> **Revisión posterior a la auditoría.** Se conservan los criterios originales y se
> añaden **escenarios de aceptación** (Dado / Cuando / Entonces). Los escenarios son
> **documentales**: ninguno se ha ejecutado (ver
> [testing-strategy.md](testing-strategy.md)). Lo que el modelo no define va marcado
> **[supuesto]** o **[pendiente de confirmación]**. Los estados de T-xx, A-x y H-x
> siguen [id-index.md](../05-architecture/id-index.md); las contradicciones del modelo,
> [model-discrepancies.md](../05-architecture/model-discrepancies.md).

## US-01 — Registrar un producto en el catálogo

**Rol:** Vendedor. **Origen:** §2.2 (`Product`), §3.

Como vendedor, quiero registrar un producto con **nombre, precio,
stock y categoría**, para tenerlo disponible para la venta.

**Criterios de aceptación (trazados):**

- Nombre obligatorio, no vacío, guardado recortado (§2.2, `Product.Rename`).
- Precio estrictamente positivo (§2.2, `Product.ChangePrice`). Hoy es **solo
  dominio**: un precio 0 insertado por fuera del adaptador pasa, salvo que el
  `CHECK` de T-20 exista (**estado sin confirmar**, DISC-05).
- Stock inicial ≥ 0 (§2.2; barrera motor `ck_product_stock_non_negative`).
- Categoría **obligatoria y existente** (§2.2, FK-1); elegida de las
  **cinco fijas** (§9.1).
- Imagen **opcional**: `image_key` es clave opaca del binario en
  almacenamiento externo; ausente = `NULL`, nunca `""` (§2.2, D-08).
- **Nada más**: sin descripción, sin SKU, sin código de referencia (DP-03).

**Escenarios de aceptación**

- **Dado** un vendedor autenticado y una categoría sembrada, **cuando** registra un producto con nombre `"  Martillo  "`, precio 25,50 y stock 10, **entonces** el producto queda con nombre `Martillo` (recortado) y stock 10. *(§2.2)*
- **Dado** un vendedor autenticado, **cuando** intenta registrar un producto con nombre vacío o precio 0, **entonces** la operación se rechaza. *(§2.2; el precio 0 depende del dominio, DISC-05)*
- **Dado** un vendedor autenticado, **cuando** registra un producto con una categoría que no existe, **entonces** se rechaza. *(FK-1)*
- **Dado** un producto sin imagen, **cuando** se guarda, **entonces** `image_key` es `NULL`, nunca cadena vacía. *(§2.2)*

## US-02 — Mantener un producto

**Rol:** Vendedor. **Origen:** §2.2.

Como vendedor, quiero renombrar un producto, cambiar su precio,
retirar y reponer stock, para mantener el catálogo al día.

**Criterios:**

- `Product.Rename` / `Product.ChangePrice` / `Product.Withdraw` /
  `Product.Restock` (§2.2).
- **Retirar más stock del disponible falla** — regla de proceso, solo
  dominio (§2.2).
- Tras cualquier operación, `stock >= 0` — garantizada **en el motor**
  (§2.2, ADR-002).
- Cambiar precio **no** altera ventas ya registradas: el precio
  congelado en la línea no sigue al catálogo (§1).

**Escenarios de aceptación**

- **Dado** un producto con stock 5, **cuando** se retiran 3, **entonces** el stock es 2. *(§2.2)*
- **Dado** un producto con stock 5, **cuando** se intenta retirar 6, **entonces** la operación falla y el stock sigue en 5. *(§2.2)*
- **Dado** un producto con stock 5, **cuando** alguien intenta dejar el stock en −1 por un `UPDATE` directo, **entonces** el motor lo rechaza (`ck_product_stock_non_negative`). *(§4; verificada en §10.2)*
- **Dado** un producto vendido a 10,00, **cuando** se cambia su precio a 12,00, **entonces** la línea de venta existente sigue mostrando 10,00. *(§1, §2.4)*
- **Dado** un producto con stock 1 leído por dos operaciones a la vez, **cuando** ambas intentan retirar 1, **entonces** el stock final no es negativo y **solo una** operación se confirma. El tipo de error de la otra y su posible reintento son **[pendiente de confirmación]**. *(ADR-002; [errors-and-concurrency.md](../05-architecture/errors-and-concurrency.md))*

## US-03 — Adjuntar o reemplazar la imagen de un producto

**Rol:** Vendedor. **Origen:** §2.2, §7.1, D-08.

Como vendedor, quiero adjuntar una imagen a un producto (o cambiarla),
para identificarlo visualmente.

**Criterios:**

- Se guarda solo `image_key` (clave opaca): **ni binario ni ruta**
  en la base (§2.2, D-08).
- Al reemplazar o dar de baja, **el binario sí se elimina** — el único
  dato del sistema que se borra físicamente (§7.1).
- **Orden obligatorio:** primero anular `image_key` y confirmar;
  **después** borrar el binario. Un binario huérfano es inofensivo;
  una clave a binario borrado es imagen rota permanente (§7.1).
- **No se promete atomicidad** con la base (§7.1).

**Escenarios de aceptación**

- **Dado** un producto con imagen, **cuando** se reemplaza, **entonces** primero se confirma la nueva `image_key` y después se borra el binario anterior. *(§7.1)*
- **Dado** que el borrado del binario falla tras confirmar la base, **cuando** termina la operación, **entonces** queda un binario huérfano y ninguna clave rota. La limpieza del huérfano es H-2 (abierto). *(§7.1, §11)*

## US-04 — Buscar y consultar productos

**Rol:** Vendedor. **Origen:** §6.1 (Q1, Q2, Q3).

Como vendedor, quiero buscar productos por texto parcial y categoría,
y ver los activos, para usarlos al registrar una venta.

**Criterios:**

- Q1: texto parcial, categoría, **solo activos**, ordenado por nombre,
  paginado (§6.1).
- Q2: producto por identificador (§6.1).
- Q3: productos por lote de identificadores, **activos** — es el punto
  de contención de concurrencia D-04 (§6.1).
- Los dados de baja **no aparecen**: filtro global sobre `deleted_at`
  (§2.2, ADR-003; T-09 saldada según §13 D-1).

**Escenarios de aceptación**

- **Dado** productos activos y uno dado de baja, **cuando** se busca por texto y categoría, **entonces** solo aparecen los activos, ordenados por nombre y paginados. *(Q1, §2.2)*
- **Dado** un lote de identificadores que incluye un producto dado de baja, **cuando** se consulta, **entonces** ese producto no se devuelve. *(Q3)*

## US-05 — Registrar una venta

**Rol:** Vendedor. **Origen:** §2.3.

Como vendedor, quiero registrar una venta con al menos una línea,
para dejar constancia del hecho comercial.

**Criterios:**

- La venta registra **quién** la realiza: obligatorio y no vacío (§2.3).
- **Al menos una línea** para poder confirmarse (§2.3, `Sale.EnsureConfirmable`).
- **Un producto no se repite** dentro de la misma venta (§2.3, `Sale.AddItem`;
  índice único del motor implementado según §13 D-2).
- **Descontar stock y añadir la línea son una sola operación**:
  `Sale.AddItem` llama a `Product.Withdraw` antes de añadir (§2.3).
- Cada línea congela **nombre, precio y categoría** del instante (§2.4). El
  congelado de la categoría depende de T-11 (**estado en conflicto**, DISC-03).
- La venta queda **inmutable**: no existe puerto de edición ni de
  borrado (§2.3).

**Escenarios de aceptación**

- **Dado** un vendedor autenticado y un producto con stock 10, **cuando** registra una venta de 4 unidades, **entonces** la venta queda registrada con una línea de nombre y precio congelados y el stock es 6. *(§2.3, §2.4)*
- **Dado** un vendedor autenticado, **cuando** intenta registrar una venta sin líneas, **entonces** se rechaza. *(`EnsureConfirmable`)*
- **Dado** una venta con una línea del producto P, **cuando** se intenta añadir otra línea de P, **entonces** se rechaza. *(§2.3)*
- **Dado** un producto con stock 3, **cuando** se intenta vender 5, **entonces** la venta se rechaza y el stock sigue en 3. *(§2.2)*
- **Dado** una venta de dos líneas donde la segunda no tiene stock suficiente, **cuando** se intenta registrar, **entonces** no queda ninguna línea registrada ni stock descontado. **[Propuesta pendiente de aprobación]**: P-T1 en [errors-and-concurrency.md](../05-architecture/errors-and-concurrency.md); el modelo solo afirma que cada descuento y su línea son una operación.
- **Dado** una venta ya registrada, **cuando** alguien busca editarla o borrarla, **entonces** no existe esa operación. *(§2.3)*

## US-06 — Consultar una venta con sus líneas

**Rol:** Vendedor / Administrador. **Origen:** §6.1 (Q6).

Como operador, quiero ver una venta completa con sus líneas, para
verificar qué se vendió y a qué precio.

**Criterios:**

- Q6: venta con sus líneas, por clave y reunión (§6.1).
- El **total se calcula** sumando subtotales; **no se almacena** (§1).
- Cada línea muestra el precio y nombre **congelados** (§2.4).

**Escenarios de aceptación**

- **Dado** una venta con líneas de 2 × 10,00 y 1 × 5,00, **cuando** se consulta, **entonces** el total mostrado es 25,00 y no existe una columna que lo almacene. *(§1)*

## US-07 — Ver ventas por rango de fechas y el reporte agregado

**Rol:** Administrador **[supuesto]**: el modelo no dice quién consulta los reportes (ver
[security-and-authorization.md](../05-architecture/security-and-authorization.md)).
**Origen:** §6.1 (Q7, Q9), §1, DP-02, §11.1 (H-1).

Como administrador, quiero ver las ventas de un período y el total
vendido por producto, para hacer seguimiento del negocio.

**Criterios:**

- Q7: ventas por rango de fecha, fecha descendente, paginado (§6.1).
- Q9: **reporte agregado por producto** sobre un rango, ordenado por
  importe descendente (§6.1).
- **H-1 (cerrado el 2026-09-20):** el reporte **agrupa por el valor congelado**
  `product_id, product_name, category_name`. Tras una recategorización dentro del
  rango puede haber **más de una fila por producto** (§11.1).
- **Tensión con CA-06.1** de `spec.md` («una fila por producto», según §11.1):
  `spec.md` no se entrega y **no se reproduce su texto**. Reescribirlo es una
  **decisión pendiente del propietario** (DISC-09).
- El reporte **se calcula en el motor** por un puerto de lectura;
  **no se persiste** (§1, D-06).
- El reporte **no se desglosa por vendedor** (DP-02, §7.1).
- El rango no puede tener fin anterior al inicio (§1, objeto de valor).
- Q8 (ventas por rango **sin paginar**) **no tiene consumidor** si el
  reporte agrega en el motor: conviene retirarlo del puerto (§6.1).
- Defecto abierto A-7 en el nombre de producto del reporte (§11.1, DISC-10).

**Escenarios de aceptación**

- **Dado** ventas de septiembre y octubre del producto P, **cuando** se pide el reporte del rango completo, **entonces** se agrupa por `product_id`, `product_name` y `category_name` congelados. *(H-1)*
- **Dado** que P se vendió como «Herramientas» en septiembre y «Ferretería» en octubre, **cuando** se pide el reporte de ambos meses, **entonces** aparecen **dos filas** de P, una por etiqueta. *(§11.1)*
- **Dado** un reporte ya leído de septiembre, **cuando** se registra una venta nueva con otra etiqueta en octubre, **entonces** el reporte de septiembre no cambia. *(criterio «un reporte cerrado no cambia nunca», §11.1)*
- **Dado** un rango con fin anterior al inicio, **cuando** se solicita, **entonces** se rechaza. *(§1)*
- **Dado** cualquier reporte, **cuando** se consulta, **entonces** no incluye ningún desglose por vendedor. *(DP-02)*

## US-08 — Gestionar usuarios (alta de vendedores)

**Rol:** Administrador. **Origen:** §2.5, §9.2, §11 (H-3), §13 D-3, DP-04.

Como administrador, quiero dar de alta usuarios operadores con rol `seller`,
para controlar quién registra ventas.

> **Cambio respecto a la versión anterior.** Antes decía que el administrador crea
> usuarios con rol `admin` o `seller`. **DP-04 establece que nadie otorga `admin` en
> ejecución**: el rol `admin` lo provisiona el despliegue desde el entorno
> (§11 H-3; §9.2: el administrador inicial lo crea el arranque).

**Criterios:**

- Nombre de usuario obligatorio, **único**, guardado en **minúsculas y
  recortado** (§2.5, `User.NormalizeUsername`).
- Hash de clave obligatorio y no vacío; **el dominio nunca ve la clave
  en claro** — el hash lo produce un puerto (§2.5, D-09).
- Rol dentro del conjunto cerrado `('admin','seller')` (§2.5, `Roles.IsValid`); desde
  la aplicación **solo se asigna `seller`** (DP-04).
- **A-1 está cerrado según §13 D-3** (2026-09-20): sin token → 401, con `seller`
  → 403. No verificado en este repositorio; §9.2 aún lo describe como roto
  (DISC-08).
- Un usuario con ventas **no se elimina**: FK-4 `RESTRICT` (§5). FK-4 está
  **pendiente (T-12)**.

**Escenarios de aceptación**

- **Dado** un administrador autenticado, **cuando** da de alta a `"Ana "` con rol `seller`, **entonces** se crea el usuario `ana` (minúsculas, recortado) con el hash producido por el puerto. *(§2.5)*
- **Dado** la existencia de `ana`, **cuando** se intenta dar de alta `"Ana "`, **entonces** se rechaza por nombre duplicado. *(§2.5)*
- **Dado** una petición sin credenciales, **cuando** intenta dar de alta un usuario, **entonces** recibe 401. *(§13 D-3; reportado, no verificado)*
- **Dado** un usuario `seller` autenticado, **cuando** intenta dar de alta un usuario, **entonces** recibe 403. *(§13 D-3; reportado, no verificado)*
- **Dado** un administrador autenticado, **cuando** intenta dar de alta un usuario con rol `admin`, **entonces** no se otorga el rol. *(DP-04)* El código y el cuerpo exactos del rechazo son **[pendiente de confirmación]**.

## US-09 — Iniciar sesión

**Rol:** Vendedor / Administrador. **Origen:** §2.5, §9.2, §6.1 (Q10).

Como operador, quiero iniciar sesión con mi nombre de usuario y clave,
para acceder al sistema.

**Criterios:**

- Q10: usuario por nombre, **igualdad exacta**, en cada inicio de
  sesión (§6.1) — sirve el índice único `IX_user_username` (§6.2).
- La verificación de clave pasa **por el puerto de hash**: el dominio
  nunca ve la clara (§2.5, D-09).
- El administrador inicial **no lo siembra la base**: lo crea el arranque
  de la aplicación con credenciales de entorno (§9.2).

**Escenarios de aceptación**

- **Dado** el usuario `ana` y su clave correcta, **cuando** inicia sesión, **entonces** accede; la verificación pasa por el puerto de hash. *(§2.5)*
- **Dado** una clave incorrecta, **cuando** inicia sesión, **entonces** se rechaza y ningún mensaje ni log contiene `password_hash`. *(§7)*
- **Dado** una base recién desplegada, **cuando** arranca la aplicación con credenciales de entorno, **entonces** existe exactamente un administrador inicial. *(§9.2)*

## US-10 — Listar categorías

**Rol:** Vendedor. **Origen:** §2.1, §6.1 (Q4, Q5), §9.1.

Como vendedor, quiero ver las cinco categorías al crear un producto,
para clasificarlo.

**Criterios:**

- Q4: listado ordenado por nombre; Q5: categoría por identificador (§6.1).
- **Solo lectura**: ningún puerto crea, renombra ni borra categorías (§2.1).
- Son datos semilla de la migración inicial, con identificadores
  literales (§9.1).

**Escenarios de aceptación**

- **Dado** una base migrada, **cuando** se listan las categorías, **entonces** aparecen cinco (General, Herramientas, Electricidad, Fontanería y Pinturas), ordenadas por nombre. *(§9.1)*
- **Dado** cualquier puerto del sistema, **cuando** se busca crear, renombrar o borrar una categoría, **entonces** esa operación no existe. *(§2.1)*

## US-11 — Dar de baja un producto (lógica)

**Rol:** Vendedor. **Origen:** §2.2, §7.1, ADR-003, T-09.

Como vendedor, quiero dar de baja un producto que ya no vendo, sin
borrar su historial de ventas.

**Criterios:**

- Baja **lógica** mediante `deleted_at`; **nunca borrado físico** (§2.2, §7.1).
  T-09 está saldada según §13 D-1.
- Desaparece de búsquedas (filtro global, §2.2) pero **sus líneas de
  venta permanecen intactas** y el reporte sigue contándolo (§7.1).
- Al dar de baja, **se elimina también el binario de imagen** (§7.1),
  siguiendo el orden de US-03.
- Un borrado físico que intente un `psql` debe **fallar ruidosamente**
  por FK-3 (§5, ADR-003). FK-3 está **implementada según §13 D-2** y **no
  verificada** (DISC-04).

**Escenarios de aceptación**

- **Dado** un producto con ventas, **cuando** se da de baja, **entonces** deja de aparecer en las búsquedas y sus líneas de venta siguen intactas. *(§7.1)*
- **Dado** un producto con ventas, **cuando** alguien intenta borrar su fila con `DELETE`, **entonces** el motor lo rechaza por FK-3. *(según §13 D-2; no verificado)*
- **Dado** un producto de baja, **cuando** se consulta el reporte, **entonces** sigue contando sus ventas. *(§7.1)*

## Historias explícitamente rechazadas (y por qué)

| Historia no escrita | Razón del modelo |
|---|---|
| «Editar una venta» | No existe la operación (§2.3) |
| «Crear una categoría» | D-10: cinco fijas, sin mantenimiento (§2.1) |
| «Registrar un cliente» | No existe la entidad (§1, §7) |
| «Vender en otra moneda» | D-05: monomoneda por construcción (§3) |
| «Ver ventas por vendedor» | DP-02 lo prohíbe (§6.3, §7.1) |
| «Ver quién cambió un precio y cuándo» | §8: si aparece ese requisito real, vuelve al propietario y se resuelve con **bitácora**, no con columnas |
| «Un administrador otorga el rol admin» | DP-04: nadie lo otorga en ejecución (§11, H-3) |
