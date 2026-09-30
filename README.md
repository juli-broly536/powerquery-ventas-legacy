# Preparación de datos de ventas en Power BI — Documentación

Este documento explica cómo limpié y transformé el archivo `Ventas_export_legacy.xlsx` en Power Query, antes de armar cualquier visualización en Power BI. El código completo de cada consulta (en lenguaje M) está disponible en los archivos `consulta_clientes.pq` y `consulta_ventas.pq` de este mismo repositorio.

## 1. Transformaciones realizadas y en qué orden

1. Cargué el archivo con el conector de Excel, entrando directo a Power Query con "Transformar datos" (no "Cargar"), para poder revisar todo antes de traerlo al modelo final.
2. Renombré las 20 columnas, de nombres técnicos del sistema legacy (`COD_OP`, `NOM_CLI`, `F_VTA`...) a nombres en `snake_case` legibles (`id_operacion`, `nombre_cliente`, `fecha_venta`...).
3. Corregí el tipo de `fecha_venta` de Fecha/Hora a Fecha.
4. Corregí el tipo de `fecha_alta_cliente` a Fecha, usando configuración regional española (Argentina) en vez de dejar que Power Query lo detecte solo.
<img width="1919" height="1079" alt="01-fecha-config-regional" src="https://github.com/user-attachments/assets/5bf65a11-0c83-4ce8-a3ff-1e7c9beb5b5e" />

   ![Fecha con configuración regional argentina](capturas/01-fecha-config-regional.png)

5. Cambié `id_producto` de Número entero a Texto.
6. Quité 48 filas duplicadas (mismo `id_operacion` repetido), usando esa columna como referencia porque nunca debería repetirse un código de operación.
<img width="1919" height="1079" alt="02-duplicados-quitados" src="https://github.com/user-attachments/assets/bad3b625-7d50-49a0-b122-309b37f134b3" />

   ![Duplicados eliminados - 901 filas](capturas/02-duplicados-quitados.png)

7. Filtré las filas con `id_operacion` vacío. Esto terminó de sacar las filas completamente en blanco del archivo — aunque en teoría eran 4, el paso de "Quitar duplicados" ya había tratado 3 de ellas como "duplicadas entre sí" (porque todos sus valores, incluido el `null`, eran iguales), así que acá solo quedaba 1 por sacar. Total final: 900 filas (de las 949 originales).
8. Reemplacé los nulos de `email_cliente` y `telefono_cliente` por el texto "No disponible".
<img width="368" height="790" alt="03-reemplazo-nulos" src="https://github.com/user-attachments/assets/20bd0b75-b419-4506-903c-acac2dc0b9db" />

   ![Reemplazo de nulos por "No disponible"](capturas/03-reemplazo-nulos.png)

9. Reemplacé los nulos de `descuento_pct` por `0`.
10. Armé una columna nueva, `total_venta_calculado`, con la fórmula `cantidad × precio_unitario × (1 − descuento_pct)`. Borré la columna `total_venta` original y renombré la nueva a `total_venta`.
<img width="840" height="582" alt="04-formula-total-venta" src="https://github.com/user-attachments/assets/bde31eb0-3ef7-48db-85a3-aa15b72e1ea0" />

    ![Fórmula de total_venta recalculada](capturas/04-formula-total-venta.png)

11. Separé todo en dos tablas usando "Duplicar" sobre la consulta `VENTAS_EXPORT` ya limpia: `Clientes` y `Ventas`. Como usé "Duplicar" en vez de "Referencia", cada una de las dos consultas vuelve a cargar y aplicar todos los pasos anteriores de forma independiente (no están encadenadas entre sí), pero el resultado final es el mismo dataset limpio dividido en dos.
12. En la tabla `Ventas`, unifiqué las variantes de mayúsculas/minúsculas de `canal_venta` (tenía 7 combinaciones de solo 3 canales reales: `Sucursal`/`SUCURSAL`, `online`/`ONLINE`/`Online`, `Telefonico`/`TELEFONICO`) con la transformación "Poner en mayúscula cada palabra".
<img width="261" height="779" alt="07-canal-venta-normalizado" src="https://github.com/user-attachments/assets/0e5836b7-a8f2-422c-aa10-8a41c5ee4a72" />

    ![Canal_venta normalizado a 3 valores](capturas/07-canal-venta-normalizado.png)

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

**`canal_venta`**: no tenía nulos, pero sí un problema equivalente: 7 formas distintas de escribir 3 canales reales (mayúsculas, minúsculas y capitalización mezcladas). Lo unifiqué aplicando "Poner en mayúscula cada palabra" sobre toda la columna, dejando solo `Sucursal`, `Online` y `Telefonico`.

## 4. Criterio de separación: clientes vs. transacción

Separé las columnas según si describen **a la persona** (no cambian según la venta) o **a la venta puntual** (son propias de esa transacción):

- **Tabla `Clientes`** (9 columnas): `id_cliente`, `nombre_cliente`, `email_cliente`, `telefono_cliente`, `ciudad_cliente`, `provincia_cliente`, `segmento_cliente`, `cliente_activo`, `fecha_alta_cliente`.
<img width="1919" height="1077" alt="05-tabla-clientes-final" src="https://github.com/user-attachments/assets/298dc077-a582-476d-a228-c92a40980d0b" />

  ![Tabla Clientes final con panel de pasos aplicados](capturas/05-tabla-clientes-final.png)

- **Tabla `Ventas`** (11 columnas): `id_operacion`, `id_cliente`, `fecha_venta`, `id_producto`, `nombre_producto`, `categoria_producto`, `cantidad`, `precio_unitario`, `descuento_pct`, `total_venta`, `moneda`, `canal_venta`.
<img width="1919" height="1079" alt="06-tabla-ventas-final" src="https://github.com/user-attachments/assets/b95521bc-ef5d-4bf1-bd9e-b0c1167dfc10" />

  ![Tabla Ventas final con panel de pasos aplicados](capturas/06-tabla-ventas-final.png)

Dejé `id_cliente` en las dos tablas a propósito: en `Clientes` identifica una sola vez a cada persona, y en `Ventas` se repite por cada compra que hizo — esa repetición es justo lo que permite conectar ambas tablas después en el modelo de Power BI, con la misma lógica que las `FOREIGN KEY` que usé en SQL para el proyecto RetailPro.
