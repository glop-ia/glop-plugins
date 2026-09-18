---
name: revision-carta
description: >
  Usa esta skill cuando el usuario quiera revisar su carta o su surtido con los datos de venta
  de Glop: "¿qué platos tiran y cuáles sobran?", "¿qué debería quitar de la carta?", "¿cómo se
  ha vendido cada producto este año contra el pasado?", "¿cómo me ha afectado subir el precio
  del menú?", "¿qué hamburguesa se vende más?", "ranking de la familia postres". Monta un
  análisis guiado con glop_ventas_get_top_products y glop_ventas_get_sales_by_category:
  productos estrella, productos que casi no salen, qué sube y qué baja entre dos periodos y el
  antes y el después de un cambio de precio. Separa siempre el dato de la opinión. Para una
  pregunta suelta sobre un producto está analisis-ventas. No calcula márgenes por plato.
---

# Revisión de carta

El dueño quiere decidir qué se queda, qué sale y qué empuja. Esta skill le da los datos de
venta ordenados para decidir. La decisión es suya.

Las cifras salen de las tools. Tú fijas el alcance, cruzas periodos, clasificas y explicas.

## Reglas

1. **Dato y opinión, separados.** Primero las cifras; después, si las pide, tu lectura,
   marcada como tal ("con estos datos, yo miraría…"). Nunca "hay que quitar X".
2. **Periodos comparables.** Misma longitud y, si puedes, los mismos días de la semana y la
   misma temporada. Un producto de verano comparado con marzo siempre "baja".
3. **Unidades e ingresos, los dos.** Un producto puede vender poco y facturar mucho. Cuando
   ordenes, di por qué (`sort_by` `quantity` o `revenue`).
4. **Sin margen por plato.** La analítica sabe a cuánto se ha vendido cada artículo, no
   cuánto cuesta hacerlo: el escandallo no está aquí y el precio de compra es de los
   ingredientes, no del plato. Si pregunta por rentabilidad, dilo, y como mucho da el precio
   medio de venta (`average_price`).
5. **Lo que no se vende no aparece.** Un artículo sin ninguna venta en el periodo no sale en
   la analítica. "Lo que menos se vende" es lo que menos ha vendido *de lo que ha vendido
   algo*; si quiere saber qué no ha salido nunca, díselo.
6. Habla en su idioma. Tablas para los rankings; importes con dos decimales.

## Paso 1 — Alcance

Pregunta lo que falte, en un solo mensaje:

- **Periodo**: por defecto, los últimos 90 días. Para comparar, el mismo tramo del año
  anterior o los 90 días previos. Máximo 365 días por llamada.
- **Qué parte de la carta**: toda, una familia ("postres"), o un tipo de producto
  ("hamburguesas", "gildas").
- **Local**, si tiene varios (`glop_ventas_get_terminals_list`).
- **La pregunta de fondo**: qué quitar, qué empujar, cómo va un producto, o el efecto de un
  cambio de precio. Cambia qué enseñas primero.

Si es la primera llamada de la conversación, `glop_diagnostico`; si `mcp_activa` no es
`true`, dilo y para.

## Paso 2 — Datos

### Toda la carta

- `glop_ventas_get_sales_by_category` del periodo: peso de cada familia en unidades e
  ingresos.
- `glop_ventas_get_top_products` con `limit` 100 y `sort_by` `revenue`: los que más
  facturan. Otra llamada con `sort_by` `quantity`.
- `glop_ventas_get_top_products` con `sort_order` `asc`: los que menos salen.

Con más de 100 artículos vendidos no vas a verlos todos en una llamada: dilo y céntrate en
los extremos o trabaja por familia.

### Una familia o un tipo de producto

`glop_ventas_get_top_products` **no filtra por familia**. Usa `name_filter` con la raíz del
tipo de producto (`hamburg`, `postre`, `gilda`) y `limit` 100, y avisa de que es una
selección por nombre: puede colarse un artículo de otra familia o quedarse fuera uno que se
llame distinto. El total de la familia, de `glop_ventas_get_sales_by_category`, te sirve
para comprobar cuánto se te escapa: si la suma de lo que has encontrado queda muy lejos del
total de la familia, dilo.

### Comparar dos periodos

Las mismas llamadas para cada periodo, con el mismo filtro, y cruza por nombre de artículo.
Para cada uno: unidades e ingresos de los dos periodos, diferencia y %. Los artículos que
solo aparecen en un periodo, aparte ("nuevos" y "desaparecidos"): puede que sean altas o
bajas de carta o cambios de nombre.

### Antes y después de un cambio de precio

1. Pregunta la **fecha** del cambio y qué artículos (si no la sabe, `average_price` en
   periodos cortos sucesivos muestra cuándo cambió).
2. Compara el mismo número de días antes y después, con los mismos días de la semana.
3. Enseña, para cada artículo: precio medio, unidades e ingresos antes y después.
4. Pon al lado la evolución del **total del negocio** en los mismos dos periodos
   (`glop_ventas_compare_sales_periods`). Si todo el negocio ha bajado un 10 %, que ese plato
   baje un 10 % no es por el precio.
5. Lectura, solo con lo que dicen los números: "vende un 6 % menos de unidades pero
   factura un 9 % más". Nada de elasticidades ni causas que no puedas ver.

## Paso 3 — Presentar

1. **Resumen en tres o cuatro líneas**: qué familia pesa más, los tres productos que más
   facturan, cuántos productos hacen el 80 % de la facturación si se ve claro.
2. **Estrellas**: los que están arriba en unidades y en ingresos.
3. **Los que casi no salen**: los últimos por unidades, con cuánto facturan. Si alguno
   factura bien pese a vender poco, sepáralo.
4. **Lo que sube y lo que baja** (si hay comparación).
5. **Tu lectura**, solo si la pide, marcada como opinión y apoyada en las cifras de arriba.
   Lo que no sabes (coste, tiempo de cocina, si un plato arrastra a otros), dilo como
   pregunta, no lo supongas.

Si quiere el análisis en Excel o PDF y tu entorno puede crear ficheros, genéralo con estas
mismas tablas.

## Lo que esta skill no hace

- No calcula márgenes ni coste por plato (no hay escandallo en la analítica).
- No cambia precios ni da de baja artículos en el TPV.
- No ve artículos que no se han vendido nunca en el periodo.
- No responde preguntas sueltas del tipo "¿cuántas X vendimos ayer?": eso es
  `analisis-ventas`.
