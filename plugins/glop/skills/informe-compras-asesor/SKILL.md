---
name: informe-compras-asesor
description: >
  Usa esta skill cuando el usuario quiera preparar un informe de sus compras a proveedores en
  Glop para enviárselo a su asesor, gestor o contable: "prepárame el informe de compras del
  mes para mi asesor", "necesito mandar a la gestoría lo que he comprado este trimestre",
  "haz un PDF con las compras para el contable". Reúne las cifras con las tools glop_compras_*,
  compone un informe en PDF pensado para quien lleva la contabilidad (resumen, gasto por
  proveedor con base e IVA, artículos principales y limitaciones de los datos) y prepara el
  correo de envío. Si el asistente dispone de una herramienta de correo, ofrece enviarlo
  desde ahí tras confirmación; si no, entrega el PDF y el texto del correo listos para pegar.
  No sustituye a las facturas de proveedor ni a los libros registro.
---

# Informe de compras para el asesor

Prepara un documento que el dueño del negocio pueda mandar a su asesor sin tener que
explicarlo: qué ha comprado, a quién, cuánto ha sido base y cuánto IVA, y qué fiabilidad
tienen esas cifras. Va en tres fases: **datos**, **PDF** y **envío**. Cada una se cierra con
el usuario antes de pasar a la siguiente.

Las cifras las calculan las tools de Glop. Tú no sumas ni recalculas nada: ordenas,
presentas y explicas.

## Lo que no se negocia

1. **Nada sale del asistente sin un sí explícito del usuario.** Ni el PDF se da por bueno
   ni el correo se envía sin que el usuario haya visto el contenido y el destinatario.
2. **El informe no es un documento fiscal.** Lo dice en su portada y lo dices tú: el asesor
   contabiliza con las facturas de proveedor, y este informe le sirve para cuadrar,
   anticipar el IVA soportado y detectar facturas que faltan.
3. **No inventes datos del asesor.** Nombre, correo y tratamiento se preguntan o se toman de
   lo que el usuario haya dicho. Nunca se buscan en fuentes externas.
4. **Manda lo justo.** Al asesor le va el PDF. No pegues las tablas en el cuerpo del correo
   ni añadas datos que no se le hayan enseñado al usuario.
5. Habla en el idioma del usuario. Fechas en `YYYY-MM-DD` en las tools y en formato local
   en el informe (`31/08/2026`). Importes con dos decimales y símbolo de moneda.

## Fase 1 — Datos

1. Si es la primera llamada de la conversación, `glop_diagnostico`. Si `mcp_activa` no es
   `true` o alguna tool `glop_compras_*` está en `tools_deshabilitadas`, dilo y para.
2. **Fija el periodo.** Si el usuario no lo indica, propón el **mes natural anterior**
   completo (es lo que suele pedir un asesor) y confírmalo antes de seguir. Para un
   trimestre, usa el trimestre natural. El rango máximo es de 365 días.
3. Pregunta, en un solo mensaje, lo que falte:
   - Nombre del negocio como quiere que figure (razón social o nombre comercial).
   - Nombre del asesor y su correo, solo si va a haber envío.
   - Si quiere incluir la sección de artículos principales (por defecto sí) y la de compras
     frente a ventas (por defecto **no**: tarda más de un minuto en sitios grandes y su
     lectura exige matices).
4. Llama a las tools, todas con el mismo `from` y `to`:
   - `glop_compras_get_purchases_by_supplier` con `limit` 100. Trae por proveedor
     `total_base`, `total_tax`, `total`, `num_documents`, primera y última compra y
     `share_percentage`, y en `period_totals` los totales del periodo. Es la columna
     vertebral del informe.
   - `glop_compras_get_top_purchased_products` con `limit` 15, si el usuario la quiere.
   - `glop_compras_compare_purchases_vs_sales` con `granularity` `month`, solo si la ha
     pedido expresamente. Guarda su `disclaimer` tal cual.
5. Anota `document_types_included` de cada respuesta: por defecto cuenta albaranes (tipo 2)
   y facturas de proveedor (tipo 3), y los pedidos quedan fuera. El informe lo dirá.
6. Enseña al usuario el **resumen en texto** antes de montar nada: total del periodo (base,
   IVA, total), número de documentos, los cinco proveedores con más peso y cualquier cosa
   que llame la atención (un proveedor con un solo documento y mucho importe, un mes sin
   compras, un total que no se parece al que él espera). Si algo no le cuadra, la primera
   causa es siempre la de la sección "Limitaciones": lo tecleado directamente en el TPV no
   está aquí.

## Fase 2 — El PDF

Genera el PDF con la herramienta de creación de ficheros o de ejecución de código que
tenga tu entorno. Si no tienes ninguna, entrega el informe en HTML o Markdown listo para
"Imprimir → Guardar como PDF" y dilo con claridad; no prometas un PDF que no puedes crear.

Nombre del fichero: `compras-<negocio>-<YYYY-MM>.pdf` (o `<YYYY>-T<n>` para un trimestre),
sin espacios ni acentos.

### Estructura

1. **Portada**: "Informe de compras a proveedores", nombre del negocio, periodo, fecha de
   emisión, y la frase: *"Informe informativo generado desde el TPV Glop. No sustituye a las
   facturas de proveedor ni a los libros registro."*
2. **Resumen del periodo**: base imponible, IVA soportado, total, número de documentos,
   número de proveedores, tipos de documento incluidos. En una tabla de una fila o en
   tarjetas.
3. **Gasto por proveedor**: tabla ordenada por gasto con proveedor, número de documentos,
   base, IVA, total, peso sobre el periodo, primera y última compra. Fila final de totales
   que debe coincidir con `period_totals`; si la tool devolviera menos proveedores que los
   que existen, dilo en una nota al pie en lugar de forzar el cuadre.
4. **Artículos principales** (opcional): descripción, cantidad, gasto sin IVA, precio medio
   ponderado, precio de la última compra, proveedores que lo sirven.
5. **Compras frente a ventas** (solo si se pidió): la tabla mensual de la tool y su
   `disclaimer` literal debajo. No lo llames margen.
6. **Limitaciones de los datos**, siempre, con este contenido:
   - Solo recoge las compras registradas por los canales de Glop (módulo de Compras, este
     asistente y el chat de la app). Lo tecleado directamente en el TPV no aparece.
   - Un documento editado o borrado en el TPV después de registrarse no se actualiza aquí.
   - Los abonos figuran como importes negativos dentro de su documento.
   - Un albarán (tipo 2) es mercancía recibida que puede no estar facturada todavía; el
     asesor debe cotejar con las facturas reales.
   - El IVA se agrupa por el tipo aplicado en cada línea.

### Estilo

Sobrio y legible en blanco y negro: tipografía del sistema, tablas con cabecera sombreada,
importes alineados a la derecha, sin gráficos que no aporten. Pie de página con "Generado
con Glop · página X de Y". Sin logotipos que no te haya dado el usuario.

Enseña el PDF (o su vista previa) y pide el visto bueno. Un cambio de periodo obliga a
volver a la fase 1; un cambio de formato se resuelve aquí.

## Fase 3 — Envío al asesor

1. **Mira qué herramientas de correo tienes disponibles** en esta conversación: un
   conector de Gmail, Outlook o Microsoft 365, un servicio de envío como Brevo o SendGrid, o
   cualquier tool cuyo nombre o descripción hable de enviar correo. No des por hecho que
   existe ni que no existe: compruébalo en tu lista de herramientas.
2. **Si la tienes**: redacta el correo, enséñalo entero al usuario (destinatario, asunto,
   cuerpo y nombre del adjunto) y pide un sí explícito. Solo entonces envíalo, con el PDF
   adjunto y, si el conector lo permite, con copia al propio usuario. Confirma después con
   lo que devuelva la herramienta; si falla, dilo y pasa al punto 3.
3. **Si no la tienes**: entrega el PDF y, en un bloque aparte, el asunto y el cuerpo del
   correo listos para copiar, y explica en una línea que tiene que adjuntar el fichero desde
   su cliente de correo. No intentes enviar por otros medios.
4. Si el asistente dispone de una tool de notificación pero no de correo (un aviso al
   propio usuario, un mensaje en un canal), úsala solo para avisarle de que el informe está
   listo, nunca para hacer llegar el PDF al asesor.

### Plantilla del correo

- **Asunto**: `Compras a proveedores <negocio> · <periodo>`
- **Cuerpo**: saludo con el nombre del asesor; una frase con el periodo y el total (base,
  IVA, total); una frase que diga que el detalle por proveedor va en el PDF adjunto y que se
  ha generado desde el TPV Glop a partir de los albaranes y facturas registrados; una
  frase que recuerde que no sustituye a las facturas y que agradecerá aviso si detecta
  alguna que falte; despedida con el nombre del usuario.

Sin tablas ni cifras por proveedor en el cuerpo: eso va en el adjunto.

## Lo que esta skill no hace

- No registra ni modifica documentos de compra (`alta-albaran-compra`).
- No responde preguntas puntuales sobre compras (`analisis-compras`).
- No emite libros registro de facturas recibidas ni modelos tributarios.
- No busca el correo del asesor en ningún sitio ni envía nada sin confirmación.
