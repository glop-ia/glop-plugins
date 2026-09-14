# Cumplimiento del MCP de Glop con la guía de plugins de OpenAI

> Author: Daniel Ruiz — Date: 2026-09-14
>
> Cotejo del servidor MCP de GlopApiRest (`/api/v1/mcp`) contra
> https://developers.openai.com/plugins/build/mcp-server, `build/auth` y `deploy/submission`
> (versión de la documentación leída el 14/09/2026). Lo que aquí se marca como pendiente es
> trabajo en **GlopApiRest**, no en este repo: este repo solo empaqueta.

## Resumen

| Requisito de OpenAI | Estado | Dónde |
| --- | --- | --- |
| Streamable HTTP en URL estable y pública HTTPS | ✅ | `https://api.glop.es/api/v1/mcp` (`routes/V1/mcp.php`) |
| Versión de protocolo negociada | ✅ | `NegociarVersionHandler` (2025-11-25) |
| `instructions` en el `initialize` | ✅ con matiz | `config/mcp.php`: muy largo; OpenAI pide lo importante en los **primeros 512 caracteres** |
| Nombres de tool `lowercase_underscore` + `title` | ✅ | `CatalogoTools` |
| `inputSchema` explícito por tool | ✅ | `McpServiceProvider` |
| `structuredContent` en objeto (no lista) | ✅ | APPGLOP-1236 |
| **Anotaciones `readOnlyHint` / `destructiveHint` / `openWorldHint`** | ❌ | `annotations: null` en todas las tools |
| `outputSchema` | ❌ (recomendado, no obligatorio) | ninguna tool lo declara |
| `securitySchemes` por tool | ❌ | el SDK `mcp/sdk` 0.7 no lo modela |
| Protected Resource Metadata (RFC 9728) | ✅ | `McpOAuthMetadataController::recursoProtegido` |
| `WWW-Authenticate` con `resource_metadata` en el 401 | ✅ | `McpResourceMetadata` |
| Authorization Server Metadata (RFC 8414) | ✅ parcial | falta `authorization_response_iss_parameter_supported`, `client_id_metadata_document_supported`, `registration_endpoint` |
| PKCE `S256` publicado | ✅ | `code_challenge_methods_supported` |
| Parámetro `resource` copiado al token (`aud`) | ⚠️ verificar | Passport 12: `McpClienteDedicado` lee `aud` para la licencia, pero no consta que el `resource` de la petición se valide |
| **Registro del cliente: CIMD o DCR** | ❌ | solo cliente preregistrado (client_id + secret pegados a mano) |
| UserInfo con `email` + `email_verified` (`openid`, `email`) | ❌ | solo necesario para restricción de dominio en workspaces Enterprise |
| Revocación RFC 7009 | ✅ | APPGLOP-1197 |
| Confirmación en dos fases para escrituras | ✅ | `ConfirmacionEscritura` (`confirmar=true`) |
| Elicitation | ✅ servidor | `PreguntaAlUsuario` (APPGLOP-1321); ChatGPT la admite |
| Sin secretos ni datos innecesarios en resultados | ✅ revisar | los resultados llevan IDs de negocio (`PRODUCT_ID`…), que son necesarios; revisar que no salga nada interno |
| `/.well-known/openai-apps-challenge` (verificación de dominio) | ❌ | no existe; se añade cuando se abra la solicitud en el portal |
| mTLS con la CA de OpenAI | opcional | no implementado; no bloquea |
| Skills importables desde el servidor (`io.modelcontextprotocol/skills`) | no aplica | las skills van en este paquete, se suben con el borrador |

## Lo que bloquea una publicación pública

### 1. Anotaciones de tools (bloqueante en el portal)

El portal exige `readOnlyHint`, `openWorldHint` y `destructiveHint` en **cada** tool y las
lee del servidor en **Scan Tools**; la justificación escrita no las sustituye. Hoy todas van a
`null`.

El SDK ya lo soporta (`Mcp\Schema\ToolAnnotations`, parámetro `annotations` de
`Builder::addTool`). La fuente de verdad ya existe: el flag `write` de `CatalogoTools`.
Propuesta:

| Grupo | `readOnlyHint` | `destructiveHint` | `openWorldHint` |
| --- | --- | --- | --- |
| Lectura local (`*_buscar`, `*_obtener`, `*_listar`, `glop_stock_consultar`, `glop_articulo_resolver`, `glop_compra_valorar`, `glop_diagnostico`) | `true` | `false` | `false` |
| Analítica GlopBridge (`glop_ventas_*`, `glop_compras_*`, `glop_informe_*`) | `true` | `false` | `false` |
| Altas (`glop_proveedor_crear`, `glop_articulo_crear`, `glop_mapeo_crear`) | `false` | `false` | `false` |
| Actualizaciones (`glop_proveedor_actualizar`, `glop_articulo_actualizar`, `glop_mapeo_actualizar`) | `false` | `true` (sobrescriben) | `false` |
| Documentos (`glop_compra_documento_crear`, `glop_compra_documento_convertir`, `glop_stock_documento_crear`) | `false` | `true` (mueven stock y el MCP no puede deshacerlo) | `false` |

`openWorldHint` va a `false` en todas: el servidor solo toca la cuenta del sitio autenticado,
aunque esté alojado fuera. La justificación del portal debe citar la confirmación en dos
fases como salvaguarda de las `destructiveHint: true`.

**Ojo con `glop_compra_documento_crear` sin `confirmar`**: aunque no escribe, la tool es la
misma que la que escribe, así que su anotación es la de escritura.

### 2. Registro del cliente OAuth (bloqueante de diseño)

OpenAI identifica a ChatGPT como cliente OAuth por **CIMD** (preferido), **DCR** o un
**cliente predefinido** configurado en el portal. Nuestro modelo es distinto: **cada sitio
tiene su propio `oauth_client`** (`es_mcp`), y es ese `client_id` el que identifica la
licencia (`aud` del token). Un plugin publicado tiene **un solo** cliente OAuth para todos
los usuarios, así que la licencia no puede venir del cliente: tiene que salir del **login**.

Esto no lo resuelve una anotación. Hace falta una decisión de producto y un cambio en el
servidor de autorización:

- Un cliente OAuth único para ChatGPT (CIMD con `none` o `private_key_jwt`, o DCR con
  `registration_endpoint`).
- Una pantalla de login de Glop en `/api/v1/auth/oauth/authorize` en la que el usuario se
  identifique (cuenta de app.glop.es) y **elija el sitio**; el token se emite con el `aud`
  del sitio elegido.
- `token_endpoint_auth_methods_supported` tendría que admitir `none` (CIMD público) o
  `private_key_jwt` además de `client_secret_*`.

Mientras tanto, el plugin funciona en **modo desarrollador** de ChatGPT con las credenciales
del sitio pegadas a mano (documentado en `docs/mcp/instalacion.md` de GlopApiRest), y se
puede **publicar al workspace** propio, pero no al directorio público.

### 3. Verificación de dominio

El portal pide servir un token en
`https://api.glop.es/.well-known/openai-apps-challenge` (texto plano, solo ese token). Es una
ruta nueva en `routes/wellknown.php` que lea el token de `env('OPENAI_APPS_CHALLENGE')`.

## Lo recomendable antes de enviar

- **`instructions`**: reordenar `config/mcp.php` para que la primera frase de cada regla
  esencial quede dentro de los primeros 512 caracteres. Hoy el bloque supera los 3.000.
- **`outputSchema`**: al menos en las tools que devuelven listas envueltas
  (`{ items, total }`) y en `glop_diagnostico`, que es la que el revisor va a llamar primero.
- **`authorization_response_iss_parameter_supported: true`** y devolver `iss` en la respuesta
  de autorización: con eso ChatGPT usa la redirect URI estable
  `https://chatgpt.com/connector_platform_oauth_redirect` en vez del patrón por conector
  (`redirect_patterns` de `config/mcp.php`).
- **Validar `resource`**: comprobar que Passport rechaza un token cuyo `aud` no sea
  `https://api.glop.es/api/v1/mcp` cuando la petición traiga `resource`.
- **Privacidad**: pasar los resultados de cada tool por la política de privacidad publicada.
  El portal rechaza campos de usuario no declarados (IDs internos, timestamps de sesión…).
- **Credenciales de demo** para el revisor: un sitio demo con el add-on activo, sin 2FA.
- **Cinco casos de prueba positivos y tres negativos** con la respuesta esperada. Los
  `defaultPrompt` de `plugin.json` sirven de base.

## Compatibilidad con Claude Code

El mismo paquete lleva `.claude-plugin/plugin.json` y `.mcp.json` (formato Claude) además de
`plugin.json` y `mcp.json` (Agent Plugins). OpenAI acepta también los manifiestos de Claude
como fallback, así que un único repo sirve para las dos tiendas. Las skills no mencionan a
ningún proveedor: la guía de migración de OpenAI lo exige.
