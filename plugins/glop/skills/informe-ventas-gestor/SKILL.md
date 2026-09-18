---
name: informe-ventas-gestor
description: >
  Usa esta skill cuando el usuario quiera preparar las ventas de su negocio en Glop para su
  gestor, asesor o contable: "prepárame las ventas del mes para el gestor", "necesito el IVA
  desglosado del trimestre para la gestoría", "haz un PDF con las ventas de agosto para el
  contable", "¿dónde saco los números del mes para mi asesor?". Reúne el informe con
  glop_informe_ventas_gestor (simplificadas por día y serie, facturas una a una con los datos
  del cliente, rectificativas y desglose por tipo de IVA), compone un PDF pensado para quien
  lleva la contabilidad y prepara el correo de envío. Si el asistente dispone de una
  herramienta de correo, ofrece enviarlo tras confirmación; si no, entrega el PDF y el texto
  del correo listos para pegar. No sustituye a los libros registro ni a la exportación
  contable de Glop.
---

# Informe de ventas para el gestor

Prepara un documento que el dueño del negocio pueda mandar a su gestor sin tener que
explicarlo: qué ha vendido en el periodo, con qué IVA, qué facturas ha emitido y a quién, y
qué rectificativas hay. Va en tres fases: **datos**, **PDF** y **envío**. Cada una se cierra
con el usuario antes de pasar a la siguiente.

Las cifras las da `glop_informe_ventas_gestor`. Tú agrupas, sumas para los totales,
compruebas que cuadran y presentas; no recalculas bases ni cuotas.

## Lo que no se negocia

1. **Nada sale del asistente sin un sí explícito del usuario.** Ni el PDF se da por bueno
   ni el correo se envía sin que haya visto el contenido y el destinatario.
2. **El informe no es un documento fiscal.** Lo dice en su portada y lo dices tú: el gestor
   contabiliza con la exportación contable de Glop o con sus propios libros; este informe
   le sirve para cuadrar y anticipar el IVA repercutido.
3. **No inventes datos del gestor.** Nombre, correo y tratamiento se preguntan o se toman
   de lo que el usuario haya dicho. Nunca se buscan fuera.
4. **Datos de clientes, solo donde hacen falta.** Las facturas llevan NIF y nombre del
   cliente porque el gestor los necesita (modelo 347). Van en el PDF, nunca en el cuerpo
   del correo.
5. Habla en el idioma del usuario. Fechas en `YYYY-MM-DD` en las tools y en formato local
   en el informe (`31/08/2026`). Importes con dos decimales y símbolo de moneda.

## Fase 1 — Datos

1. Si es la primera llamada de la conversación, `glop_diagnostico`. Si `mcp_activa` no es
   `true` o `glop_informe_ventas_gestor` está en `tools_deshabilitadas`, dilo y para.
2. **Fija el periodo.** Si no lo indica, propón el **mes natural anterior** completo y
   confírmalo. Para un trimestre, el trimestre natural. El informe de un mes de un negocio
   grande es muy largo (del orden de mil registros): para un trimestre, pide **un mes cada
   vez** y júntalos.
3. Pregunta, en un solo mensaje, lo que falte:
   - Nombre del negocio como quiere que figure (razón social o nombre comercial).
   - Nombre y correo del gestor, solo si va a haber envío.
   - Si quiere añadir el reparto por formas de pago (por defecto no).
4. Llama a `glop_informe_ventas_gestor` con `date_from` y `date_to`. Devuelve registros con
   un campo `tipo`:
   - `simplificada`: **uno por día y serie**, con `fecha`, `serie`, `num_documentos`,
     `numero_desde` y `numero_hasta`.
   - `factura`: **uno por factura**, con `fecha`, `serie`, `numero`, `id_terminal` y
     `cliente` (`id`, `cuenta_contable`, `nif`, `nombre`, `direccion`, `cp`, `localidad`,
     `provincia`).
   - `rectificativa`: uno por abono, con los importes **en negativo** y `cliente` a `null`
     si no estaba identificado.
   - `documentos_sin_lineas`: como mucho uno, con `num_documentos` e `importe` de los
     documentos emitidos que no tienen líneas.

   Cada registro trae `base`, `cuota`, `total`, a veces `recargo` (solo si hay recargo de
   equivalencia; si falta, es cero) y `ivas`, con un tramo por tipo de IVA (`porcentaje`,
   `base`, `cuota` y, si lo hay, `recargo`).
5. **Suma los totales del periodo** por clase de registro y por porcentaje de IVA, y
   comprueba que base + cuota + recargo da el total. Si algo no cuadra, dilo; no lo ajustes.
6. Si lo ha pedido, `glop_ventas_get_sales_by_payment_method` del mismo periodo, y
   comprueba que suma el total del periodo. Si no coincide al céntimo, di la diferencia en
   una nota y no la repartas: el informe corta por jornada de caja y el reparto por formas
   de pago puede no cortar los días igual.
7. Enseña al usuario el **resumen en texto** antes de montar nada: total del periodo
   (base, IVA, recargo si lo hay, total), el desglose por tipo de IVA, cuántas
   simplificadas, facturas y rectificativas hay, y cualquier cosa que llame la atención
   (una serie con saltos de numeración, una factura sin NIF, documentos sin líneas).

## Fase 2 — El PDF

Genera el PDF con la herramienta de creación de ficheros o de ejecución de código que
tenga tu entorno. Si no tienes ninguna, entrega el informe en HTML o Markdown listo para
"Imprimir → Guardar como PDF" y dilo; no prometas un PDF ni un enlace de descarga que no
puedes crear.

Nombre del fichero: `ventas-<negocio>-<YYYY-MM>.pdf` (o `<YYYY>-T<n>` para un trimestre),
sin espacios ni acentos.

### Estructura

1. **Portada**: "Informe de ventas", nombre del negocio, periodo, fecha de emisión, y la
   frase: *"Informe informativo generado desde el TPV Glop. No sustituye a los libros
   registro ni a la exportación contable."*
2. **Resumen del periodo**: base imponible, IVA repercutido, recargo de equivalencia (solo
   si lo hay) y total, en una fila por clase (simplificadas, facturas, rectificativas) y una
   fila de total.
3. **IVA por tipo**: una fila por porcentaje con base, cuota, recargo (si lo hay) y total.
   Es la tabla que más va a mirar el gestor.
4. **Facturas simplificadas**: una fila por día y serie con número de documentos, rango de
   numeración, base, cuota y total. Con muchas series, agrupa por día y deja la serie como
   subfila.
5. **Facturas**: una fila por factura con fecha, serie y número, NIF y nombre del cliente,
   base, cuota y total. Marca las que no tengan NIF.
6. **Rectificativas**: igual que las facturas, con los importes en negativo.
7. **Formas de pago** (solo si se pidió): importe y % de cada una, con la nota de cuadre si
   hace falta.
8. **Notas**, siempre:
   - El periodo se corta por la **jornada de apertura de caja**, igual que la exportación
     contable de Glop: una venta de madrugada cuenta en el día en que se abrió la caja.
   - El NIF y el nombre salen de la **ficha actual del cliente**: si se editó después de
     emitir la factura, aparece el dato nuevo.
   - Si hay `documentos_sin_lineas`: cuántos son y su importe, que no se puede repartir por
     tipo de IVA.

### Estilo

Sobrio y legible en blanco y negro: tipografía del sistema, tablas con cabecera sombreada,
importes alineados a la derecha, sin gráficos. Pie de página con "Generado con Glop ·
página X de Y". Sin logotipos que no te haya dado el usuario.

Enseña el PDF (o su vista previa) y pide el visto bueno. Un cambio de periodo obliga a
volver a la fase 1; un cambio de formato se resuelve aquí.

## Fase 3 — Envío al gestor

1. **Mira qué herramientas de correo tienes** en esta conversación: un conector de Gmail,
   Outlook o Microsoft 365, un servicio de envío, o cualquier tool cuyo nombre o descripción
   hable de enviar correo. No des por hecho que existe ni que no existe.
2. **Si la tienes**: redacta el correo, enséñalo entero (destinatario, asunto, cuerpo y
   nombre del adjunto) y pide un sí explícito. Solo entonces envíalo con el PDF adjunto y,
   si el conector lo permite, con copia al usuario. Confirma con lo que devuelva la
   herramienta; si falla, dilo y pasa al punto 3.
3. **Si no la tienes**: entrega el PDF y, en un bloque aparte, el asunto y el cuerpo listos
   para copiar, y explica en una línea que tiene que adjuntar el fichero desde su correo.

### Plantilla del correo

- **Asunto**: `Ventas <negocio> · <periodo>`
- **Cuerpo**: saludo con el nombre del gestor; una frase con el periodo y el total (base,
  IVA, total); una frase que diga que el desglose por tipo de IVA, las simplificadas, las
  facturas y las rectificativas van en el PDF adjunto, generado desde el TPV Glop; una
  frase que recuerde que es informativo y que la contabilización va con la exportación
  contable; despedida con el nombre del usuario.

Sin tablas, sin cifras por cliente y sin NIF en el cuerpo: eso va en el adjunto.

## Lo que esta skill no hace

- No genera la exportación contable de Glop ni ficheros para programas de contabilidad.
- No emite libros registro de facturas expedidas ni modelos tributarios (303, 347…).
- No responde preguntas sueltas sobre ventas (`analisis-ventas`) ni prepara el informe de
  compras (`informe-compras-asesor`).
- No busca el correo del gestor en ningún sitio ni envía nada sin confirmación.
