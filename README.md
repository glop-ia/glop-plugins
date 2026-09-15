# glop-plugins

Marketplace y plugins oficiales de Glop para ChatGPT, Codex y Claude Code.

Un plugin empaqueta el **servidor MCP de Glop** (`https://api.glop.es/api/v1/mcp`, vive en
GlopApiRest) junto con **skills** que enseñan al modelo a usar sus tools en flujos concretos.
Este repo no contiene código del servidor: solo el paquete instalable.

## Estructura

```
glop-plugins/
├── .agents/plugins/marketplace.json   # marketplace (formato Agent Plugins / OpenAI)
├── .claude-plugin/marketplace.json    # el mismo marketplace, formato Claude Code
├── docs/CUMPLIMIENTO-MCP-OPENAI.md    # cotejo del MCP contra la guía de OpenAI
└── plugins/
    └── glop/
        ├── plugin.json                # manifiesto portable (schema agent-plugins.org 1.0.0)
        ├── mcp.json                   # servidor MCP remoto (streamable-http)
        ├── .claude-plugin/plugin.json # compatibilidad Claude Code
        ├── .mcp.json                  # compatibilidad Claude Code
        ├── assets/                    # icono y logo (pendientes)
        └── skills/
            ├── alta-albaran-compra/   # foto/PDF de albarán → documento de compra en Glop
            ├── analisis-ventas/       # preguntas sobre ventas (glop_ventas_*)
            └── analisis-compras/      # gasto, precios de proveedor y stock (glop_compras_*)
```

Cada skill lleva `SKILL.md` (frontmatter `name` + `description`, el `name` coincide con la
carpeta) y `agents/openai.yaml` con la dependencia del MCP.

## Probar en local

### ChatGPT (modo desarrollador) / Codex

1. En ChatGPT: **Settings → Security and login → Developer mode**.
2. En https://chatgpt.com/plugins, botón **+**, URL `https://api.glop.es/api/v1/mcp`,
   autenticación OAuth con el *Client ID* y *Client Secret* del sitio.
3. Añadir este repo como marketplace:
   ```bash
   codex plugin marketplace add ./ruta/a/glop-plugins
   ```
   o abrir el repo en la app de escritorio de ChatGPT: lee `.agents/plugins/marketplace.json`
   y muestra el plugin **Glop** en el directorio bajo el origen local.
4. Instalarlo, abrir un chat en modo **Work** y escribir `@Glop`.

### Claude Code

```bash
claude plugin marketplace add ./ruta/a/glop-plugins
claude plugin install glop@glop
```

## Publicar

- **Workspace de ChatGPT**: desde https://chatgpt.com/plugins → Personal → Publish (solo
  admins). No sale al directorio público.
- **Directorio público** (ChatGPT + Codex): https://platform.openai.com/plugins → Create
  plugin → **With MCP**, con la URL de producción y las skills de `plugins/glop/skills/`.
  Antes hay que cerrar los bloqueantes de `docs/CUMPLIMIENTO-MCP-OPENAI.md`
  (anotaciones de tools, registro de cliente OAuth, verificación de dominio).

## Mantener las skills

Las skills dicen **en qué orden** hacer las cosas y **cómo hablar con el usuario**. Las reglas
de negocio viven en el servidor (descripciones y respuestas de las tools): no se duplican
aquí, para que al cambiar una tool no haya un segundo texto que barrer. `alta-albaran-compra`
sustituye a la versión larga de `docs/mcp/skill-alta-albaran/` de GlopApiRest.
