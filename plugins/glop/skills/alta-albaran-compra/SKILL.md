---
name: alta-albaran-compra
description: >
  Usa esta skill SIEMPRE que el usuario adjunte una foto, escaneo o PDF de un albarán de
  compra o de una factura de proveedor y quiera registrarlo en Glop — "registra este
  albarán", "mete esta factura de compra", "lee este albarán del proveedor", o simplemente
  adjunte la imagen y pida procesarla. Lee el documento, coteja proveedor y artículos
  contra el TPV a través del MCP de Glop, comprueba que las cuentas cuadran con los
  importes impresos y, tras confirmación explícita del usuario, registra el documento.
  Requiere el servidor MCP de Glop conectado y el add-on contratado. No sirve para
  albaranes de venta a clientes ni para pedidos a proveedor.
---

# Alta de albarán de compra en Glop

Convierte la foto o el PDF de un albarán de proveedor en un documento de compra real en Glop.
El proceso es **conversacional y por fases**: cada fase se cierra con el usuario antes de pasar
a la siguiente, y **no se registra nada sin una confirmación explícita del resumen final**.

Lee `references/conceptos-glop.md` antes de la fase 3: sin entender qué es un artículo de
compra y qué es un escandallo, el emparejamiento de líneas se hace mal.

> **El OCR lo pones tú.** Glop no lee la imagen: tú la lees y él valida, calcula y registra.

---

## Las reglas que no se negocian

1. **Nada se escribe sin un sí explícito del usuario.** Son tres escrituras posibles —alta de
   proveedor, vínculo artículo-proveedor y el albarán— y las tres se confirman por separado.
   Todo lo demás son consultas.
2. **No inventes datos.** Lo que no leas con seguridad, se pregunta. Una cifra dudosa se
   vuelve a mirar con zoom antes de preguntar.
3. **No decidas nunca un artículo por parecido de nombre.** Presenta candidatos y que elija el
   usuario. Un vínculo equivocado se reutiliza en todas las compras siguientes.
4. **Los cálculos los hace Glop.** No conviertas cajas a unidades, no calcules totales de
   cabecera y no toques stock ni precios: el servidor lo hace y lo devuelve.
5. **Un albarán que no cuadra no entra**, salvo que el usuario acepte la diferencia
   expresamente.
6. **Un albarán ya registrado no se vuelve a registrar.** Se enseña el que ya existe.
7. **Lo registrado no se corrige desde aquí.** El MCP no borra ni edita documentos: si algo
   quedó mal, se arregla en Glop o en la app. No lo intentes ni lo prometas.
8. Habla en español, claro y sin tecnicismos.

---

## Fase 0 — Preparación

1. Comprueba que hay imagen o PDF adjunto.
2. Llama a `glop_diagnostico`. Confirma que `mcp_activa` es `true` y mira
   `tools_deshabilitadas`: si falta alguna de las que necesitas, dilo ahora y no a mitad del
   proceso.
3. Carga el contexto con `glop_terminales_listar`, `glop_almacenes_listar`, `glop_ivas_listar`
   y `glop_envases_listar`.
4. Si hay varios terminales, pregunta cuál registra el albarán: define la serie, el almacén y
   el empleado. Si solo hay uno, úsalo y dilo.

## Fase 1 — Lectura del documento

Extrae la estructura completa y **anota dónde tienes poca confianza**:

- **Proveedor**: nombre o razón social, NIF, dirección, CP, población, teléfono.
- **Documento**: número de albarán del proveedor (máximo **15 caracteres**), fecha del
  documento, fecha de entrega si figura.
- **Líneas**: referencia del proveedor, código de barras, descripción, cantidad, unidad o
  formato (cajas, unidades, kg…), precio, % de descuento, importe de la línea, % de IVA.
- **Totales**: bases por tipo de IVA, cuotas, recargo de equivalencia, descuentos de pie,
  portes y total del documento.

Antes de seguir, comprueba tú mismo que cantidad × precio − descuento ≈ importe de la línea.
Si no cuadra, vuelve a mirar esa zona de la imagen con zoom. No sigas con cifras incoherentes:
las va a rechazar el servidor en la fase 5, y para entonces habrás hecho trabajo de más.

## Fase 2 — El proveedor

1. Búscalo con `glop_proveedores_buscar`, primero por **NIF** normalizado (sin puntos, guiones
   ni espacios) y si no, por palabras del nombre (sin S.L. ni S.A.).
2. **Si existe**, enseña la coincidencia (ID, nombre, NIF) y sigue.
3. **Si no existe**, propón el alta con los datos leídos y pide confirmación. Se crea con
   `glop_proveedor_crear`, que previsualiza primero y solo escribe con `confirmar=true`.

## Fase 3 — Los artículos, línea a línea

> Antes de esta fase, lee `references/conceptos-glop.md` si no lo has hecho.

Para cada línea, usa **`glop_articulo_resolver`** y pásale todo lo que traiga el papel:
`provider_id`, `provider_ref`, `barcode` y `description`. Prueba cuatro vías en orden y para en
la primera que acierta. **No busques a mano con `glop_articulos_buscar`.**

Lee siempre el campo `siguiente_paso` de la respuesta: dice qué hacer en cada caso.

Según lo que devuelva:

- **`resuelto: true`** → usa ese `PRODUCT_ID` y sigue.
- **`resuelto: false` con candidatos** → enséñaselos al usuario y que elija. Si venía una
  referencia del proveedor, propón guardar el vínculo con `glop_mapeo_crear` para que el
  próximo albarán de ese proveedor se reconozca solo. **Es lo que hace que el catálogo
  aprenda**: cada documento procesado deja el siguiente más fácil.
- **El artículo no existe** → ofrece tres salidas: crearlo con `glop_articulo_crear`
  (preguntando familia de compra, IVA, envase y si lleva stock — no los inventes),
  asociarlo a un artículo que indique el usuario, o dejar la línea fuera del albarán.

### Si dejas una línea fuera, ajusta el total impreso

Los importes del papel incluyen esa línea; el documento que vas a registrar, no. Si mandas el
total impreso tal cual, el cuadre de la fase 5 fallará por una diferencia que **no es un error
de lectura**.

**Resta del `PRINTED_TOTAL` el importe de la línea excluida, con su IVA**, y dilo en el resumen:
*"total del albarán 32,67 €; se registran 26,40 € porque la línea X queda fuera"*. No mandes el
total del papel y luego fuerces `ACCEPT_DIFFERENCE`: eso apaga la única comprobación que
detecta las líneas mal leídas.

Y deja constancia en `NOTES` de qué línea quedó fuera y por qué.

**Si creas un artículo, avisa de que nace sin escandallo**: entrará en el almacén, pero no
descontará nada al vender hasta que alguien lo complete en Glop.

### Lo que tienes que mirar en cada línea

- **Cajas o unidades.** Si el papel factura en cajas, manda la cantidad en `PACKAGE_UD` con su
  envase y **deja que Glop convierta**. Te devolverá cuántas unidades han entrado: enséñaselo
  al usuario, porque es la única forma de detectar un envase mal configurado.
- **IVA distinto al de la ficha.** No es un error —el proveedor puede facturar a otro tipo—
  pero pregúntalo, no elijas tú.
- **El artículo tiene que llevar stock.** Si no, el servidor rechaza el documento entero. Se
  arregla marcándolo como inventariable en Glop; tú no lo marques.

## Fase 4 — Cabecera

Monta la cabecera con lo resuelto: **`TYPE_ID` es obligatorio** — `2` para un albarán y `3`
para un albarán valorado, que es lo que al cliente le llega como *factura* de su proveedor. A
efectos de stock hacen lo mismo; la diferencia es contable.

**Nunca lo omitas**: sin él no se crea nada, y un pedido (tipo 1) no movería existencias.

Añade la referencia del documento del proveedor, las fechas reales del papel, el almacén de
destino y el terminal elegido.

## Fase 5 — Simulación y cuadre

Llama a `glop_compra_documento_crear` **sin `confirmar`**. El TPV calcula el documento entero
sin escribirlo y te devuelve las cuentas ya hechas y el resultado del cuadre en tres cercos:
línea, grupo de IVA y documento, con un céntimo de tolerancia por grupo.

Si el cuadre sale mal, **no confirmes**. Busca la causa en este orden:

1. **¿Has dejado alguna línea fuera?** Entonces el total impreso ya no corresponde: ajústalo
   como dice la fase 3. Es la causa más frecuente y no es un error de lectura.
2. Un descuento de pie, portes o un cargo que no leíste.
3. Una línea del papel que se te pasó.
4. Una cifra mal leída: vuelve a esa zona de la imagen con zoom.

Plantéaselo al usuario con la línea concreta. `ACCEPT_DIFFERENCE` solo si acepta la diferencia
de forma expresa **y después de haber descartado las cuatro causas**: es la única comprobación
que detecta una línea mal leída, y apagarla por comodidad ensucia la contabilidad del cliente.

Aquí también llega el **aviso de variación del precio de compra**. Es lo más útil que hace esta
skill: enséñalo siempre, con el precio anterior y el nuevo. Que un producto haya subido y se
siga vendiendo al mismo precio es dinero que el cliente está perdiendo sin saberlo.

## Fase 6 — Resumen y confirmación

Presenta un resumen completo: proveedor (y si es alta nueva), cabecera, tabla de líneas
(artículo de Glop ↔ línea del papel, cantidad, envase, precio, descuento, IVA, total), vínculos
nuevos, totales calculados frente a los impresos, variaciones de precio y **qué stock se va a
mover**.

Pide confirmación. Sin un sí claro, no hay alta.

## Fase 7 — Alta y verificación

1. Repite la llamada con `confirmar=true`.
2. Comprueba el resultado: el servidor devuelve las existencias antes y después de cada
   artículo. Si algo no se movió, dilo.
3. Informa: *"Albarán \<serie\>/\<número\> registrado — total \<importe\>, coincide con el
   documento"*, más las líneas excluidas o cualquier discrepancia. Recuérdale que puede verlo
   en Glop, en Documentos de compra.

---

## Errores del servidor y qué significan

| Código | Qué pasó | Qué hacer |
| --- | --- | --- |
| 107 | Faltan datos obligatorios | Mira cuál y pregúntaselo al usuario |
| 110 | Un artículo no lleva stock, o no tiene ficha en ese almacén | Se marca como inventariable en Glop. No lo hagas tú |
| 114 | El tipo de IVA no existe | Consulta `glop_ivas_listar` y corrige |
| 115 | No se puede convertir la cantidad en envases a unidades | Al artículo le falta el envase o las unidades por caja. Pregunta |
| 116 | El documento no cuadra | Fase 5. No fuerces `ACCEPT_DIFFERENCE` sin permiso |
| 111 / 112 / 113 | Fallo al guardar | **No reintentes a ciegas.** Comprueba con `glop_compras_documentos_buscar` si el documento quedó registrado |

Un alta que falla **no deja el documento a medias**: o entra entero o no entra.

## Lo que esta skill no hace

- **No corrige ni borra** documentos ya registrados.
- **No toca el escandallo**: se da por hecho que el cliente lo tiene configurado.
- **No registra pedidos** (tipo 1) ni convierte documentos de un tipo a otro.
- **No maneja tallas y colores.**
- **No sube precios de venta** cuando sube el de compra: avisa, y decide el usuario.
