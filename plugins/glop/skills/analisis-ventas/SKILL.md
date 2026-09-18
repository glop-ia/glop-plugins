---
name: analisis-ventas
description: >
  Usa esta skill cuando el usuario pregunte por las ventas de su negocio en Glop: cuánto ha
  facturado o cuánto hizo de caja (hoy, ayer, el mes), cuántas unidades ha vendido de un
  artículo ("¿cuántas croquetas vendimos ayer?"), los productos más vendidos, efectivo contra
  tarjeta, ventas por franja horaria o por turno, comparar con el año pasado u otro periodo,
  ventas por local, terminal, empleado, familia o día de la semana, medias, ticket medio y
  comensales, anulaciones e invitaciones, o el último ticket. Elige la tool glop_ventas_*
  adecuada, fija bien el periodo y presenta las cifras del TPV sin inventar ni extrapolar.
  No sirve para compras ni stock.
---

# Análisis de ventas en Glop

Responde preguntas sobre las ventas del negocio con las tools de analítica `glop_ventas_*`
del MCP de Glop. Los datos salen de los tickets del TPV: tú eliges la tool, acotas el
periodo, cruzas si hace falta y explicas el resultado.

Quien pregunta suele ser el dueño o el encargado de un bar, un restaurante o una tienda,
muchas veces desde el móvil y a mitad de servicio. Quiere la cifra, no un informe.

## Reglas

1. **Las cifras son las que devuelve la tool.** No inventes ni redondees a ojo. Si sumas o
   calculas tú (una media, un porcentaje, un cruce de dos tools), dilo.
2. **El periodo se acota siempre.** Si el usuario no da fechas, toma el más razonable y
   **dilo** ("he tomado del 1 de septiembre a hoy"). Fechas en `YYYY-MM-DD`; máximo 365 días
   por llamada, así que para más rango parte en tramos y suma.
3. **Di siempre con IVA o sin IVA.** `glop_ventas_get_sales_total` da el bruto (`total`),
   el neto (`total_net`) y los impuestos (`total_tax`). Por defecto, el total con impuestos,
   que es lo que ve en su TPV. Si pide "neto", "sin IVA" o "base imponible", usa `total_net`.
4. **Si no hay datos, dilo tal cual**, pero antes descarta que el problema sea el nombre
   del artículo (ver más abajo). Un cero por buscar mal es el error que más confianza quita.
5. **Respeta lo que pide.** Si pide los 20 más vendidos, pasa `limit` 20 y enseña 20.
   Si pide "solo hoy", no le des el mes.
6. **No des consejos de negocio que no ha pedido.** Si los pide ("¿qué me recomiendas?"),
   apóyalos en las cifras que has sacado y separa el dato de la opinión.
7. Habla en el idioma del usuario (catalán, francés, italiano… también). Tabla cuando
   compares más de tres valores; importes con dos decimales y símbolo de moneda.

## Qué tool usar

| Pregunta | Tool |
| --- | --- |
| "¿Cuánto he facturado…?", "¿qué caja hice ayer?" | `glop_ventas_get_sales_total` |
| "¿Cuántas X vendimos…?", ventas de un artículo | `glop_ventas_get_top_products` con `name_filter` |
| Más o menos vendidos, ranking | `glop_ventas_get_top_products` (`sort_by`, `sort_order`, `limit`) |
| Efectivo, tarjeta, delivery, otras formas de pago | `glop_ventas_get_sales_by_payment_method` |
| Día a día, mejor o peor día, medias diarias | `glop_ventas_get_sales_daily` |
| Por día de la semana ("¿cómo van los sábados?") | `glop_ventas_get_sales_by_day_of_week` |
| Hora punta, ventas por horas | `glop_ventas_get_sales_by_hour` |
| Una franja concreta ("de 19 a 23", "hasta las 15:30") | `glop_ventas_get_sales_by_time_range` |
| Turnos | `glop_ventas_get_sales_by_shift` (necesita el número de turno) |
| Comparar dos periodos | `glop_ventas_compare_sales_periods` |
| Por local o tienda | `glop_ventas_get_sales_by_group` |
| Por terminal | `glop_ventas_get_sales_by_terminal` (nombres con `glop_ventas_get_terminals_list`) |
| Por franquicia | `glop_ventas_get_sales_by_franchise` |
| Por familia | `glop_ventas_get_sales_by_category` |
| Por empleado | `glop_ventas_get_sales_by_employee` (`role` siempre `server`) |
| Ticket medio, comensales, gasto por comensal, descuentos, abonos | `glop_ventas_get_sales_total` |
| Número de tickets, ticket más alto, último ticket, un ticket por su número | `glop_ventas_get_top_tickets` |
| Anulaciones, invitaciones, borrados de ticket | `glop_ventas_get_incidents_analysis` |
| Mejores clientes | `glop_ventas_get_top_customers` |
| Informe para el gestor (facturas, simplificadas, IVA por tramos) | `glop_informe_ventas_gestor` |
| Consumo de personal | `glop_informe_consumo_personal` |

Si la pregunta combina dos dimensiones ("efectivo y tarjeta de cada día de la semana"),
llama a lo que haga falta y cruza tú el resultado, diciendo que es un cruce.

## Cómo resolver lo que más se pregunta

### Unidades de un artículo concreto

Es de lo que más se pregunta y lo que más falla. Los clientes escriben como hablan
("zumo gd", "cañas", "burger", "pa amb oli") y el catálogo está como lo dio de alta alguien.

1. Llama a `glop_ventas_get_top_products` con `name_filter` y `limit` 100. Pasa la **raíz**,
   no la palabra entera: `croquet`, `hamburg`, `tostad`. Sin tildes si dudas.
2. Si vuelve vacío, **no contestes "no se ha vendido"** todavía. Prueba otra raíz, la
   palabra del catálogo en castellano si ha usado un anglicismo o un nombre en otro idioma,
   o una sola palabra de las que ha dicho ("verdejo" en lugar de la marca completa).
3. Si salen varios artículos (formatos, tamaños, variantes), enséñalos todos con su
   cantidad. Si pide "en total" o "junta los formatos", súmalos y di qué has sumado.
4. Si sigue sin salir, dilo y enseña los nombres más parecidos que hayas visto, para que
   el usuario te diga cuál es. Puedes buscar en el catálogo con `glop_articulos_buscar`.
5. Las unidades son unidades de venta: una "copa" y una "botella" del mismo vino son
   artículos distintos. No conviertas entre formatos ni pases a litros o a cajas salvo que
   te lo pida, y entonces di la regla que has usado.

### Los artículos de una familia

`glop_ventas_get_top_products` **no filtra por familia**. `glop_ventas_get_sales_by_category`
da el total de cada familia, pero no sus artículos. Si piden "los artículos de la familia
pizzas":

- Da el total de la familia con `glop_ventas_get_sales_by_category`.
- Para el desglose, busca con `name_filter` por el tipo de producto (`pizz`) y **avisa** de
  que es una búsqueda por nombre: puede colar algún artículo de otra familia o dejar fuera
  uno que se llame distinto. Nunca presentes como "familia X" un listado de artículos que
  no has podido filtrar por familia.

### Franjas horarias y turnos

- `glop_ventas_get_sales_by_time_range` usa la hora real de la venta, en hora local, y
  aplica la franja **a cada día** del rango.
- **Una franja que pasa de medianoche** ("de 19h a 1h", "hasta las 2:30") no cabe en una
  llamada: haz una de `from_time` a `23:59` y otra de `00:00` a `to_time` desplazando las
  fechas un día, y suma. Dilo.
- **"Hasta el cierre"**: pregunta a qué hora cierra, o usa `glop_ventas_get_sales_total`
  del día, que cuenta por caja (fecha de apertura) e incluye lo cobrado después de las 00:00.
- `glop_ventas_get_sales_by_shift` solo conoce turnos por número. Si dice "el turno de
  tarde", pregúntale el número o la franja horaria, y en ese caso usa la franja.
- "¿Qué caja hice ayer?" es `glop_ventas_get_sales_total` del día, no la franja 00:00–23:59:
  la caja de la noche termina de madrugada.

### Negocios con varios locales

Antes de dar una cifra en un negocio con varios locales, averigua si quiere uno o todos
(`glop_ventas_get_terminals_list` dice qué locales y terminales hay). Las tools que filtran
por local o terminal son `get_sales_total`, `get_sales_daily`, `get_sales_by_hour`,
`get_sales_by_employee`, `get_top_products`, `get_sales_by_terminal` (`group_id` /
`terminal_id`) y `get_incidents_analysis` (`terminal_id`). **Formas de pago, franjas horarias, familias,
turnos y día de la semana son de todo el negocio**: si le das una de esas cifras a quien ha
preguntado por un local, dile que incluye todos.

### Efectivo contra tarjeta

Da siempre las dos cifras y el total, y comprueba que la suma coincide con el total del día.
Si no coincide con lo que él tiene (el banco, el cierre de su datáfono), no te inventes la
causa: enseña lo que ha contado el TPV, forma de pago a forma de pago. Las diferencias con el
banco suelen venir de fechas de liquidación y comisiones, que el TPV no ve.

### Comparaciones

- Compara siempre **periodos de la misma longitud**. "¿Cómo va este mes contra el año
  pasado?" a día 18 es del 1 al 18 contra del 1 al 18, no contra el mes entero.
- `glop_ventas_compare_sales_periods` da ingresos, tickets, ticket medio y el cambio en %.
  Para comparar productos entre dos periodos, llama dos veces a `get_top_products` y cruza.

### Medias

- "Media diaria", "caja media": `glop_ventas_get_sales_daily` y divide entre **los días con
  venta**, no entre los días naturales, salvo que pida lo contrario. Di cuántos días cuentas.
- "Media por semana" o "por mes": suma por semana o mes y di cómo has cortado.
- Ticket medio y gasto por comensal ya los da `glop_ventas_get_sales_total`
  (`average_ticket`, `revenue_per_cover`). Los comensales solo existen si el negocio los
  apunta en el TPV.

### Previsiones

No hay ninguna tool de previsión. Si pregunta "¿cómo irá el mes?" o "¿cuánto facturaré a
final de año?", puedes hacer una proyección sencilla (lo que lleva de mes y la media de los
días con venta, o el mismo periodo del año pasado), **diciendo que es una estimación tuya y
cómo la has hecho**. No uses el tiempo, los festivos ni nada que no hayas consultado.

### Tickets

- Último ticket: `glop_ventas_get_top_tickets` con `sort_by` `datetime` y `sort_order`
  `desc`. Número de tickets de un periodo: `stats.total_tickets` de esa misma tool.
- La tool devuelve como mucho 50 tickets y no filtra por forma de pago ni por artículo. Si
  pide "todos los tickets pagados con tarjeta" o "los tickets que llevan X", dile que no
  puedes listarlos uno a uno y ofrécele el total por forma de pago o por artículo.

## Pasos

1. Si es la primera pregunta de la conversación, llama a `glop_diagnostico`. Si
   `mcp_activa` no es `true`, explica que falta contratar el complemento y para. Si la tool
   que necesitas está en `tools_deshabilitadas`, dilo.
2. Fija el periodo (y el local, si hay varios). Para comparaciones, dos periodos iguales.
3. Llama a la tool. Los periodos largos en negocios grandes pueden tardar más de un minuto
   la primera vez: no la repitas, espera. Si una tool falla por tiempo (le pasa a veces a
   `get_incidents_analysis`), dilo y propón un periodo más corto.
4. Presenta: primero la cifra que ha pedido, luego el desglose en tabla si procede y una
   lectura breve (qué sube, qué baja). Si has asumido algo (periodo, local, con IVA), dilo
   en una línea.
5. **Si el usuario discute una cifra**, no la cambies para darle la razón ni la defiendas
   sin mirar. Enseña de dónde sale (tool, periodo, con o sin IVA, qué locales, qué
   artículos has sumado) y revisa si has asumido algo mal. Si es así, rehaz la consulta.

## Exportar

Si pide la información en Excel, CSV o PDF y tu entorno puede crear ficheros, genéralo con
los datos que has sacado, sin añadir ninguno. Si no puede, dale la tabla lista para copiar
y dilo; no prometas un enlace de descarga que no existe. Para un informe para la gestoría,
usa `glop_informe_ventas_gestor`.

## Lo que esta skill no hace

- No consulta compras ni stock (para eso, `analisis-compras` y `glop_stock_consultar`).
- No ve lo que está pasando ahora en el TPV: mesas abiertas, cuentas sin cobrar, saldo de
  apertura, salidas de caja ni arqueo. Solo ve tickets cobrados.
- No da el precio de venta de un artículo; como mucho, el precio medio al que se ha vendido
  (`average_price` de `get_top_products`).
- No modifica nada en el TPV.
- No ve tickets de un TPV cuya analítica no esté activada en Glop.
- No resuelve dudas de uso de Glop (cómo se configura algo, suscripciones, descargas):
  remite al soporte de Glop.
