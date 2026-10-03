# {{NOMBRE}}

{{DESCRIPCION}}

## Stack

{{STACK}}

Gestor de dependencias / build: {{GESTOR}}

No uses otro gestor ni generes lockfiles de otro.

## Comandos

{{COMANDOS}}

## Estructura

{{ESTRUCTURA}}

## Flujo de trabajo

Sigue este orden en cualquier cambio que no sea trivial:

1. **Entender:** si el pedido es ambiguo, usa la skill `brainstorming` (Superpowers) antes de tocar código.
2. **Planear:** para cambios de varios pasos, escribe el plan con `writing-plans` y espera mi aprobación.
3. **TDD:** usa `test-driven-development`. Primero un test que falle, después el código mínimo para pasarlo y luego refactor. Para bugs, empieza con un test que reproduzca el fallo.
4. **Verificar:** antes de decir que algo está listo, usa `verification-before-completion` y corre de verdad typecheck, lint y tests. Muéstrame la salida.
5. **Revisar:** para cambios grandes, pide una revisión con los agentes de `pr-review-toolkit` y con el revisor del lenguaje (`typescript-reviewer`, `python-reviewer` o `java-reviewer`, según corresponda).
6. **Commit:** solo cuando yo lo pida, con `/commit-commands:commit`.

Para errores sin causa clara, usa `systematic-debugging` en lugar de probar cambios al azar.

## Convenciones

- Código, nombres, commits y comentarios en **inglés**. La conversación conmigo, en español.
- Commits con Conventional Commits (`feat:`, `fix:`, `refactor:`, `test:`, `docs:`, `chore:`).
{{CONVENCIONES_LENGUAJE}}
- Valida en los bordes del sistema (DTOs y entradas externas). Dentro del dominio, confía en los tipos.
- Secretos solo en variables de entorno validadas al arrancar. Nunca en el código ni en logs.

## No hacer

- No instales dependencias nuevas sin decirme cuál y por qué.
- No modifiques migraciones que ya se aplicaron. Crea una nueva.
- No desactives tests, reglas de lint ni chequeos de tipos para "hacer pasar" algo.
- No hagas `git push` ni cambies la configuración de CI sin que te lo pida.
