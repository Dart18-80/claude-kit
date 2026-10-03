# Avisos de terceros

Parte del contenido de este repositorio proviene de proyectos de código abierto con licencia MIT. Se incluye con su aviso de copyright, como exige esa licencia.

## ECC — Everything Claude Code

- Repositorio: https://github.com/affaan-m/ECC
- Commit de origen: `ef648e0`
- Licencia: MIT

Archivos adaptados (se ajustaron referencias a otras partes de ECC y se agregaron enlaces a este kit):

| Archivo en claude-kit | Origen en ECC |
|---|---|
| `plugins/stack-typescript/agents/typescript-reviewer.md` | `agents/typescript-reviewer.md` |
| `plugins/stack-postgres/agents/database-reviewer.md` | `agents/database-reviewer.md` |
| `plugins/stack-nestjs/skills/nestjs-patterns/` | `skills/nestjs-patterns/` |
| `plugins/stack-nestjs/skills/backend-patterns/` | `skills/backend-patterns/` |
| `plugins/stack-nestjs/skills/api-design/` | `skills/api-design/` |
| `plugins/stack-prisma/skills/prisma-patterns/` | `skills/prisma-patterns/` |
| `plugins/stack-prisma/skills/database-migrations/` | `skills/database-migrations/` |
| `plugins/stack-postgres/skills/postgres-patterns/` | `skills/postgres-patterns/` |
| `plugins/stack-redis/skills/redis-patterns/` | `skills/redis-patterns/` |
| `plugins/stack-nextjs/skills/nextjs-turbopack/` | `skills/nextjs-turbopack/` |
| `plugins/stack-nextjs/skills/react-patterns/` | `skills/react-patterns/` |
| `plugins/stack-nextjs/skills/react-testing/` | `skills/react-testing/` |
| `plugins/stack-react-native/skills/react-native-patterns/` | `skills/react-native-patterns/` |
| `plugins/stack-python/skills/python-patterns/`, `python-testing/` | `skills/python-patterns/`, `skills/python-testing/` |
| `plugins/stack-python/agents/python-reviewer.md` | `agents/python-reviewer.md` |
| `plugins/stack-fastapi/skills/fastapi-patterns/` | `skills/fastapi-patterns/` |
| `plugins/stack-fastapi/agents/fastapi-reviewer.md` | `agents/fastapi-reviewer.md` |
| `plugins/stack-django/skills/django-patterns/`, `django-tdd/`, `django-security/`, `django-verification/` | `skills/django-*` (mismos nombres) |
| `plugins/stack-django/agents/django-reviewer.md`, `django-build-resolver.md` | `agents/django-reviewer.md`, `agents/django-build-resolver.md` |
| `plugins/stack-java/skills/java-coding-standards/` | `skills/java-coding-standards/` |
| `plugins/stack-java/agents/java-reviewer.md`, `java-build-resolver.md` | `agents/java-reviewer.md`, `agents/java-build-resolver.md` |
| `plugins/stack-springboot/skills/springboot-patterns/`, `springboot-tdd/`, `springboot-security/`, `springboot-verification/`, `jpa-patterns/` | `skills/` (mismos nombres) |
| `plugins/stack-llm/skills/cost-aware-llm-pipeline/`, `regex-vs-llm-structured-text/` | `skills/` (mismos nombres) |
| `plugins/stack-agents/skills/agent-harness-construction/`, `loop-design-check/`, `agent-architecture-audit/`, `mcp-server-patterns/` | `skills/` (mismos nombres) |
| `plugins/stack-rag/agents/rag-pipeline-reviewer.md` | `agents/rag-pipeline-reviewer.md` |
| `plugins/stack-ml/skills/mle-workflow/`, `pytorch-patterns/`, `ml-adoption-playbook/` | `skills/` (mismos nombres; a `mle-workflow` se le quitaron las secciones que dependían del catálogo de ECC) |
| `plugins/stack-ml/agents/mle-reviewer.md`, `pytorch-build-resolver.md` | `agents/mle-reviewer.md`, `agents/pytorch-build-resolver.md` |

`postgres-patterns` y `database-reviewer` indican además que se basan en *Supabase Agent Skills* (equipo de Supabase, licencia MIT).

```
MIT License

Copyright (c) 2026 Affaan Mustafa

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```
