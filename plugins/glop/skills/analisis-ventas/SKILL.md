---
name: analisis-ventas
description: >
  Usa esta skill cuando el usuario pregunte por las ventas de su negocio en Glop: cuánto ha
  vendido en un periodo, comparar periodos ("esta semana contra la anterior"), ventas por hora,
  por día de la semana, por empleado, por familia, por terminal, por forma de pago, productos
  o clientes que más venden, tickets más altos o incidencias (anulaciones, invitaciones). Elige
  la tool glop_ventas_* adecuada, fija bien el rango de fechas y presenta el resultado con las
  cifras del TPV, sin inventar ni extrapolar. No sirve para compras ni stock.
---

# Análisis de ventas en Glop

Responde preguntas sobre las ventas del negocio con las tools de analítica `glop_ventas_*`
del MCP de Glop. Los datos vienen de los tickets del TPV; tú eliges la tool, acotas el periodo
y explicas el resultado.

## Reglas

1. **Las cifras son las que devuelve la tool.** No calcules totales a mano cuando exista una
   tool que los dé, y no extrapoles ("a este ritmo…") sin decir que es una estimación tuya.
2. **El periodo se acota siempre.** Si el usuario no dice fechas, pregunta o asume el más
   razonable y **dilo** ("he tomado el mes en curso"). Fechas en `YYYY-MM-DD`.
3. **Si no hay datos, dilo tal cual.** Una respuesta vacía no es un error tuyo ni del usuario:
   puede que el sitio no tenga analítica activa o que el periodo no tenga tickets.
4. **No mezcles con IVA y sin IVA.** Di siempre qué base estás dando; si la tool devuelve
   ambas, usa la que el usuario pida y, por defecto, el total con impuestos que ve en su TPV.
5. Habla en el idioma del usuario, con formato de tabla cuando compares más de tres valores.

## Cómo elegir la tool

| El usuario pregunta… | Tool |
| --- | --- |
| "¿Cuánto he vendido…?" (un total) | `glop_ventas_get_sales_total` |
| Evolución día a día | `glop_ventas_get_sales_daily` |
| "¿Esta semana contra la anterior?", "¿este mes contra el año pasado?" | `glop_ventas_compare_sales_periods` |
| Por hora / franja horaria | `glop_ventas_get_sales_by_hour`, `glop_ventas_get_sales_by_time_range` |
| Por día de la semana | `glop_ventas_get_sales_by_day_of_week` |
| Por turno | `glop_ventas_get_sales_by_shift` |
| Por empleado | `glop_ventas_get_sales_by_employee` |
| Por familia / grupo | `glop_ventas_get_sales_by_category`, `glop_ventas_get_sales_by_group` |
| Por terminal / franquicia | `glop_ventas_get_sales_by_terminal`, `glop_ventas_get_sales_by_franchise` (antes `glop_ventas_get_terminals_list` para los nombres) |
| Por forma de pago | `glop_ventas_get_sales_by_payment_method` |
| Productos, clientes o tickets top | `glop_ventas_get_top_products`, `glop_ventas_get_top_customers`, `glop_ventas_get_top_tickets` |
| Anulaciones, invitaciones, descuentos | `glop_ventas_get_incidents_analysis` |
| Informe para el gestor / asesoría (facturas, simplificadas, IVA por tramos) | `glop_informe_ventas_gestor` |
| Consumo de personal | `glop_informe_consumo_personal` |

Si la pregunta combina dos dimensiones ("ventas por empleado los sábados"), llama a las dos
tools y cruza tú el resultado, diciendo que es un cruce.

## Pasos

1. Si es la primera pregunta de la conversación, llama a `glop_diagnostico` y confirma que
   `mcp_activa` es `true`. Si no lo es, explica que falta contratar el complemento y para.
2. Fija `from` y `to`. Para comparaciones, fija los dos periodos con la misma longitud.
3. Llama a la tool. Si tarda (los periodos largos en sitios grandes pueden tardar más de un
   minuto la primera vez), no la repitas: espera.
4. Presenta: cifra principal, desglose en tabla si procede, y una lectura breve (qué sube,
   qué baja). No añadas recomendaciones de negocio que el usuario no haya pedido.

## Lo que esta skill no hace

- No consulta compras ni stock (para eso, `analisis-compras` y `glop_stock_consultar`).
- No modifica nada en el TPV.
- No ve tickets tecleados en un TPV cuya analítica no esté activada en Glop.
