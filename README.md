# Preparación de datos de ventas en Power BI — Documentación

Este documento explica cómo limpié y transformé el archivo `Ventas_export_legacy.xlsx` en Power Query, antes de armar cualquier visualización en Power BI.

## 1. Transformaciones realizadas y en qué orden

1. Cargué el archivo con el conector de Excel, entrando directo a Power Query con "Transformar datos" (no "Cargar"), para poder revisar todo antes de traerlo al modelo final.
2. Renombré las 20 columnas, de nombres técnicos del sistema legacy (`COD_OP`, `NOM_CLI`, `F_VTA`...) a nombres en `snake_case` legibles (`id_operacion`, `nombre_cliente`, `fecha_venta`...).
3. Corregí el tipo de `fecha_venta` de Fecha/Hora a Fecha.
4. Corregí el tipo de `fecha_alta_cliente` a Fecha, usando configuración regional española (Argentina) en vez de dejar que Power Query lo detecte solo.
5. Cambié `id_producto` de Número entero a Texto.
6. Quité 48 filas duplicadas (mismo `id_operacion` repetido), usando esa columna como referencia porque nunca debería repetirse un código de operación.
7. Filtré las filas con `id_operacion` vacío. Esto terminó de sacar las filas completamente en blanco del archivo — aunque en teoría eran 4, el paso de "Quitar duplicados" ya había tratado 3 de ellas como "duplicadas entre sí" (porque todos sus valores, incluido el `null`, eran iguales), así que acá solo quedaba 1 por sacar. Total final: 900 filas (de las 949 originales).
8. Reemplacé los nulos de `email_cliente` y `telefono_cliente` por el texto "No disponible".
9. Reemplacé los nulos de `descuento_pct` por `0`.
10. Armé una columna nueva, `total_venta_calculado`, con la fórmula `cantidad × precio_unitario × (1 − descuento_pct)`. Borré la columna `total_venta` original y renombré la nueva a `total_venta`.
11. Uní mayúsculas y minúsculas distintas en `canal_venta` (tenía 7 variantes de solo 3 canales reales) con la transformación "Poner en mayúscula cada palabra".
12. Separé todo en dos tablas usando "Referencia" (no "Duplicar", para no repetir el trabajo de limpieza dos veces): `Clientes` y `Ventas`.

## 2. Tipos de datos: qué elegí y por qué

**`fecha_venta`**: la pasé de Fecha/Hora a Fecha porque las 949 filas tenían la hora en `00:00:00` — no había ningún dato real ahí. Dejarla como Fecha/Hora solo agrega peso innecesario a la columna y puede complicar los filtros de fecha en el dashboard (por ejemplo, agrupar por mes es más directo con una fecha simple que con una fecha que además arrastra una hora).

**`fecha_alta_cliente`**: la reescribí a propósito usando "configuración regional: español (Argentina)", en vez de confiar en la detección automática que hace Power Query al importar. El motivo es que los datos vienen en formato día/mes/año, y sin especificar el idioma, Power Query podría interpretar mal una fecha ambigua (por ejemplo, `2/5/2023` podría leerse como 2 de mayo o como 5 de febrero según qué configuración regional use por default). Forzarlo evita ese riesgo silencioso.

**`id_producto`**: lo cambié de Número entero a Texto porque es un código identificador, no una cantidad que se suma o promedia. Si se queda como número, alguien podría arrastrarlo sin querer a una medida en Power BI y que se sume automáticamente (por ejemplo, el producto 50 + el producto 25 daría "75", un número sin ningún significado real).

## 3. Valores nulos y duplicados: cómo los resolví

**Duplicados exactos (48 filas)**: los eliminé con "Quitar duplicados" sobre `id_operacion`, porque cada código de operación identifica una venta única y nunca debería repetirse.

**Filas completamente vacías**: las saqué filtrando `id_operacion` distinto de nulo, ya que una fila sin ni siquiera el ID de operación es basura del export, no un dato real.

**`email_cliente` y `telefono_cliente`** (10% y 17% vacíos): reemplacé los nulos por "No disponible" en vez de borrar esas filas. Son datos de contacto opcionales, y la venta en sí sigue siendo información real y válida — borrar la fila entera por falta de un mail hubiera significado perder ventas reales por un dato secundario.

**`descuento_pct`** (5% vacío): reemplacé los nulos por `0`, porque matemáticamente representa "no hubo descuento aplicado" sin inventar ningún valor. Además, si lo hubiera dejado en `null`, cualquier cálculo que multiplique esta columna (como el total de la venta) habría dado `null` también, arrastrando el problema.

**`total_venta`**: acá encontré algo más que un problema de nulos. Comparé algunos valores del archivo original contra el cálculo `cantidad × precio_unitario × (1 − descuento_pct)` (por ejemplo, la fila 1: `2 × 45,91 × 1 = 91,82`, pero el archivo tenía cargado `87,23`) y no coincidían — es decir, la columna original no era confiable ni siquiera en las filas donde sí tenía dato. Por eso no "rellené" los nulos de la columna vieja: armé una columna nueva calculada desde cero con la fórmula, borré la original, y dejé solo la calculada con el nombre `total_venta`. Así se resolvieron a la vez los nulos y los valores mal calculados.

## 4. Criterio de separación: clientes vs. transacción

Separé las columnas según si describen **a la persona** (no cambian según la venta) o **a la venta puntual** (son propias de esa transacción):

- **Tabla `Clientes`** (9 columnas): `id_cliente`, `nombre_cliente`, `email_cliente`, `telefono_cliente`, `ciudad_cliente`, `provincia_cliente`, `segmento_cliente`, `cliente_activo`, `fecha_alta_cliente`.
- **Tabla `Ventas`** (11 columnas): `id_operacion`, `id_cliente`, `fecha_venta`, `id_producto`, `nombre_producto`, `categoria_producto`, `cantidad`, `precio_unitario`, `descuento_pct`, `total_venta`, `moneda`, `canal_venta`.

Dejé `id_cliente` en las dos tablas a propósito: en `Clientes` identifica una sola vez a cada persona, y en `Ventas` se repite por cada compra que hizo — esa repetición es justo lo que permite conectar ambas tablas después en el modelo de Power BI, con la misma lógica que las `FOREIGN KEY` que usé en SQL para el proyecto RetailPro.

Usé "Referencia" en vez de "Duplicar" para crear las dos tablas nuevas a partir de la consulta ya limpia, así no tuve que repetir todos los pasos de limpieza dos veces.
