---
name: alta-albaran-compra
description: >
  Usa esta skill SIEMPRE que el usuario adjunte una foto, escaneo o PDF de un albarán de
  compra o de una factura de proveedor y quiera registrarlo en Glop: "registra este albarán",
  "mete esta factura de compra", "lee este albarán del proveedor", o simplemente adjunte la
  imagen y pida procesarla. Lee el documento, resuelve proveedor y artículos con las tools del
  MCP de Glop, enseña la previsualización que calcula el TPV y, tras confirmación explícita del
  usuario, registra el documento. Requiere el MCP de Glop conectado y el complemento
  contratado. No sirve para albaranes de venta a clientes ni para pedidos a proveedor.
---

# Alta de albarán de compra en Glop

Convierte la foto o el PDF de un albarán de proveedor en un documento de compra en Glop. El
proceso es **conversacional y por fases**: cada fase se cierra con el usuario antes de pasar
a la siguiente.

Esta skill dice **en qué orden** hacer las cosas y **cómo hablar con el usuario**. Las reglas
de negocio (qué se escribe y cuándo, cómo se convierten envases, cómo se cuadran los totales,
qué datos exige un artículo nuevo) **las dicta el servidor**: están en la descripción de cada
tool y en sus respuestas. Cuando una respuesta traiga `siguiente_paso` o una instrucción,
mándan ellas sobre este documento.

> **El OCR lo pones tú.** Glop no lee la imagen: tú la lees y él valida, calcula y registra.

## Lo que no se negocia

1. **Nada se escribe sin un sí explícito del usuario**, y el sí se pide sobre la
   previsualización que devuelve el servidor, nunca sobre un resumen tuyo.
2. **No inventes datos.** Lo que no leas con seguridad, se pregunta.
3. **No decidas un artículo por parecido de nombre.** Presenta candidatos y que elija el
   usuario.
4. **Los cálculos los hace Glop.** No conviertas envases ni calcules totales de cabecera.
5. **Descartar el documento entero siempre es una salida válida.**
6. Habla en el idioma del usuario, claro y sin tecnicismos.

## Fase 0 — Preparación

1. Comprueba que hay imagen o PDF adjunto. Si no, pídelo.
2. Llama a `glop_diagnostico`. Si `mcp_activa` no es `true` o falta alguna de las tools que
   necesitas en `tools_deshabilitadas`, dilo ahora y para.
3. Carga `glop_terminales_listar`, `glop_almacenes_listar`, `glop_ivas_listar` y
   `glop_envases_listar`.
4. Si hay varios terminales, pregunta cuál registra el albarán. Si solo hay uno, úsalo y dilo.

## Fase 1 — Lectura del documento

Extrae la estructura completa y **anota dónde tienes poca confianza**:

- **Proveedor**: nombre o razón social, NIF, dirección, teléfono.
- **Documento**: número del albarán del proveedor, fecha del documento, fecha de entrega.
- **Líneas**: referencia del proveedor, código de barras, descripción, cantidad, unidad o
  formato (cajas, unidades, kg), precio, descuento, importe, tipo de IVA (si viene como R1, R2
  o R3, guárdalo tal cual).
- **Totales**: bases por IVA, cuotas, recargo, descuentos de pie, portes y total.
- **Anotaciones a mano**: abonos, cantidades tachadas, líneas añadidas. Sepáralas de lo
  impreso: lo impreso puede no ser lo que se recibió.

Antes de seguir, comprueba tú mismo que cantidad × precio − descuento ≈ importe de cada línea.
Si no cuadra, vuelve a mirar esa zona con zoom. No sigas con cifras incoherentes.

Si el papel indica más hojas de las que tienes ("página 1 de 2", sin total), pide las que
faltan antes de continuar.

## Fase 2 — El proveedor

1. Búscalo con `glop_proveedores_buscar`, primero por NIF normalizado y, si no, por palabras
   del nombre (sin S.L. ni S.A.).
2. Si existe, enseña la coincidencia (ID, nombre, NIF) y sigue.
3. Si no existe, propón el alta con los datos leídos y usa `glop_proveedor_crear`. Un ticket
   de un establecimiento (mercado, supermercado) también necesita su proveedor: pregunta los
   datos que no estén en el papel.

## Fase 3 — Los artículos, línea a línea

Para cada línea llama a **`glop_articulo_resolver`** con todo lo que traiga el papel
(`provider_id`, `provider_ref`, `barcode`, `description`) y **haz lo que diga
`siguiente_paso`**. No busques a mano con `glop_articulos_buscar`.

- **Resuelto**: usa ese `PRODUCT_ID`.
- **Candidatos**: enséñaselos al usuario y que elija. Si venía referencia del proveedor,
  propón guardar el vínculo con `glop_mapeo_crear`: es lo que hace que el siguiente albarán
  de ese proveedor se reconozca solo.
- **No existe**: ofrece tres salidas: crearlo con `glop_articulo_crear` (la tool te dirá qué
  datos necesita; pregúntaselos al usuario, no los inventes), asociarlo a un artículo que
  indique el usuario, o dejar la línea fuera.

Manda cada línea como viene en el papel: la cantidad en el envase que indique (cajas,
unidades) y el IVA impreso. Si algo de la línea no coincide con lo que Glop tiene en la ficha,
la previsualización lo avisará; entonces se pregunta, no se decide.

### Si dejas una línea fuera

Los importes del papel la incluyen y el documento no. **Resta del total impreso el importe de
esa línea con su IVA** y dilo en el resumen ("total del albarán 32,67 €; se registran 26,40 €
porque la línea X queda fuera"). Deja constancia en `NOTES`. No mandes el total del papel y
luego fuerces la aceptación de la diferencia: apagarías la comprobación que detecta líneas mal
leídas.

## Fase 4 — Previsualización

Llama a `glop_compra_documento_crear` **sin `confirmar`** y **sin pedir permiso para
hacerlo**: esa llamada no escribe nada. Devuelve el documento calculado por el TPV, el
resultado del cuadre y los avisos (variación de precio de compra, envases, IVA distinto al
de la ficha). Es ese resumen, y no uno tuyo, el que se enseña al usuario.

Si el cuadre falla, **no confirmes**. La respuesta dice qué línea o grupo falla; busca la
causa en el papel en este orden:

1. Una línea que dejaste fuera sin ajustar el total (fase 3).
2. Un descuento de pie, portes o un cargo que no leíste.
3. Una línea que se te pasó.
4. Una cifra mal leída: vuelve a esa zona con zoom.

Plantéaselo al usuario con la línea concreta. Solo si acepta la diferencia de forma expresa,
y después de descartar las cuatro causas, repite con lo que la tool indique para aceptarla.

**El aviso de variación del precio de compra enséñalo siempre**, con el precio anterior y el
nuevo: es lo más útil que hace este flujo.

## Fase 5 — Confirmación y alta

1. Presenta el resumen de la previsualización: proveedor (y si es alta nueva), cabecera, tabla
   de líneas (artículo de Glop, línea del papel, cantidad, envase, precio, IVA, total), vínculos
   nuevos, totales calculados frente a los impresos, variaciones de precio y qué stock se va a
   mover. Todas las dudas del papel (anotaciones a mano, contradicciones) van en este mismo
   mensaje.
2. Pide confirmación. Sin un sí claro, no hay alta.
3. Repite la llamada con `confirmar=true`. Comprueba en la respuesta que las existencias se
   movieron como se anunció; si algo no cuadra, dilo.
4. Informa: serie y número del documento, total, y que puede verlo en Glop en Documentos de
   compra.

Si el alta falla, **no reintentes a ciegas**: comprueba con `glop_compras_documentos_buscar`
si el documento quedó registrado. Un alta que falla no deja el documento a medias.

## Lo que esta skill no hace

- No corrige ni borra documentos ya registrados: eso se hace desde Glop.
- No toca el escandallo. Si creas un artículo, avisa de que nace sin él y no descontará nada
  al vender hasta que se complete en Glop.
- No registra pedidos (tipo 1) ni convierte documentos.
