---
name: init
description: Configura el proyecto actual con claude-kit — instala a nivel de proyecto los plugins de stack (nestjs, prisma, postgres, redis, nextjs, react-native, typescript) y los plugins base (superpowers, pr-review-toolkit, security-guidance, commit-commands), y crea el CLAUDE.md del proyecto. Úsalo solo cuando el usuario ejecute /kit:init.
argument-hint: "[stacks...] [--sin-base] [--dry-run]   ej: nestjs prisma postgres redis"
disable-model-invocation: true
allowed-tools: [Read, Glob, Grep, Write, Edit, Bash]
---

# /kit:init — configurar este proyecto con claude-kit

Argumentos del usuario: `$ARGUMENTS`

Objetivo: dejar el proyecto actual listo para trabajar, registrando los plugins **a nivel de proyecto** (`--scope project`). Así quedan anotados en `.claude/settings.json`, que se versiona con el repo, y no afectan a otros proyectos.

Responde al usuario en español. Sé breve: muestra el plan, ejecútalo y muestra el resumen.

## Constantes

- Marketplace del kit: `claude-kit`, en GitHub `Dart18-80/claude-kit`
- Marketplace oficial: `claude-plugins-official`, en GitHub `anthropics/claude-plugins-official`
- Plugins base (oficiales): `superpowers`, `pr-review-toolkit`, `security-guidance`, `commit-commands`

| Stack (argumento) | Alias aceptados | Plugin |
|---|---|---|
| `typescript` | `ts` | `stack-typescript` |
| `nestjs` | `nest` | `stack-nestjs` |
| `prisma` | — | `stack-prisma` |
| `postgres` | `postgresql`, `pg`, `supabase` | `stack-postgres` |
| `redis` | `bull`, `bullmq` | `stack-redis` |
| `nextjs` | `next`, `react` | `stack-nextjs` |
| `react-native` | `rn`, `expo` | `stack-react-native` |

## Paso 1 — Ubicar el proyecto

1. Ejecuta `git rev-parse --show-toplevel`. Si responde, esa es la **raíz del proyecto**. Si falla, la raíz es el directorio actual. En ese caso avisa que el proyecto no es un repo git y que conviene ejecutar `git init` para versionar la configuración. Si el usuario quiere seguir, sigue.
2. Si el directorio actual no es la raíz del proyecto, avisa y pregunta si quiere configurar la raíz o el subdirectorio. Los plugins de proyecto se leen desde el directorio donde se abre Claude Code.
3. Todos los comandos siguientes se ejecutan **desde la raíz elegida**.

## Paso 2 — Decidir los stacks

**Si hay argumentos**, conviértelos a plugins con la tabla, sin distinguir mayúsculas. Ignora `--sin-base` y `--dry-run`, que son banderas. Si un argumento no está en la tabla, avisa y muestra la lista válida.

**Si no hay stacks en los argumentos**, detéctalos:

- Lee el `package.json` de la raíz y, si es un monorepo (`pnpm-workspace.yaml`, o `workspaces` en `package.json`), también los `package.json` de `apps/*` y `packages/*`.
- Reglas de detección:
  - `@nestjs/core` → `nestjs`
  - `prisma` o `@prisma/client`, o un archivo `schema.prisma` → `prisma`
  - `pg`, `postgres`, `@supabase/supabase-js`, o `provider = "postgresql"` en `schema.prisma` → `postgres`
  - `ioredis`, `redis`, `bullmq`, `bull`, `@nestjs/bullmq` o `@nestjs/bull` → `redis`
  - `next` → `nextjs`
  - `react-native` o `expo` → `react-native`
  - `typescript` en dependencias, o un `tsconfig.json` → `typescript`
- Muestra lo detectado y **pregunta antes de instalar**. Si no hay `package.json`, pregunta qué stack usará el proyecto.

**Siempre:** si se eligió `nestjs`, `nextjs`, `react-native` o `prisma`, agrega también `typescript`.

## Paso 3 — Mostrar el plan

Antes de ejecutar nada, muestra:

- la raíz del proyecto,
- los plugins de stack que se instalarán,
- los plugins base que se instalarán (salvo que venga `--sin-base`),
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
```

Manejo de errores:

- **El marketplace oficial no está registrado:** ejecuta `claude plugin marketplace add anthropics/claude-plugins-official` y reintenta.
- **El marketplace `claude-kit` ya existe:** no es un error. Continúa.
- **Un plugin ya está instalado:** continúa con el siguiente.
- **`claude` no se encuentra en el PATH:** detente y muestra los comandos para que el usuario los ejecute en su terminal.
- Cualquier otro error: muéstralo tal cual, sigue con el resto y lístalo al final.

## Paso 5 — CLAUDE.md

Busca `CLAUDE.md` en la raíz del proyecto.

**Si no existe**, créalo desde la plantilla `templates/CLAUDE.md`, ubicada en el directorio base de esta skill. Rellena los marcadores `{{...}}` con datos reales del proyecto:

- `{{NOMBRE}}`: el `name` del `package.json`, o el nombre de la carpeta.
- `{{DESCRIPCION}}`: el `description` del `package.json`. Si no hay, pregunta al usuario en una línea. Si el usuario no responde, deja `TODO: describir el proyecto`.
- `{{STACK}}`: lista con viñetas de los stacks elegidos y sus versiones principales tomadas del `package.json`.
- `{{GESTOR}}`: el gestor de paquetes según el lockfile: `pnpm-lock.yaml` → pnpm, `yarn.lock` → yarn, `bun.lockb` o `bun.lock` → bun, `package-lock.json` → npm. Por defecto, pnpm.
- `{{COMANDOS}}`: los scripts reales del `package.json`, en especial `dev`, `build`, `test`, `lint`, `typecheck` y `format`, con el formato `<gestor> <script>`. Si un script no existe, no lo inventes. Anótalo como `TODO` solo si es `test` o `lint`.
- `{{ESTRUCTURA}}`: las carpetas principales (`apps/`, `packages/`, `src/`, `prisma/`…) con una línea cada una. Si el proyecto está vacío, escribe `Proyecto nuevo — completar cuando exista la estructura.`

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
