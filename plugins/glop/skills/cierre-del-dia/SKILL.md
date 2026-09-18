---
name: cierre-del-dia
description: >
  Usa esta skill cuando el usuario quiera el resumen de un día de su negocio en Glop: "¿cómo ha
  ido hoy?", "hazme el cierre de ayer", "¿qué caja hicimos el sábado?", "resumen del día con
  efectivo y tarjeta", "¿cómo va el día?" a mitad de servicio. Da de una vez el total con y sin
  IVA, efectivo contra tarjeta, tickets y ticket medio, los productos más vendidos, las
  anulaciones e invitaciones y la comparación con el mismo día de la semana anterior, todo con
  las tools glop_ventas_*. Sirve para un día o para un fin de semana; para periodos más largos
  o preguntas sueltas está analisis-ventas. No es un arqueo de caja: no ve el efectivo del cajón.
---

# Cierre del día

El dueño de un bar o de una tienda quiere saber, al final del día o a la mañana siguiente,
cómo ha ido. Hoy lo pregunta en cuatro mensajes (cuánto, cuánto en efectivo y cuánto en
tarjeta, qué se ha vendido, si ha habido anulaciones). Esta skill se lo da en uno.

Las cifras salen de las tools. Tú eliges el día, llamas, cuadras y presentas.

## Reglas

1. **El día es el de la caja.** Una caja que abre el sábado y cierra el domingo a las 2:00
   es el sábado. Por eso el total sale de `glop_ventas_get_sales_total` con `from` y `to`
   iguales al día, que cuenta por fecha de apertura de caja. No uses una franja de
   00:00 a 23:59 para el total.
2. **Qué día.** "Hoy" es hoy; "ayer" y "el sábado" son el último que haya pasado. Si
   pregunta por la mañana temprano "¿cómo fue?" sin más, es ayer. Dilo siempre con la fecha
   ("sábado 12 de septiembre").
3. **Hoy es un día a medias.** Si el día es hoy, di que es lo que lleva cobrado hasta ahora
   y que las mesas abiertas no cuentan: la analítica solo ve tickets cobrados.
4. **Cuadra antes de enseñar.** La suma de las formas de pago tiene que dar el total. Si no
   da, enséñalo tal cual y di la diferencia; no la repartas ni la escondas.
5. **Nada de consejos** si no los pide. Una línea de lectura ("un 12 % por encima del sábado
   pasado, sobre todo por la tarde") sí; recomendaciones, no.
6. Habla en su idioma. Importes con dos decimales y símbolo de moneda.

## Pasos

1. Si es la primera pregunta de la conversación, `glop_diagnostico`. Si `mcp_activa` no es
   `true`, explícale que falta el complemento y para.
2. **¿Tiene varios locales?** Si no lo sabes, `glop_ventas_get_terminals_list`. Con más de
   un local, pregunta si quiere uno o todos, salvo que ya lo haya dicho.
3. Llama, con el mismo día en `from` y `to`:
   - `glop_ventas_get_sales_total` (con `group_id` o `terminal_id` si es un local): `total`,
     `total_net`, `total_tax`, `num_tickets`, `average_ticket` y, si los hay, comensales,
     descuentos y abonos.
   - `glop_ventas_get_sales_by_payment_method`.
   - `glop_ventas_get_top_products` con `limit` 10 (con `group_id` o `terminal_id` si es un
     local).
   - `glop_ventas_get_sales_by_hour` (con `group_id` si es un local), para decir cuál fue la
     hora fuerte.
   - `glop_ventas_get_incidents_analysis` para anulaciones, borrados e invitaciones. Si falla
     por tiempo, sigue sin ella y dilo en una línea: no bloquees el resumen por esto.
   - `glop_ventas_get_sales_total` del **mismo día de la semana anterior**, con el mismo
     filtro de local, para comparar.
4. Monta el resumen.

## Cómo presentarlo

Corto, que se lee en el móvil. En este orden:

1. **Una línea con la cifra**: "Sábado 12/09: 3.482,50 € con IVA (3.165,91 € sin IVA),
   214 tickets, ticket medio 16,27 €. Un 8 % más que el sábado anterior."
2. **Formas de pago**: tabla de una fila por forma de pago con importe y %, y el total.
3. **Lo más vendido**: los cinco primeros por unidades, con importe. Si pide más, los diez.
4. **Hora fuerte**: la hora o las dos horas con más venta, en una línea.
5. **Incidencias**: número e importe de anulaciones, borrados de ticket e invitaciones. Si
   una sola persona acumula muchas, dilo con su nombre tal como lo da la tool, sin juzgar.
6. **Comensales y gasto por comensal**, solo si el negocio los apunta (si la tool no los
   trae, no los menciones).

Si el negocio tiene varios locales y ha pedido todos, añade una tabla con el total de cada
local (`glop_ventas_get_sales_by_group`) y avisa de que las formas de pago, las horas y las
incidencias son de todo el negocio, porque esas tools no se pueden separar por local.

### Un fin de semana o varios días seguidos

Mismo esquema con `from` y `to` abarcando los días, más una tabla día a día con
`glop_ventas_get_sales_daily`. Compara con los mismos días de la semana anterior.

## Si algo no le cuadra

- **"En el datáfono / en el banco tengo otra cifra"**: enseña la tarjeta tal como la ha
  contado el TPV. Las diferencias con el banco suelen ser fechas de liquidación y
  comisiones, que el TPV no ve. No te inventes tickets perdidos.
- **"Falta dinero de efectivo"**: esta skill no ve el cajón. Lo que puedes darle es el
  efectivo cobrado según el TPV, para que él lo compare con su recuento.
- **"No salen las ventas de hoy"**: puede que el TPV no haya sincronizado todavía o que la
  analítica no esté activa para ese terminal. Dilo así; no lo des por "día sin ventas".

## Lo que esta skill no hace

- No hace el arqueo ni ve el efectivo real, el saldo de apertura, las salidas de caja ni
  las mesas abiertas.
- No cierra la caja ni toca nada en el TPV.
- No hace análisis de periodos largos ni responde preguntas sueltas: eso es
  `analisis-ventas`.
