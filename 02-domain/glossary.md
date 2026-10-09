# Glosario del dominio — Simple Stock Flow

> Fuente: `spec/data-model.md` §1. En lenguaje de negocio; código y columnas
> en inglés (artículo XI, citado en §1).

| Término (negocio) | Definición funcional | Dónde vive (técnico) |
|---|---|---|
| **Producto** | Artículo del catálogo. Tiene **nombre, precio, stock, categoría e imagen opcional, y nada más** (DP-03) | `Product` · tabla `product` |
| **Categoría** | Clasificación de un producto. Conjunto **fijo de cinco**, sembrado, sin mantenimiento (D-10) | `Category` · tabla `category` |
| **Precio** | Valor monetario vigente del producto en el catálogo. Estrictamente positivo | `Money` (objeto de valor) · columna `product.price` |
| **Stock** | Unidades disponibles del producto. Nunca negativo | `product.stock` |
| **Imagen del producto** | **Clave opaca** del binario en almacenamiento externo. Ni el binario ni una ruta (D-08). Ausente = `NULL`, nunca cadena vacía | `product.image_key` |
| **Venta** | Hecho comercial consumado e **inmutable**: quién, cuándo y qué. Una vez registrada no se edita ni se borra | `Sale` · tabla `sale` |
| **Línea de venta** | Renglón de la venta: producto, cantidad y **precio congelado** del momento. No existe fuera de su venta | `SaleItem` · tabla `sale_item` |
| **Cantidad** | Unidades vendidas en una línea. Estrictamente positiva | `Quantity` (objeto de valor) · `sale_item.quantity` |
| **Total de la venta** | Suma de subtotales. **Se calcula, no se almacena** | `Sale.Total` · **sin columna** |
| **Subtotal de la línea** | Precio unitario por cantidad. **Se calcula, no se almacena** | `SaleItem.Subtotal` · **sin columna** |
| **Usuario** | Operador interno que se autentica y registra ventas. **No hay entidad cliente ni comprador** | `User` · tabla `user` |
| **Rol** | Atribución del usuario en un conjunto cerrado de dos: `admin` o `seller` | `user.role` |
| **Hash de clave** | Huella irreversible de la contraseña. El dominio **nunca ve la clave en claro** (D-09) | `user.password_hash` |
| **Rango de fechas** | Ventana temporal del reporte. El fin no puede ser anterior al inicio | Objeto de valor de la capa de aplicación · **sin tabla** |
| **Reporte de ventas** | Agregación por producto sobre un rango. **No se persiste**: se calcula en el motor por un puerto de lectura (D-06) | Modelo de lectura · **sin tabla** |

## Dos matices del modelo que no se pueden perder

**«Congelado» no es desnormalización** (§1): la línea guarda una **copia del
valor en el instante de la venta** y esa copia no sigue al catálogo. El precio
de venta y el nombre vendido son **hechos propios de la venta**. Es lo que permite
renombrar o reprecificar un producto sin reescribir reportes de períodos cerrados.

**Correcciones explícitas del modelo** (§1): el sistema es **monomoneda por
construcción** (D-05) — no existe «moneda de la venta» — y DP-03 **cierra** la
definición de Producto: sin descripción, sin SKU, sin código de referencia.
