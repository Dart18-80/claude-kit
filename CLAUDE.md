# claude-kit

Marketplace de plugins de Claude Code. No es una aplicación: casi todo son archivos Markdown y JSON.

## Reglas al editar este repo

- Después de cualquier cambio, ejecuta `claude plugin validate .` desde la raíz. Debe pasar; los avisos de `version` faltante son esperados.
- El `name` de cada entrada en `.claude-plugin/marketplace.json` debe coincidir con el `name` de su `plugins/<x>/.claude-plugin/plugin.json`.
- No cambies el `name` de un plugin ya publicado, porque rompe las instalaciones existentes. Si hace falta, usa el mapa `renames` de `marketplace.json`.
- Un stack nuevo requiere tres cambios: carpeta en `plugins/`, entrada en `marketplace.json`, y tabla y reglas de detección en `plugins/kit/skills/init/SKILL.md`. Actualiza también la tabla del README.
- Si agregas contenido tomado de otro proyecto, regístralo en `THIRD_PARTY_NOTICES.md`.
- Las skills de stack se escriben en inglés. `/kit:init` y la documentación van en español.
