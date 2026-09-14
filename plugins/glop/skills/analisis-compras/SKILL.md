---
name: analisis-compras
description: >
  Usa esta skill cuando el usuario pregunte por sus compras a proveedores en Glop: cuánto ha
  gastado y en qué proveedores, qué productos compra más, cómo ha evolucionado el precio de
  compra de un artículo ("¿me han subido la Coca-Cola?"), o qué proporción de lo que factura
  se le va en compras. También para buscar documentos de compra ya registrados y consultar
  existencias. Elige la tool glop_compras_* o de stock adecuada y traslada siempre las
  limitaciones de los datos. No registra albaranes: para eso está alta-albaran-compra.
---

# Análisis de compras y stock en Glop

Responde preguntas sobre el gasto en proveedores, los precios de compra y las existencias con
las tools `glop_compras_*`, `glop_compras_documentos_buscar` y `glop_stock_consultar`.

## Reglas

1. **Las cifras son las de la tool.** Cada respuesta de compras dice qué tipos de documento
   ha contado en `document_types_included`: trasládalo. Por defecto cuentan albaranes y
   facturas de proveedor (tipos 2 y 3); los pedidos (tipo 1) no son gasto.
2. **Todo sin IVA cuando se compara con ventas.** `glop_compras_compare_purchases_vs_sales`
   compara base contra base y trae un `disclaimer`: repítelo. **No es margen por artículo ni
   coste de la mercancía vendida.**
3. **La analítica solo ve lo registrado por los canales de Glop** (módulo de Compras, este
   asistente o el chat de la app). Lo tecleado directamente en el TPV no está. Si el usuario
   ve menos gasto del que espera, esta es la primera causa.
4. **Un documento editado o borrado en el TPV no se actualiza aquí.** Dilo si sale una cifra
   que el usuario discute.
5. Los abonos son líneas negativas: restan solos en el gasto y no salen en el histórico de
   precios.
6. Habla en el idioma del usuario. Fechas en `YYYY-MM-DD`.

## Cómo elegir la tool

| El usuario pregunta… | Tool |
| --- | --- |
| "¿Cuánto le he comprado a X?", "¿en qué proveedores se me va el dinero?" | `glop_compras_get_purchases_by_supplier` |
| "¿Qué compro más?", "¿en qué se me va el dinero de compras?" | `glop_compras_get_top_purchased_products` |
| "¿Cómo ha evolucionado el precio de…?", "¿me han subido el precio?" | `glop_compras_get_purchase_price_history` |
| "¿Qué parte de lo que facturo se va en compras?" | `glop_compras_compare_purchases_vs_sales` (máximo 366 días; `granularity` `total` o `month`) |
| Ver un albarán o pedido concreto ya registrado | `glop_compras_documentos_buscar` y `glop_compra_documento_obtener` |
| Existencias de un artículo o almacén | `glop_stock_consultar` (con `glop_almacenes_listar` si hay varios) |
| Quién sirve un artículo y a qué precio | `glop_mapeo_buscar` |

## Pasos

1. Si es la primera pregunta de la conversación, llama a `glop_diagnostico` y confirma que
   `mcp_activa` es `true`.
2. Fija el periodo (`from`, `to`). Si el usuario no lo dice, toma el mes en curso y dilo.
3. Para preguntas por proveedor o artículo, usa los filtros por nombre (`supplier_name`,
   `product_name`), que ignoran mayúsculas y acentos. Si hay varias coincidencias, pregunta.
4. Llama a la tool. `compare_purchases_vs_sales` puede tardar más de un minuto la primera
   vez en sitios grandes: no la repitas.
5. Presenta la cifra, el desglose en tabla y las limitaciones que apliquen (reglas 1 a 5).

## Lo que esta skill no hace

- No registra ni modifica documentos: para meter un albarán, `alta-albaran-compra`.
- No analiza ventas (para eso, `analisis-ventas`).
- No calcula márgenes por artículo: Glop no cruza escandallo con ventas en estas tools.
