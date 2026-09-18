# Changelog

Todos los cambios relevantes del plugin Glop se anotan aquí. El formato sigue
[Keep a Changelog](https://keepachangelog.com/es-ES/1.1.0/) y el versionado,
[SemVer](https://semver.org/lang/es/).

## [Sin publicar]

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

[1.0.0]: https://github.com/glop-ia/glop-plugins/releases/tag/v1.0.0
