# Conceptos de Glop para el agente de albaranes

> **Author: Daniel Ruiz — Date: 2026-08-17** · APPGLOP-1233
>
> Este documento está escrito **para el modelo**, no para una persona. Es el contexto que
> necesita para razonar sobre un albarán de compra sin equivocarse en lo que importa.
>
> **Actualizado el 19/08/2026** · APPGLOP-1258, APPGLOP-1260
>
> Ya **no se entrega con ninguna skill**: la skill se canceló el 18/08 (APPGLOP-1232). Lo que
> llega a todos los clientes MCP sin instalar nada es el bloque `instructions` de
> `config/mcp.php`, que es corto a propósito. Este fichero se queda como el modelo completo, y
> como el sitio donde se escriben las reglas antes de decidir cuáles caben en ese bloque.
>
> Alcance de la **fase 1** del MVP (APPGLOP-1204): el escandallo se explica pero **no se
> opera**. Se asume que el cliente ya tiene sus productos escandallados.

---

## 1. Lo único que hay que tener claro

**Un albarán de compra mueve stock.** No es un papel que se archiva: cada línea que se registra
suma existencias de un artículo concreto y actualiza su último precio de compra.

De ahí sale todo lo demás. Si una línea se empareja con el artículo equivocado, no se guarda un
dato feo: **se suma stock a un producto que no ha entrado por la puerta**, y se le cambia el
precio de compra a otro que no se ha comprado.

---

## 2. Los tres tipos de artículo

Es la distinción que más daño hace cuando se ignora.

| Tipo | Qué es | Ejemplo |
| --- | --- | --- |
| **De venta** | Lo que aparece en la carta y se cobra en el TPV | "Hamburguesa completa" |
| **Solo de compra** | Lo que entra por el almacén y no se vende tal cual. La mayoría están escandallados con uno de venta para que se descuenten | "Carne picada de ternera, kg" |
| **De compra-venta** | Lo que se compra y se vende en el mismo formato | Una lata de Coca-Cola |

**Las líneas de un albarán son artículos de compra**, no de venta. Un albarán trae "carne picada
de ternera", nunca "hamburguesa completa".

### La regla práctica

**Nunca emparejes una línea de albarán con un artículo por parecido de nombre.** El
emparejamiento se hace por las vías fiables del §5, en ese orden.

El motivo es que el vínculo **se guarda y se reutiliza**: una vez creado, todas las compras
siguientes de ese proveedor usan ese emparejamiento sin volver a preguntar (APPGLOP-1224). Un
vínculo equivocado no es un error de una vez, es un error que se repite solo.

> **Pendiente de confirmar con Joaquín**: qué campo de `tb_articulos` expresa esta distinción.
> `TIPOPRODUCTO` y `COMPUESTO` existen en la tabla pero están vacíos en los inquilinos revisados,
> y el código de GlopCloud y GlopApiRest solo los transporta, sin darles semántica. Hasta que se
> aclare, **trata esta distinción como algo conceptual, no como un filtro que puedas consultar**.

---

## 3. Escandallo (en las tablas de Glop, «composición»)

El escandallo de un artículo de venta es **de qué está compuesto**. El escandallo de una
hamburguesa son 200 g de carne de ternera, una loncha de queso y un pan.

Es lo que hace que vender una hamburguesa descuente carne del almacén.

- **Primer nivel**: lo comprado se usa tal cual. Un kilo de carne da cinco hamburguesas de 200 g.
- **Segundo nivel**: se fabrica un producto intermedio —una carne especial de ternera, pavo y
  especias— que a su vez está escandallado con sus ingredientes.
- **Combinados**: se escandallan distinto. A un JB no se le declara que lleva Coca-Cola; se
  descuentan por separado. Del JB solo se escandallan los mililitros de la botella.

Tablas: `tb_articulos_composicion` y `tb_articulos_composicion_base`.

### En fase 1 no lo toques

**No consultes ni modifiques escandallos.** Se da por hecho que están hechos. Registrar el
albarán suma stock al artículo de compra; que eso descuente correctamente al vender es
responsabilidad del escandallo que el cliente ya tiene configurado.

### La excepción que sí tienes que decir en voz alta

Si creas un artículo nuevo desde un albarán (`glop_articulo_crear`), ese artículo **nace sin
escandallo**. Entra en el almacén, pero no descontará nada al vender hasta que alguien lo
escandalle en Glop.

**Avísalo siempre al usuario.** Es un trabajo que queda pendiente fuera del MCP, y si no lo
dices, el usuario dará el albarán por terminado y el stock se irá desviando en silencio.

---

## 4. Envases de compra: cajas contra unidades

El stock se lleva **en unidades de la ficha del artículo**. Un albarán, en cambio, suele venir en
el envase con el que lo sirve el proveedor.

**La conversión la hace Glop, no tú.** Manda la cantidad tal como viene en el papel e indica el
envase; el servidor calcula las unidades (APPGLOP-1226). No conviertas cantidades por tu cuenta.

Ejemplo real de envases de un inquilino (`tb_envases_compra`):

| ID | Descripción | Medida | Unidades |
| --- | --- | --- | --- |
| 1 | UNIDAD | UD. | 1 |
| 3 | BOTELLA 0.70L | CL. | 70 |
| 5 | CAJA 24 UDS. | UDS. | 24 |
| 6 | KILOS | GR. | 1000 |
| 7 | BARRIL 50L | CL. | 5000 |

### La regla que no es obvia

**Un envase que mide contenido no multiplica el stock.**

- **2 cajas** del envase 5 (mide `UDS.`) → **48 unidades**. La caja agrupa unidades.
- **12 botellas** del envase 3 (mide `CL.`) → **12 unidades**, no 8,4 litros. El 0,70 describe
  *cuánto cabe* en la botella, no cuántas botellas hay.

Si el envase no permite calcular el factor, el servidor devuelve **error 115** y no registra
nada. Es preferible a registrar el albarán con una cantidad inventada.

---

## 5. Cómo se empareja una línea con un artículo

Usa `glop_articulo_resolver`. Prueba estas vías, en este orden:

1. **Referencia del proveedor** — la referencia que ese proveedor usa para ese artículo. Es la
   más fiable: es exactamente para esto.
2. **Referencia de la ficha** — el código interno del artículo en Glop.
3. **Código de barras** — fiable, pero **no único**: puede haber dos artículos con el mismo.
4. **Descripción** — **nunca resuelve por sí sola**, ni aunque haya un solo candidato. Sirve para
   proponerle candidatos al usuario, no para decidir.

Una resolución solo se considera **exacta** si llegó por una vía fiable **y** hay un único
candidato.

### Cuando no se resuelve

Pregunta. No elijas por parecido, no inventes y no des por bueno el candidato más probable. Las
tres salidas legítimas son:

- El usuario identifica el artículo → se crea el vínculo con `glop_mapeo_crear`, y a partir de
  ahí ese proveedor queda aprendido.
- El artículo no existe → se crea con `glop_articulo_crear`, avisando de lo del §3.
- El usuario decide dejar la línea fuera.

---

### Dónde vive la referencia de cada proveedor (APPGLOP-1268)

Un artículo puede comprarse a varios proveedores, y cada uno lo llama con **su** referencia. Esas
referencias no viven todas en el mismo sitio, y la regla es simple:

| Proveedor | Dónde va su referencia |
| --- | --- |
| El **por defecto** del artículo | En la **ficha**: `tb_articulos.REFPROVEEDOR`, campo público `PROVIDER_REF` |
| **Los demás** | En la tabla de vínculos, con `glop_mapeo_crear` |

**La del proveedor por defecto NO se duplica en la tabla.** Si lo intentas, `glop_mapeo_crear` lo
rechaza y te dice que uses `PROVIDER_REF`: escribirla en los dos sitios deja al mismo proveedor
repetido en la ficha de Glop, una vez como por defecto y otra en la lista de otros proveedores.

Por eso la resolución busca **primero el proveedor por defecto** y solo después la tabla.

---

## 6. Formatos de venta: no los confundas con los envases

Una botella de vino se puede vender entera o por copas. Eso son **formatos de venta**
(`tb_formatos_venta`, `tb_articulos_formatos_venta`): el artículo en formato botella descuenta
una botella, y el formato copa descuenta la quinta parte.

**No tienen nada que ver con los envases de compra** y en un albarán no pintan nada. Se
mencionan aquí solo para que no se mezclen: los envases son cómo *entra* la mercancía, los
formatos son cómo *sale*.

---

## 7. Lo que impone el servidor y no puedes saltarte

Estas reglas se validan en Glop, no en la conversación. Si no se cumplen, el alta **falla y no
deja nada a medias**. Conocerlas evita proponerle al usuario algo que no se puede hacer:

| Regla | Qué pasa si no se cumple |
| --- | --- |
| El artículo tiene que ser **de stock** (inventariable) para llevar referencia de proveedor o entrar en una línea | Error 110 |
| El documento **no se puede duplicar**: mismo proveedor + misma referencia del papel + mismo tipo | Se rechaza el alta |
| Las cuentas tienen que **cuadrar** con los importes impresos, con un céntimo de tolerancia por grupo de IVA | Error 116, salvo que el usuario acepte la diferencia expresamente |
| La cantidad en envases tiene que poder **convertirse a unidades** | Error 115 |
| El **tipo de documento es obligatorio**: 2 = albarán, 3 = albarán valorado (lo que al cliente le llega como factura de su proveedor) | Sin él no se crea nada |

Un pedido (tipo 1) **no mueve stock** y queda fuera de este flujo.

### Los tickets de compra también entran (APPGLOP-1258)

Un **ticket** de un establecimiento —mercado, frutería, carnicería, supermercado— se registra
igual que un albarán, y **siempre como tipo 3** (albarán valorado). No es una interpretación:
es la regla, decidida por Joaquín el 18/08/2026.

No es un caso raro. De los 13 documentos reales del primer cliente, **6 son tickets**: el 60 %
de lo que mete en su almacén no llega como albarán de proveedor.

Dos consecuencias que conviene saber:

- **Necesita un proveedor dado de alta**, igual que cualquier documento. Un ticket de mercado no
  trae proveedor en el sentido de Glop, así que habrá que crear su ficha.
- **La referencia de un tipo 3 se guarda en la columna de referencia de _factura_ de proveedor**,
  así que el número del ticket se verá ahí en el TPV.

Y una que muerde: **el control de duplicados está acotado por tipo**. El mismo ticket metido una
vez como 2 y otra como 3 entraría **dos veces**, duplicando existencias. Por eso la regla es
"ticket = siempre 3", sin excepciones.

### Códigos de IVA del papel: R1, R2, R3 (APPGLOP-1260)

Hay documentos que dan el tipo impositivo como un código en vez de como porcentaje. Son claves
de la normativa, iguales para todos los proveedores:

| Código | Tipo de IVA |
| --- | --- |
| `R1` | 4 % |
| `R2` | 10 % |
| `R3` | 21 % |

Puedes mandarlos **tal cual** en `TAX_ID`: el servidor los resuelve contra el catálogo de IVA
del sitio. Si ese cliente no tiene ese tipo configurado, el error 114 te dirá cuáles tiene —
pregúntale al usuario cuál corresponde en vez de elegir el más parecido, porque el tipo cambia
las bases del documento.

> **Pendiente de confirmar con Joaquín**: R1/R2/R3 son las claves del régimen de **recargo de
> equivalencia**, y hoy el alta escribe el recargo a cero. Si el proveedor factura con recargo,
> el documento se registrará sin él y no cuadrará.

### Antes de escribir, simula

`glop_compra_documento_crear` funciona en dos pasos: primero devuelve una previsualización con
las cuentas ya hechas **por el TPV**, y solo escribe con `confirmar=true`.

Enséñale al usuario esa previsualización y espera un sí explícito. Lo mismo para dar de alta un
proveedor, crear un artículo y crear un vínculo: **nada se escribe sin que el usuario lo diga**.

### Lo que no está en el papel: pregúntalo (APPGLOP-1262)

Tres situaciones del banco de albaranes reales en las que **preguntar no es cortesía, es la única
salida correcta**:

**1. Una línea sin descripción utilizable.** Dos tickets traen líneas del tipo
`ART 1  3,340 kg  8,90 EUR/kg`. Qué es ese artículo **no está en el papel**: solo lo sabe quien
fue a comprar. Pregúntalo. `glop_articulo_resolver` te lo recuerda en su respuesta.

**2. Correcciones manuscritas.** Un documento trae escrito a mano *«Abono: −1 Amstel Oro, −1
envase»* sobre el papel impreso. Ahí **lo impreso no es lo que se recibió**. Si ves algo escrito a
mano y no tienes claro qué corrige, pregunta: registrar lo impreso mete en el almacén una unidad
que no entró.

**3. Descartar el documento.** Siempre disponible y sin consecuencias: hasta que no llamas con
`confirmar=true` no se ha escrito nada, así que descartar es simplemente no volver a llamar. Si
una línea no se aclara y el usuario no lo sabe, descartar el documento entero es preferible a
inventarse un artículo.

> **Límite conocido, y conviene tenerlo presente.** Quien lee el papel es el modelo del cliente,
> no nosotros. El servidor **no puede garantizar** que se detecte un manuscrito ni que se
> pregunte antes de registrar: es la consecuencia asumida de mover el OCR al cliente (reunión del
> 11/08) y, sin skill, no queda ninguna red por debajo. Los dos primeros casos van anclados a la
> respuesta de la tool que los necesita, que es lo más fuerte que se puede hacer; el tercero solo
> puede vivir en el contexto.

---

### El MCP no corrige lo ya insertado

No se pueden borrar ni rectificar documentos desde aquí. Si algo entró mal, se corrige desde
Glop o desde la app. No lo intentes ni lo prometas.
