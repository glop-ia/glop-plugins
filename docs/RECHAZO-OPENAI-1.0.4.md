# Rechazo de OpenAI a la v1.0.4 (y hallazgos del escáner de MCP)

> Fecha: 21/10/2026. Registrado antes de borrar el plugin del dashboard de OpenAI Platform
> (plugin `plugin_asdk_app_6aa9260aeb0081919463d13190168d60`, Organización «Glop dev»), para decidir
> en la nueva solicitud (v1.0.6) cuáles de estos puntos se abordan.

## Motivo del rechazo (correo de OpenAI)

OpenAI no aprobó Glop (v1.0.4) porque **su servidor MCP no estaba conectado para la revisión**.
Presentar una versión nueva del plugin existente no lo arregla: hay que **crear un plugin nuevo**.
Pasos que piden:

1. Borrar el plugin existente en el dashboard de OpenAI Platform.
2. Volver a subir el ZIP anterior para crear un plugin nuevo y **conectar el servidor MCP durante el alta**.
3. Dar **credenciales de prueba que funcionen e instrucciones** para que su equipo pueda probarlo.
4. Enviar el plugin nuevo a revisión.

Si se cree que es un error, se puede apelar respondiendo al correo.

## Estado que tenía el plugin al borrarlo

- Revisión: **Changes required**; publicación: «Not published»; ficha de metadata y skills en 1.0.4.
- MCP `glop`: URL `https://api.glop.es/api/v1/mcp`, autenticación «Authorized», «MCP key: Not specified»,
  dominio verificado. Las 7 skills pasaron los checks.
- Todas las tools y las server instructions figuraban «Not published» (último escaneo: 01/10/2026 8:40).

## Hallazgos del escáner (6)

| Elemento | Hallazgo | Posible acción |
| --- | --- | --- |
| Server instructions | «These server instructions need further review.» (sin más detalle) | Acortar y poner lo esencial en los primeros 512 caracteres (ver `CUMPLIMIENTO-MCP-OPENAI.md`) |
| `glop_proveedor_crear` | «Appears to solicit a government-issued identifier, such as an SSN, passport number, or driver's license number.» (el campo NIF del proveedor) | Valorar redactar el campo como «identificador fiscal de la empresa (CIF/NIF)» y aclarar en la justificación que es un dato de empresa, no una persona |
| `glop_proveedor_actualizar` | El mismo hallazgo del NIF | Idem |
| `glop_mapeo_crear` | «Marked `destructiveHint: false`, but its behavior appears to cause material loss or a hard-to-reverse change.» | Poner `destructiveHint: true` (el vínculo artículo-proveedor se reutiliza en compras futuras y un error se repite) o justificarlo |
| `glop_compra_documento_crear` | (1) «Marked `openWorldHint: false`, but its behavior appears to interact with an independently controlled external system.» (2) server instructions «need further review» | Valorar `openWorldHint: true` (Glop Cloud / TPV es un sistema distinto del MCP) o justificar `false` |
| `glop_stock_documento_crear` | «This new tool definition needs further review before it can be published.» | Revisión manual; preparar justificación |

Las anotaciones están en `GlopApiRest/docs/mcp/anotaciones-justificacion-openai.md`; ahí hay que
retocar lo que se decida. Son cambios en **GlopApiRest**, no en este repo.

## Qué hay que hacer en la nueva solicitud

- Crear el plugin desde cero (ZIP de la v1.0.6) y **conectar el MCP durante el alta**.
- Aportar credenciales de prueba e instrucciones de uso para el equipo de revisión. **Las
  credenciales NO se guardan en este repo**: se escriben solo en el formulario del portal.
- Decidir los hallazgos de la tabla antes de enviar.
