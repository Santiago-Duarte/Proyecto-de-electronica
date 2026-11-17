# AGENTS.md — Torreta de Seguimiento Facial con Láser

## Proyecto

Torreta Pan-Tilt de 2 ejes que identifica a una persona desde una foto subida con el celular, la sigue con una cámara y le apunta con un láser. Proyecto de Electrónica General, de uso local: ESP32 por USB a un PC y el teléfono en la misma red Wi-Fi. Hoy el repo solo tiene documentación SDD: no hay código, manifiestos, tests ni CI.

- Stack: firmware C++ sobre ESP32 ESP-WROOM-32 (framework y versión sin fijar); Python 3 en el PC (versión y librerías sin fijar); sin base de datos; sin toolchain ni build declarado. Toda versión se fija por ADR antes de instalar nada.
- Arquitectura: tres componentes desacoplados — firmware (actuadores y láser por USB), visión (cámara → ángulos) y UI web local (foto objetivo). Core puro testeable + adaptadores de I/O; dependencia unidireccional PC→ESP32, la UI no habla con el firmware.

## Estructura

- `src/` — código fuente: **no existe todavía**; el layout se propone en el primer `plan.md` y requiere aprobación.
- `tests/` — pruebas: **no existe todavía**; se crea junto con el primer código.
- `docs/` — `constitution.md` (reglas innegociables), `mission.md`, `roadmap.md`, `architecture.md` y `memory/{decisions,lessons,glossary}.md`.
- `specs/` — `NNN-nombre-corto/{spec,plan,tasks}.md`; `specs/_template/` es plantilla, no editar.
- `README.md` — hechos de hardware y arquitectura de software; fuente de verdad física.
- `.opencode/` — commands `sdd-*` y skill `sdd-workflow`; no modificar sin pedido.
- `opencode.json` — configuración de OpenCode (`instructions`, `mcp`).

## Comandos

- Instalar dependencias: `—` (no hay manifiesto; no instalar hasta ADR)
- Ejecutar: `—` (no hay código)
- Tests (todos): `—` (no hay runner)
- Test individual: `—`
- Lint/formato: `—`
- Build: `—` (no hay toolchain de firmware declarado)

Ningún comando está disponible aún. Prohibido documentar o sugerir comandos no ejecutados en este repo; el mismo cambio que agrega código o toolchain debe agregarlos aquí ya verificados.

## Estilo y convenciones

- Lenguaje: C++ en firmware y Python 3 en PC; estándar y versiones por ADR. Sin dependencias externas si la biblioteca estándar alcanza.
- Nombres: Python `snake_case` para funciones, variables y módulos, `PascalCase` para clases; C++ según el estilo idiomático del framework que se fije por ADR.
- Idioma: identificadores, nombres de archivos, ramas y commits en inglés; comentarios, mensajes al usuario, UI y documentación en español.
- El core (geometría, mapeo píxel→ángulo, decisión de centrado) no importa cámara, serial, red ni UI; los adaptadores no contienen lógica de negocio.
- Sigue los patrones del código existente antes de introducir uno nuevo; si no hay código, el patrón se define en el plan y se registra en ADR.

## Reglas

- Antes de tocar código, lee `docs/constitution.md` y la spec activa: ahí manda la constitución.
- Haz cambios pequeños y enfocados en lo que se pidió; nada de refactors no solicitados.
- **Pregunta antes de:** añadir dependencias, cambiar la estructura de carpetas, modificar el contrato PC↔ESP32 o el formato de la foto de referencia, borrar o mover archivos, editar `README.md`, `docs/constitution.md`, `docs/mission.md`, `docs/roadmap.md` u `opencode.json`, inicializar git.
- **Nunca:** commitear secretos o credenciales, editar archivos generados o `specs/_template/`, desactivar tests o lint para que pase, hardcodear pines o índices de dispositivo fuera de configuración, inventar versiones o requisitos, energizar el láser sin verificación previa en banco (apagado por defecto, timeout, nunca a ojos).
- Si algo es ambiguo o contradice la spec, detente y pregunta en un solo lote en lugar de suponer.

## Al terminar cualquier tarea

- Ejecuta lint y tests disponibles, y confirma que pasan; si no hay comandos aún, dilo explícitamente y no afirmes validación.
- Comprueba que el cambio cumple la spec activa y la constitución, con cada AC verificado o su imposibilidad registrada.
- Actualiza en el mismo cambio `tasks.md`, el estado de la spec, `specs/README.md` y los ADR/lecciones que correspondan.
- Resume qué cambiaste, qué archivos tocaste y qué queda pendiente.

## Git

- El directorio **no es** un repositorio git: no ejecutes `git init`, commits ni push sin pedido explícito.
- Commits: Conventional Commits en inglés (`feat:`, `fix:`, `docs:`, `test:`, `chore:`).
- Ramas: `feat/…`, `fix/…`, `docs/…`.
- No hagas push ni merge sin que se pida explícitamente.
