---
name: init
description: Configura el proyecto actual con claude-kit — instala a nivel de proyecto los plugins de stack (typescript, nestjs, prisma, postgres, redis, nextjs, react-native, python, fastapi, django, java, springboot, llm, agents, rag, ml) y los plugins base (superpowers, pr-review-toolkit, security-guidance, commit-commands), y crea el CLAUDE.md del proyecto. Úsalo solo cuando el usuario ejecute /kit:init.
argument-hint: "[stacks...] [--sin-base] [--dry-run]   ej: nestjs prisma postgres | fastapi rag | springboot | agents"
disable-model-invocation: true
allowed-tools: [Read, Glob, Grep, Write, Edit, Bash]
---

# /kit:init — configurar este proyecto con claude-kit

Argumentos del usuario: `$ARGUMENTS`

Objetivo: dejar el proyecto actual listo para trabajar, registrando los plugins **a nivel de proyecto** (`--scope project`). Así quedan anotados en `.claude/settings.json`, que se versiona con el repo, y no afectan a otros proyectos.

Responde al usuario en español y sé breve: muestra el plan, ejecútalo y muestra el resumen.

## Constantes

- Marketplace del kit: `claude-kit`, en GitHub `Dart18-80/claude-kit`
- Marketplace oficial: `claude-plugins-official`, en GitHub `anthropics/claude-plugins-official`
- Plugins base (oficiales): `superpowers`, `pr-review-toolkit`, `security-guidance`, `commit-commands`
- Plugins oficiales extra para IA (se agregan solo si aplica, ver Paso 2):
  - `agent-sdk-dev`: si el proyecto usa el Claude Agent SDK (`@anthropic-ai/claude-agent-sdk` o `claude-agent-sdk`)
  - `mcp-server-dev`: si el proyecto construye un servidor MCP (`@modelcontextprotocol/sdk`, `mcp` o `fastmcp`) o el usuario lo pide

| Stack (argumento) | Alias aceptados | Plugin | Ecosistema |
|---|---|---|---|
| `typescript` | `ts` | `stack-typescript` | JS/TS |
| `nestjs` | `nest` | `stack-nestjs` | JS/TS |
| `prisma` | — | `stack-prisma` | JS/TS |
| `nextjs` | `next`, `react` | `stack-nextjs` | JS/TS |
| `react-native` | `rn`, `expo` | `stack-react-native` | JS/TS |
| `python` | `py` | `stack-python` | Python |
| `fastapi` | — | `stack-fastapi` | Python |
| `django` | `drf` | `stack-django` | Python |
| `java` | — | `stack-java` | Java |
| `springboot` | `spring`, `spring-boot` | `stack-springboot` | Java |
| `postgres` | `postgresql`, `pg`, `supabase` | `stack-postgres` | cualquiera |
| `redis` | `bull`, `bullmq` | `stack-redis` | cualquiera |
| `llm` | `ai`, `ia`, `genai` | `stack-llm` | IA (cualquiera) |
| `agents` | `agentes`, `agent`, `mcp` | `stack-agents` | IA (cualquiera) |
| `rag` | `vector`, `embeddings` | `stack-rag` | IA (cualquiera) |
| `ml` | `pytorch`, `machine-learning`, `deep-learning` | `stack-ml` | IA (Python) |

**Dependencias implícitas.** Agrégalas siempre, aunque el usuario no las pida:

- `nestjs`, `prisma`, `nextjs` o `react-native` → agrega `typescript`
- `fastapi` o `django` → agrega `python`
- `springboot` → agrega `java`
- `agents` o `rag` → agrega `llm`
- `ml` → agrega `python`

## Paso 1 — Ubicar el proyecto

1. Ejecuta `git rev-parse --show-toplevel`. Si responde, esa es la **raíz del proyecto**. Si falla, la raíz es el directorio actual. En ese caso avisa que el proyecto no es un repo git y que conviene ejecutar `git init` para versionar la configuración. Si el usuario quiere seguir, sigue.
2. Si el directorio actual no es la raíz del proyecto, avisa y pregunta si quiere configurar la raíz o el subdirectorio. Los plugins de proyecto se leen desde el directorio donde se abre Claude Code.
3. Todos los comandos siguientes se ejecutan **desde la raíz elegida**.

## Paso 2 — Decidir los stacks

**Si hay argumentos**, conviértelos a plugins con la tabla, sin distinguir mayúsculas. Ignora `--sin-base` y `--dry-run`, que son banderas. Si un argumento no está en la tabla, avisa y muestra la lista válida.

**Si no hay stacks en los argumentos**, detéctalos. Busca los archivos de manifiesto en la raíz, en `apps/*`, en `packages/*` y en las carpetas de primer nivel (por ejemplo `backend/`, `api/`, `frontend/`, `server/`). Así se cubren los monorepos y los proyectos full-stack con varios lenguajes.

- **JS/TS** (`package.json`):
  - `@nestjs/core` → `nestjs`
  - `prisma` o `@prisma/client`, o un archivo `schema.prisma` → `prisma`
  - `next` → `nextjs`
  - `react-native` o `expo` → `react-native`
  - `typescript` en dependencias, o un `tsconfig.json` → `typescript`
- **Python** (`pyproject.toml`, `requirements*.txt`, `setup.py`, `Pipfile`):
  - cualquiera de esos archivos → `python`
  - dependencia `fastapi` → `fastapi`
  - dependencia `django` o `djangorestframework`, o un archivo `manage.py` → `django`
- **Java** (`pom.xml`, `build.gradle`, `build.gradle.kts`):
  - cualquiera de esos archivos → `java`
  - `spring-boot` en el archivo de build (parent `spring-boot-starter-parent`, plugin `org.springframework.boot` o dependencias `spring-boot-starter-*`) → `springboot`
- **Bases de datos y colas** (en cualquiera de los manifiestos anteriores):
  - `pg`, `postgres`, `@supabase/supabase-js`, `psycopg`, `psycopg2`, `asyncpg`, `org.postgresql:postgresql`, o `provider = "postgresql"` en `schema.prisma` → `postgres`
  - `ioredis`, `redis`, `bullmq`, `bull`, `@nestjs/bullmq`, `@nestjs/bull`, `redis-py`, `spring-boot-starter-data-redis` → `redis`
- **IA** (en cualquiera de los manifiestos anteriores):
  - SDKs de modelos: `openai`, `@anthropic-ai/sdk`, `anthropic`, `@google/genai`, `google-genai`, `ai` (Vercel AI SDK), `@ai-sdk/*`, `litellm`, `ollama`, `langchain*`, `@langchain/*`, `spring-ai-*` → `llm`
  - Agentes: `@anthropic-ai/claude-agent-sdk`, `claude-agent-sdk`, `@openai/agents`, `openai-agents`, `langgraph`, `@langchain/langgraph`, `pydantic-ai`, `crewai`, `autogen*`, `@modelcontextprotocol/sdk`, `mcp`, `fastmcp` → `agents`
  - RAG: `pgvector`, `llama-index*`, `llamaindex`, `chromadb`, `qdrant-client`, `@qdrant/*`, `@pinecone-database/pinecone`, `pinecone`, `weaviate*`, `faiss-cpu`, `faiss-gpu`, `sentence-transformers`, o `CREATE EXTENSION vector` en migraciones → `rag`
  - ML: `torch`, `tensorflow`, `keras`, `jax`, `scikit-learn`, `xgboost`, `lightgbm`, `transformers`, `mlflow` → `ml`

Muestra lo detectado, indicando en qué archivo encontraste cada cosa, y **pregunta antes de instalar**. Si no encuentras ningún manifiesto, pregunta qué stack usará el proyecto.

Al final aplica las dependencias implícitas y decide si corresponden los plugins oficiales extra para IA.

## Paso 3 — Mostrar el plan

Antes de ejecutar nada, muestra:

- la raíz del proyecto,
- los plugins de stack que se instalarán,
- los plugins base que se instalarán (salvo que venga `--sin-base`),
- los plugins oficiales extra para IA, si aplican,
- si se creará o se actualizará `CLAUDE.md`.

Si viene `--dry-run`, termina aquí.

## Paso 4 — Instalar (desde la raíz del proyecto)

Ejecuta los comandos uno por uno con Bash y revisa la salida de cada uno.

```bash
# 1. Registrar el marketplace del kit para este proyecto
claude plugin marketplace add Dart18-80/claude-kit --scope project

# 2. Plugins de stack (uno por cada stack elegido)
claude plugin install stack-<nombre>@claude-kit --scope project

# 3. Plugins base (omitir con --sin-base)
claude plugin install superpowers@claude-plugins-official --scope project
claude plugin install pr-review-toolkit@claude-plugins-official --scope project
claude plugin install security-guidance@claude-plugins-official --scope project
claude plugin install commit-commands@claude-plugins-official --scope project

# 4. Extras de IA (solo si aplican)
claude plugin install agent-sdk-dev@claude-plugins-official --scope project
claude plugin install mcp-server-dev@claude-plugins-official --scope project
```

Manejo de errores:

- **El marketplace oficial no está registrado:** ejecuta `claude plugin marketplace add anthropics/claude-plugins-official` y reintenta.
- **El marketplace `claude-kit` ya existe:** no es un error. Continúa.
- **Un plugin ya está instalado:** continúa con el siguiente.
- **`claude` no se encuentra en el PATH:** detente y muestra los comandos para que el usuario los ejecute en su terminal.
- Cualquier otro error: muéstralo tal cual, sigue con el resto y lístalo al final.

## Paso 5 — CLAUDE.md

Busca `CLAUDE.md` en la raíz del proyecto.

**Si no existe**, créalo desde la plantilla `templates/CLAUDE.md`, ubicada en el directorio base de esta skill. Rellena los marcadores `{{...}}` con datos reales del proyecto. Si el proyecto tiene varios ecosistemas (por ejemplo, un backend en Python y un frontend en Next.js), arma cada sección por partes, con un subtítulo por carpeta.

- `{{NOMBRE}}`: el nombre del proyecto según el manifiesto (`name` de `package.json`, `[project].name` de `pyproject.toml`, `artifactId` de `pom.xml`, `rootProject.name` de `settings.gradle`). Si no hay manifiesto, usa el nombre de la carpeta.
- `{{DESCRIPCION}}`: la descripción del manifiesto. Si no hay, pregunta al usuario en una línea. Si el usuario no responde, deja `TODO: describir el proyecto`.
- `{{STACK}}`: lista con viñetas de los stacks elegidos, con sus versiones principales tomadas de los manifiestos (también la versión de Python o de Java, si está declarada).
- `{{GESTOR}}`: el gestor de dependencias o de build, una línea por ecosistema:
  - **JS/TS:** según el lockfile: `pnpm-lock.yaml` → pnpm, `yarn.lock` → yarn, `bun.lockb` o `bun.lock` → bun, `package-lock.json` → npm. Por defecto, pnpm.
  - **Python:** `uv.lock` → uv, `poetry.lock` → Poetry, `Pipfile.lock` → Pipenv. Si no hay ninguno, pip con un entorno virtual (`.venv`).
  - **Java:** si existe `mvnw` → Maven wrapper (`./mvnw`); si existe `gradlew` → Gradle wrapper (`./gradlew`). Si no hay wrapper, `mvn` o `gradle` según el archivo de build.
- `{{COMANDOS}}`: solo comandos que existan de verdad en el proyecto, uno por línea, con una breve explicación:
  - **JS/TS:** los scripts del `package.json` (`dev`, `build`, `test`, `lint`, `typecheck`, `format`) con el formato `<gestor> <script>`.
  - **Python:** antepone `uv run` o `poetry run` según el gestor. Incluye `pytest` si `pytest` está en las dependencias o configurado en `pyproject.toml`; `ruff check .` y `ruff format .` si `ruff` está configurado; `mypy .` si `mypy` está configurado. Para Django agrega `python manage.py runserver`, `python manage.py makemigrations` y `python manage.py migrate`. Para FastAPI agrega `uvicorn <modulo>:app --reload` solo si encontraste dónde se crea `app = FastAPI(`. Si no, deja `TODO`.
  - **Java:** con el wrapper detectado. Para Maven: `./mvnw test`, `./mvnw verify` y, si es Spring Boot, `./mvnw spring-boot:run`. Para Gradle: `./gradlew test`, `./gradlew build` y, si es Spring Boot, `./gradlew bootRun`. Agrega la nota: "En PowerShell: `.\mvnw.cmd` / `.\gradlew.bat`".
  - No inventes comandos. Si falta un comando de tests o de lint, anótalo como `TODO`.
- `{{ESTRUCTURA}}`: las carpetas principales (`apps/`, `src/`, `prisma/`, `app/`, `tests/`, `src/main/java/...`, etc.), con una línea cada una. Si el proyecto está vacío, escribe `Proyecto nuevo — completar cuando exista la estructura.`
- `{{CONVENCIONES_LENGUAJE}}`: incluye solo los bloques de los ecosistemas presentes:
  - **TypeScript:** `- TypeScript en modo strict. Nada de any sin un comentario que lo justifique.`
  - **Python:** `- Type hints en toda función pública. Formato y lint con ruff. Nada de except desnudos (except:) ni except Exception sin volver a lanzar o registrar el error.`
  - **Java:** `- Java 17+. Inyección por constructor (nunca @Autowired en campos). Records para DTOs. Optional solo como tipo de retorno, nunca en campos ni parámetros.`
  - **IA** (si hay `llm`, `agents` o `rag`): `- Toda llamada a un modelo pasa por un único módulo gateway de IA. Prompts versionados en archivos. Salidas estructuradas validadas con schema. Ningún cambio de prompt, modelo o retrieval sin correr los evals.`
  - **ML** (si hay `ml`): `- Experimentos reproducibles: semillas fijas, datasets y versiones de modelo registradas, sin datos de test usados para entrenar ni para elegir hiperparámetros.`

**Si ya existe**, no lo sobrescribas. Compáralo con la plantilla y propón solo las secciones que falten, en especial "Flujo de trabajo" y "Stack". Aplícalas únicamente si el usuario acepta.

## Paso 6 — Resumen final

Muestra:

1. Una tabla con cada plugin y su resultado (instalado, ya estaba o error).
2. Si se creó o se actualizó `CLAUDE.md`.
3. Los próximos pasos, textualmente:
   - Ejecuta **`/reload-plugins`** para activar los plugins en esta sesión. Las sesiones nuevas los cargan solas.
   - Haz commit de `.claude/settings.json` y `CLAUDE.md` para que la configuración viaje con el repo.
   - En otra máquina, al abrir el repo y confiar en la carpeta, Claude Code ofrecerá instalar los mismos plugins.
   - Para agregar un stack después, vuelve a ejecutar `/kit:init <stack>`. Solo instala lo que falte.

No hagas commit por tu cuenta. Deja ese paso al usuario.
