# Changelog

Todos los cambios relevantes del plugin Glop se anotan aquí. El formato sigue
[Keep a Changelog](https://keepachangelog.com/es-ES/1.1.0/) y el versionado,
[SemVer](https://semver.org/lang/es/).

## [Sin publicar]

### Cambiado

- Skill `informe-ventas-gestor`: pregunta si el informe se quiere en PDF, en CSV o en los
  dos. El CSV lleva una fila por documento y tipo de IVA, en formato de hoja de cálculo
  española (`;`, coma decimal, UTF-8 con BOM), sin totales ni notas para que se pueda
  importar tal cual, y se comprueba que suma el total del periodo antes de entregarlo.

## [1.0.1] - 2026-09-18

### Añadido

- Skill `cierre-del-dia`: el resumen de un día (o de un fin de semana) en un solo mensaje:
  total con y sin IVA, efectivo contra tarjeta, tickets y ticket medio, lo más vendido, hora
  fuerte, anulaciones e invitaciones y comparación con el mismo día de la semana anterior.
- Skill `informe-ventas-gestor`: informe en PDF de las ventas del periodo para el gestor, con
  `glop_informe_ventas_gestor` (IVA por tipo, simplificadas por día y serie, facturas con los
  datos del cliente y rectificativas), y el correo de envío.
- Skill `revision-carta`: análisis guiado de la carta o el surtido: productos estrella, los
  que casi no salen, qué sube y qué baja entre dos periodos y el antes y el después de un
  cambio de precio, separando el dato de la opinión.

### Cambiado

- Skill `analisis-ventas`: reescrita a partir de lo que preguntan los clientes en el chat de
  Glop. Cómo buscar las unidades de un artículo sin dar ceros falsos, franjas que pasan de
  medianoche y "hasta el cierre", negocios con varios locales (qué tools filtran por local y
  cuáles no), efectivo contra tarjeta, comparaciones del mismo tramo, medias sobre días con
  venta, proyecciones presentadas como estimación, qué hacer cuando el usuario discute una
  cifra y qué no puede responder (mesas abiertas, arqueo, precio de venta).

## [1.0.0] - 2026-09-18

Primera versión publicada.

### Añadido

- Conexión con el servidor MCP de Glop (`.mcp.json` para Claude Code, `mcp.json` para
  ChatGPT y Codex), con autenticación OAuth al instalar.
- Skill `alta-albaran-compra`: lee un albarán o una factura de proveedor desde una foto o
  un PDF, resuelve proveedor y artículos, enseña la previsualización del TPV y lo registra
  tras confirmación.
- Skill `analisis-ventas`: ventas por periodo, hora, día de la semana, empleado, familia,
  terminal, forma de pago, productos, clientes, tickets e incidencias.
- Skill `analisis-compras`: gasto por proveedor, artículos más comprados, evolución del
  precio de compra, compras frente a ventas, documentos de compra y existencias.
- Skill `informe-compras-asesor`: informe en PDF de las compras del periodo para el asesor
  o la gestoría, con el correo de envío.
- Marketplace para Claude Code (`.claude-plugin/marketplace.json`) y para ChatGPT/Codex
  (`.agents/plugins/marketplace.json`).
- Icono, logo y color de marca.
- Licencia MIT.

[1.0.1]: https://github.com/glop-ia/glop-plugins/releases/tag/v1.0.1
[1.0.0]: https://github.com/glop-ia/glop-plugins/releases/tag/v1.0.0
