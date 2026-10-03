# claude-kit

Mi entorno de trabajo para [Claude Code](https://code.claude.com), empaquetado como un **marketplace de plugins**.

Lo instalo una sola vez. Después, en cada proyecto, un comando activa solo lo que ese proyecto necesita: patrones del stack, agentes revisores y el flujo plan → TDD → revisión → verificación.

```
/kit:init nestjs prisma postgres redis
```

## Por qué existe

Los paquetes "todo en uno" para agentes de código traen cientos de skills. La mayoría no aplica a cada proyecto, y cada una ocupa espacio en el contexto. Este kit sigue tres reglas:

- **Por proyecto, no global:** los plugins de stack se activan con `--scope project`. Quedan anotados en el `.claude/settings.json` del repo y viajan con él.
- **Pocas piezas, bien elegidas:** un proyecto típico activa 3 o 4 plugins de stack, que suman entre 0.6k y 1.5k tokens fijos por sesión.
- **Flujo de trabajo con plugins oficiales:** la planeación, el TDD, la revisión y la seguridad vienen de plugins mantenidos en el marketplace oficial de Anthropic, no de copias propias.

## Qué incluye

### Plugins de este marketplace

**Instalador**

| Plugin | Contenido | Costo fijo aprox. |
|---|---|---|
| `kit` | Comando `/kit:init` y plantilla de `CLAUDE.md` | ~180 tok |

**JavaScript / TypeScript**

| Plugin | Contenido | Costo fijo aprox. |
|---|---|---|
| `stack-typescript` | Agente `typescript-reviewer` | ~100 tok |
| `stack-nestjs` | Skills `nestjs-patterns`, `backend-patterns`, `api-design` | ~290 tok |
| `stack-prisma` | Skills `prisma-patterns`, `database-migrations` | ~310 tok |
| `stack-nextjs` | Skills `nextjs-turbopack`, `react-patterns`, `react-testing` | ~360 tok |
| `stack-react-native` | Skill `react-native-patterns` | ~130 tok |

**Python**

| Plugin | Contenido | Costo fijo aprox. |
|---|---|---|
| `stack-python` | Skills `python-patterns`, `python-testing` y agente `python-reviewer` | ~250 tok |
| `stack-fastapi` | Skill `fastapi-patterns` y agente `fastapi-reviewer` | ~190 tok |
| `stack-django` | Skills `django-patterns`, `django-tdd`, `django-security`, `django-verification` y agentes `django-reviewer`, `django-build-resolver` | ~570 tok |

**Java**

| Plugin | Contenido | Costo fijo aprox. |
|---|---|---|
| `stack-java` | Skill `java-coding-standards` y agentes `java-reviewer`, `java-build-resolver` | ~340 tok |
| `stack-springboot` | Skills `springboot-patterns`, `springboot-tdd`, `springboot-security`, `springboot-verification`, `jpa-patterns` | ~560 tok |

**Datos (cualquier lenguaje)**

| Plugin | Contenido | Costo fijo aprox. |
|---|---|---|
| `stack-postgres` | Skill `postgres-patterns` y agente `database-reviewer` | ~180 tok |
| `stack-redis` | Skills `redis-patterns` y `bullmq-patterns` (colas en NestJS) | ~260 tok |

**Inteligencia artificial**

| Plugin | Para qué | Contenido | Costo fijo aprox. |
|---|---|---|---|
| `stack-llm` | Integrar LLMs en aplicaciones | Skills `llm-app-patterns`, `llm-evals`, `cost-aware-llm-pipeline`, `regex-vs-llm-structured-text` | ~580 tok |
| `stack-agents` | Construir agentes | Skills `agent-development`, `agent-harness-construction`, `loop-design-check`, `agent-architecture-audit`, `mcp-server-patterns` | ~720 tok |
| `stack-rag` | Búsqueda y Q&A sobre tus datos | Skill `rag-patterns` y agente `rag-pipeline-reviewer` | ~300 tok |
| `stack-ml` | Desarrollar modelos (ML/deep learning) | Skills `mle-workflow`, `pytorch-patterns`, `ml-adoption-playbook` y agentes `mle-reviewer`, `pytorch-build-resolver` | ~510 tok |

El costo fijo corresponde a lo que el plugin agrega a cada sesión: los nombres y las descripciones. El contenido completo de una skill se carga solo cuando se usa.

### Plugins base (marketplace oficial de Anthropic)

`/kit:init` también los instala en el proyecto, salvo que se use `--sin-base`:

| Plugin | Para qué |
|---|---|
| [`superpowers`](https://github.com/obra/superpowers) | Brainstorming, planes, TDD, debugging sistemático y verificación antes de dar algo por terminado |
| `pr-review-toolkit` | Agentes de revisión: tests, errores silenciosos, diseño de tipos, comentarios y calidad |
| `security-guidance` | Avisos de seguridad mientras se edita y revisión del diff |
| `commit-commands` | `/commit`, `/commit-push-pr` |

Para proyectos de IA, `/kit:init` agrega además estos plugins oficiales cuando aplican:

| Plugin | Cuándo |
|---|---|
| `agent-sdk-dev` | El proyecto usa el Claude Agent SDK |
| `mcp-server-dev` | El proyecto construye un servidor MCP |

## Instalación (una sola vez)

En una terminal con Claude Code instalado:

```bash
claude plugin marketplace add Dart18-80/claude-kit
claude plugin install kit@claude-kit
```

O dentro de una sesión de Claude Code (CLI o VS Code):

```
/plugin marketplace add Dart18-80/claude-kit
/plugin install kit@claude-kit
```

Te recomiendo activar las actualizaciones automáticas en `/plugin` → **Marketplaces** → `claude-kit` → **Enable auto-update**. Si no lo haces, actualiza a mano con `/plugin marketplace update claude-kit`.

## Uso

Abre Claude Code **en la raíz del proyecto** y ejecuta:

```
/kit:init                          # detecta el stack (package.json, pyproject.toml, pom.xml, build.gradle…) y pregunta antes de instalar
/kit:init nestjs prisma postgres   # stacks explícitos
/kit:init fastapi rag postgres     # API en Python con RAG sobre pgvector
/kit:init springboot postgres      # backend en Java
/kit:init agents                   # proyecto de agentes de IA
/kit:init nextjs --sin-base        # sin los plugins base
/kit:init --dry-run                # solo muestra el plan, sin instalar nada
```

El comando hace cuatro cosas:

1. Registra este marketplace en el proyecto.
2. Instala los plugins de stack y los plugins base con `--scope project`.
3. Crea un `CLAUDE.md` con el nombre, el stack, los comandos reales del proyecto (scripts de npm, pytest/ruff, Maven/Gradle) y el flujo de trabajo. Si ya existe uno, solo propone lo que falta.
4. Termina con un resumen de lo que hizo.

Después:

1. Ejecuta `/reload-plugins` para activar los plugins en la sesión actual.
2. Haz commit de `.claude/settings.json` y `CLAUDE.md`.

Para agregar un stack más adelante, vuelve a ejecutar `/kit:init <stack>`. Solo instala lo que falte.

**Stacks válidos:** `typescript`, `nestjs`, `prisma`, `nextjs` (alias `react`), `react-native` (alias `expo`), `python`, `fastapi`, `django`, `java`, `springboot` (alias `spring`), `postgres` (alias `supabase`), `redis` (alias `bullmq`), `llm` (alias `ai`), `agents` (alias `mcp`), `rag`, `ml` (alias `pytorch`).

Algunos stacks traen otros automáticamente: los de TypeScript agregan `typescript`, `fastapi`/`django`/`ml` agregan `python`, `springboot` agrega `java`, y `agents`/`rag` agregan `llm`.

## Estructura del repositorio

```
claude-kit/
├── .claude-plugin/marketplace.json   # catálogo de plugins
└── plugins/
    ├── kit/
    │   └── skills/init/
    │       ├── SKILL.md              # lógica de /kit:init
    │       └── templates/CLAUDE.md   # plantilla por proyecto
    └── stack-*/
        ├── .claude-plugin/plugin.json
        ├── skills/<skill>/SKILL.md
        └── agents/<agente>.md
```

## Mantenimiento

**Agregar un stack nuevo:**

1. Crea `plugins/stack-<nombre>/.claude-plugin/plugin.json`, más sus `skills/` y `agents/`.
2. Agrega la entrada en `.claude-plugin/marketplace.json`.
3. Agrega el stack y sus reglas de detección en `plugins/kit/skills/init/SKILL.md`.
4. Valida con `claude plugin validate .`

**Probar cambios localmente antes de hacer push:**

```bash
claude plugin marketplace add ./ruta/a/claude-kit   # marketplace local, se lee en vivo
```

**Versiones:** los plugins no declaran `version` a propósito, así cada commit en `main` es una nueva versión. Para fijar versiones estables, agrega `"version"` en cada `plugin.json` y súbela en cada release.

## Créditos

La mayoría de las skills de stack son adaptaciones de [ECC](https://github.com/affaan-m/ECC) (MIT), con atribución en [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md). Son originales de este repositorio: `bullmq-patterns`, `llm-app-patterns`, `llm-evals`, `rag-patterns`, `agent-development`, el comando `/kit:init` y la plantilla de `CLAUDE.md`.

## Licencia

MIT
