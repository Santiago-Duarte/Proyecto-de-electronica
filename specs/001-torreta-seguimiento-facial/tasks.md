# Tareas 001 — Torreta de Seguimiento Facial con Láser

Estado: propuesto — no ejecutar antes de aprobar `plan.md`
Plan: `specs/001-torreta-seguimiento-facial/plan.md`
Spec: `specs/001-torreta-seguimiento-facial/spec.md`
Constitución: `docs/constitution.md`

## Reglas de ejecución

- Cada tarea tiene un máximo de **30 minutos**. Si se excede, se detiene y se divide antes de continuar.
- Ninguna tarea se marca como hecha sin evidencia: test en verde, medición registrada o checklist firmado.
- No se instala ninguna dependencia, no se compila firmware y no se implementa comportamiento antes de cerrar las puertas de la fase 0.
- Se respeta el orden de dependencias indicado en cada tarea. `[P]` significa que puede ejecutarse en paralelo con otras tareas `[P]` cuyas dependencias ya estén cumplidas.
- Una tarea no se inicia si su dependencia está abierta, aunque el compañero esté libre.
- Cada archivo tiene un solo responsable de escritura. El revisor no edita: comenta o abre una tarea correctiva.
- Los valores de seguridad y calibración se toman de ADR-009 y ADR-010; no se inventan en el código.
- El láser no se configura por software: es siempre encendido, con batería y corte físico (ADR-011). Ninguna tarea puede añadir GPIO, relé, campo `laser` ni estado de láser.
- Las tareas de banco con láser real exigen el procedimiento de seguridad de ADR-011, operador presente, corte físico accesible y un blanco seguro; nunca se apunta deliberadamente a ojos y el área de barrido debe estar despejada.
- Dos responsables: **A = PC, visión y web**; **B = firmware, cámara y banco**. Las tareas marcadas “B revisa” o “A revisa” incluyen revisión cruzada.

## Puertas de entrada

| Puerta | Tareas | Bloquea |
|---|---|---|
| G0 Aprobación | T01 | Todo el proyecto |
| G1 Decisiones de stack | T02, T03, T04, T05, T06, T07, T08, T13 | Skeleton, core, firmware y web |
| G2 Valores reales | T08A, T09, T10, T11, T12, T14, T18 | Configuración de producción, seguridad y integración |
| G3 Contrato | T17 | Parser serial, adaptador serial y pruebas de contrato |

## Ruta crítica

- Ruta A: `T01 → T02 → T15 → T19 → T25 → T27 → T52 → T61 → T62 → T63 → T70 → T75 → T76`.
- Ruta B: `T01 → T07/T08 → T09–T12 → T08A/T14/T18 → T29 → T32 → T37 → T42 → T44 → T69 → T70 → T75 → T76`.
- Las tareas de banco no sustituyen los tests automáticos: los confirman.

---

## Fase 0 — Decisiones, mediciones y esqueleto

- [ ] **T01 — Aprobar plan, layout y reglas de trabajo** · A · 15 min · Depende: — · RF: base de RF-1…RF-12 · AC: —
  **Hecho cuando:** el plan, la estructura de `src/`/`tests/` y la excepción de seguridad del láser siempre encendido (ADR-011) están aprobados por el responsable del proyecto, y A y B confirman por escrito sus carriles y la regla de revisión cruzada.

- [ ] **T02 — ADR-001: Python, venv, tests y cobertura** · A · 25 min · Depende: T01 · RF: base de RF-1…RF-12 · AC: —
  **Hecho cuando:** `docs/memory/decisions.md` contiene ADR-001 con versión exacta de Python, pytest y cobertura, alternativa descartada, consecuencias y comandos de creación de venv; B lo revisa y no queda ninguna versión sin fijar.

- [ ] **T03 — ADR-004: biblioteca y modelo de visión** · A · 30 min · Depende: T01 · RF: RF-1, RF-3, RF-4 · AC: AC-1, AC-3
  **Hecho cuando:** ADR-004 fija biblioteca, modelo de embedding, versión, licencia, tamaño y alternativa descartada; no se instala nada todavía y B revisa que no haya dependencia sin justificar.

- [ ] **T04 — ADR-005: transporte de video y presupuesto** · A · 25 min · Depende: T01 · RF: RF-3, RF-9 · AC: AC-1, AC-6
  **Hecho cuando:** ADR-005 elige transporte, resolución, FPS y presupuesto de latencia, con alternativa descartada y método de medición; queda aprobado por B.

- [ ] **T05 — ADR-007: configuración y secretos** · A · 20 min · Depende: T01 · RF: RF-8; NFR-9 · AC: AC-5
  **Hecho cuando:** ADR-007 define formato de configuración, ubicación del archivo local no versionado y tratamiento de credenciales Wi-Fi y puerto serial; la configuración no incluye pin, estado ni timeout de láser (ADR-011); B confirma que no hay secretos en el repo.

- [ ] **T06 — ADR-008: servidor web** · A · 15 min · Depende: T01 · RF: RF-1, RF-2, RF-11, RF-12 · AC: AC-2, AC-7, AC-10
  **Hecho cuando:** ADR-008 decide biblioteca estándar o dependencia externa justificada, con impacto en `multipart` y pruebas; B revisa que no contradiga P1.

- [ ] **T07 — ADR-003: framework y runner de firmware** · B · 30 min · Depende: T01 · RF: RF-3, RF-8, RF-9 · AC: AC-5, AC-6
  **Hecho cuando:** ADR-003 fija framework, versión, runner nativo de tests y alternativa descartada; el firmware queda limitado a servos y movimiento, sin láser; A revisa que permita probar `control_core` sin hardware.

- [ ] **T08 — ADR-002: ESP32-CAM, Wi-Fi y excepción a P1** · B · 30 min · Depende: T01 · RF: RF-3, RF-9, RF-10 · AC: AC-1, AC-6
  **Hecho cuando:** ADR-002 fija modelo exacto de ESP32-CAM, framework de su firmware, justificación del enlace Wi-Fi frente a P1 y actualización de `README.md`/constitución; A revisa que no cambie el alcance de la spec.

- [ ] **T09 — Medir topes reales de pan y tilt** · B · 30 min · Depende: T01 · RF: RF-8; NFR-3 · AC: AC-5
  **Hecho cuando:** ADR-009 registra ángulo mínimo y máximo de cada eje, método e instrumento usados, y confirma que el soporte no fue forzado en ningún extremo.

- [ ] **T10 — Medir FOV y paralaje cámara-láser** · B · 30 min · Depende: T01 · RF: RF-3, RF-9 · AC: AC-9
  **Hecho cuando:** ADR-009 registra FOV horizontal/vertical, offset de paralaje y dos distancias de verificación; las mediciones son reproducibles con el montaje final.

- [ ] **T11 — Verificar masa, fijación y alimentación de la cámara** · B · 30 min · Depende: T01 · RF: RF-3; NFR-7 · AC: AC-11
  **Hecho cuando:** ADR-009 registra masa de la ESP32-CAM, tipo de fijación, consumo medido y comportamiento del regulador; el soporte mantiene la posición sin oscilar.

- [ ] **T12 — Caracterizar el láser real y su corte físico** · B · 30 min · Depende: T01 · RF: RF-5; NFR-2 · AC: AC-4, AC-9
  **Hecho cuando:** ADR-010 registra clase, potencia, distancia segura, autonomía de la batería, tipo de botón mantenido y fiabilidad de la cinta, zona segura de barrido y procedimiento de banco con corte físico; no se registra tiempo máximo de encendido ni timeout porque el láser no se controla por software; A revisa que el procedimiento nunca apunte a ojos.

- [ ] **T08A — Ratificar ADR-011 y enmendar constitución, `AGENTS.md` y `README.md`** · A · 25 min · Depende: T01, T12 · RF: RF-5; NFR-2 · AC: AC-4, AC-8
  **Hecho cuando:** `docs/memory/decisions.md` contiene ADR-011 con la excepción de láser siempre encendido y corte físico; P3, P4 y P5 de `docs/constitution.md`, la regla de seguridad de `AGENTS.md` y la sección de hardware de `README.md` ya no mencionan GPIO, control por software ni apagado por timeout; `docs/memory/lessons.md` registra el riesgo residual; B revisa que la documentación no contradiga la spec.

- [ ] **T13 — ADR-006: protocolo serial** · B · 30 min · Depende: T01 · RF: RF-3, RF-8, RF-9, RF-10 · AC: AC-5, AC-6
  **Hecho cuando:** ADR-006 fija versión, baud, formato de líneas, tamaño máximo, secuencia, TTL y política ante mensajes inválidos; el contrato no incluye campo `laser` y define qué hacer si llega uno; A revisa que cubra hello, set_target, stop, state y error.

- [ ] **T14 — ADR-009: consolidar calibración** · B · 20 min · Depende: T09, T10, T11 · RF: RF-3, RF-8, RF-9 · AC: AC-5, AC-9, AC-11
  **Hecho cuando:** ADR-009 contiene límites, FOV, paralaje, offset de torso, masa y fijación, con referencias a T09–T11; no queda ningún valor de calibración `[POR VERIFICAR]`.

- [ ] **T15 — Crear esqueleto Python y manifiesto** · A · 20 min · Depende: T01, T02 · RF: base de RF-1…RF-12 · AC: —
  **Hecho cuando:** existen `src/pc/turret/{core,app,adapters,web}`, `tests/` y el manifiesto del ADR-001; el entorno se crea y el paquete vacío se importa con código 0.

- [ ] **T16 — Crear esqueleto de firmware y manifiestos** · B · 20 min · Depende: T01, T07 · RF: base de RF-3, RF-5, RF-8, RF-9 · AC: —
  **Hecho cuando:** existen `src/firmware/main_esp32` y `src/firmware/camera_esp32` con sus manifiestos, configuración local de ejemplo y un build vacío que compila con el framework del ADR-003.

- [ ] **T17 — Definir vectores contractuales v1** · B · 30 min · Depende: T13 · RF: RF-3, RF-8, RF-9, RF-10 · AC: AC-5, AC-6
  **Hecho cuando:** existen al menos seis vectores versionados que cubren comando válido, TTL vencido, ángulo fuera de tope, mensaje malformado, secuencia duplicada, `stop` y mensaje con campo `laser` obsoleto; ninguno transporta estado de láser; A los revisa contra el ADR-006.

- [ ] **T18 — Cerrar ADR-010: seguridad y control** · A · 30 min · Depende: T09, T10, T12, T14 · RF: RF-5, RF-6, RF-8, RF-9 · AC: AC-4, AC-5, AC-6
  **Hecho cuando:** ADR-010 fija TTL de movimiento, frames de confirmación, banda muerta, ganancias, timeout de enlace y presupuesto de latencia; no fija tiempo máximo ni rearme de láser porque no existe control software; incluye el procedimiento de banco con láser siempre encendido y corte físico; B revisa que todos los valores sean medibles.

**Fin de fase 0:** no se avanza a implementación hasta que T01–T18 estén marcadas y exista ADR-009/ADR-010 sin valores abiertos de seguridad.

---

## Fase 1 — Core Python puro

- [ ] **T19 — Modelos inmutables y enums** · A · 30 min · Depende: T02, T15 · RF: base de RF-1…RF-12 · AC: —
  **Hecho cuando:** `tests/unit/core/test_models.py` pasa, cubre inmutabilidad y valores de enums, y el módulo alcanza 100 % de ramas; no existe `LaserRequest` ni `laser_on`, y `SessionSnapshot` expone `laser_control = physical`.

- [ ] **T20 — Geometría: torso, Normalizado→ángulo y recorte** · A · 30 min · Depende: T19 · RF: RF-3, RF-8 · AC: AC-1, AC-5, AC-9
  **Hecho cuando:** los tests cubren centro, bordes 0/1, fuera de rango, offset de torso y recorte de ambos ejes; el módulo alcanza 100 % de ramas y no importa I/O.

- [ ] **T21 — Centrado, banda muerta y apuntado estable** · A · 30 min · Depende: T19 · RF: RF-5, RF-7 · AC: AC-4
  **Hecho cuando:** los tests cubren dentro/fuera de zona, banda muerta, movimiento y estabilidad tras los frames del ADR-010; el módulo alcanza 100 % de ramas.

- [ ] **T22 — Selección de candidato y abstención** · A · 30 min · Depende: T19 · RF: RF-3, RF-4 · AC: AC-1, AC-3
  **Hecho cuando:** los tests cubren candidato único, varios sobre umbral, ninguno sobre umbral, empate y lista vacía; el resultado es siempre el de mayor similitud o abstención.

- [ ] **T23 — Límites y recorte de ángulos** · A · 25 min · Depende: T19 · RF: RF-8 · AC: AC-5
  **Hecho cuando:** los tests cubren por debajo, en el tope y por encima en ambos ejes; la salida nunca excede ADR-009 y el módulo alcanza 100 % de ramas.

- [ ] **T24 — Prueba estática de ausencia de control de láser** · A · 20 min · Depende: T19, T18 · RF: RF-5, RF-6, RF-7 · AC: AC-4
  **Hecho cuando:** una prueba recorre `src/` y falla si aparece GPIO de láser, relé, campo `laser`, estado de láser, gracia o tiempo máximo; se documenta que el core no tiene módulo `core.laser` y el test pasa.

- [ ] **T25 — Reductor de sesión y precedencia** · A · 30 min · Depende: T19 · RF: RF-1, RF-2, RF-6…RF-12 · AC: AC-2, AC-4, AC-6, AC-7, AC-10
  **Hecho cuando:** los tests recorren todas las filas de la tabla de transición y todas las combinaciones de precedencia; `STOPPED` y `LINK_ERROR` ganan siempre según la spec; ninguna transición modifica el láser.

- [ ] **T26 — Puertos y reloj inyectable** · A · 20 min · Depende: T19 · RF: base de RF-1…RF-12 · AC: —
  **Hecho cuando:** existen interfaces de video, serial, reloj y vista; un reloj falso avanza sin `sleep` y los tests demuestran que el core no lee tiempo real.

- [ ] **T27 — Puerta de cobertura del core** · A · 15 min · Depende: T20, T21, T22, T23, T24, T25, T26 · RF: todos · AC: —
  **Hecho cuando:** el reporte de cobertura marca 100 % de ramas en `core` y no hay líneas ni ramas excluidas sin justificación en ADR.

- [ ] **T28 — Auditoría de pureza del core** · B · 15 min · Depende: T27 · RF: todos; P3 · AC: —
  **Hecho cuando:** una prueba de imports y una revisión muestran que `core` solo usa biblioteca estándar, no importa cámara, serial, red, web ni reloj real, y no contiene ni importa lógica de láser; B firma la revisión.

---

## Fase 2 — Firmware del ESP32 principal

- [ ] **T29 — Configuración única del firmware** · B · 20 min · Depende: T18, T14 · RF: RF-8; NFR-9 · AC: AC-5
  **Hecho cuando:** un solo archivo contiene pines de servo, límites, rampa, TTL, baud y versión; no contiene pin, estado ni timeout de láser; no hay literales duplicados y el build pasa.

- [ ] **T30 — `control_core`: límites y recorte** · B · 30 min · Depende: T29 · RF: RF-8 · AC: AC-5
  **Hecho cuando:** el runner nativo pasa casos por debajo, en el tope y por encima en ambos ejes; la salida nunca excede ADR-009 y hay 100 % de ramas.

- [ ] **T31 — `control_core`: suavizado** · B · 30 min · Depende: T29 · RF: RF-3; NFR-7 · AC: AC-11
  **Hecho cuando:** un comando escalón produce incrementos acotados por la rampa del ADR-010, sin salto ni inversión, y el test nativo pasa con 100 % de ramas.

- [ ] **T32 — `control_core`: watchdog de movimiento** · B · 30 min · Depende: T29, T18 · RF: RF-8, RF-9 · AC: AC-5, AC-6
  **Hecho cuando:** los tests cubren TTL vencido, sin handshake, comando duplicado y tope; ante comando viejo el firmware mantiene la posición; el módulo no tiene funciones ni campos de láser y hay 100 % de ramas.

- [ ] **T33 — Framing y parser serial** · B · 30 min · Depende: T29, T13, T17 · RF: RF-3, RF-9, RF-10 · AC: AC-6
  **Hecho cuando:** los seis vectores de T17 se decodifican en el runner nativo; un mensaje malformado devuelve error y no mueve los servos; un campo `laser` se rechaza o ignora explícitamente y nunca altera hardware.

- [ ] **T34 — Handshake y versión de protocolo** · B · 30 min · Depende: T33 · RF: RF-9, RF-10 · AC: AC-6
  **Hecho cuando:** versión incompatible se rechaza, `hello` válido desbloquea comandos y `state` reporta versión y límites sin campo de láser; los tests nativos pasan.

- [ ] **T35 — Driver de servos** · B · 30 min · Depende: T30, T31 · RF: RF-3, RF-8 · AC: AC-1, AC-5
  **Hecho cuando:** en banco ambos ejes se mueven en el sentido correcto dentro de los límites, sin zumbido de esfuerzo continuo ni pérdida de paso; queda registro con ángulos de prueba.

- [ ] **T36 — Hardware del láser: batería, botón y corte físico** · B · 30 min · Depende: T12, T08A · RF: RF-5; NFR-2 · AC: AC-4, AC-8
  **Hecho cuando:** el módulo láser queda alimentado por batería a través del botón mantenido con cinta y sin conexión al ESP32; se verifica con el firmware apagado que el láser está encendido y que liberar el botón o desconectar la batería lo apaga; no existe GPIO ni driver de láser en el firmware.

- [ ] **T37 — Secuencia de arranque del firmware** · B · 30 min · Depende: T34, T35, T36 · RF: RF-9 · AC: AC-6
  **Hecho cuando:** con el firmware solo, los servos están quietos y el láser ya está encendido por hardware; tras `hello` acepta un comando de movimiento; tras reiniciar el firmware se repite el estado de servos seguro y el láser no cambia.

- [ ] **T38 — Banco de servos: dirección y topes** · B · 30 min · Depende: T35, T37 · RF: RF-3, RF-8 · AC: AC-5
  **Hecho cuando:** se comanda cada eje al mínimo, centro y máximo de ADR-009 y vuelve; no se fuerza el soporte y los topes coinciden con la medición.

- [ ] **T39 — Banco del láser: siempre encendido y corte físico** · B · 30 min · Depende: T36, T37 · RF: RF-5, RF-9 · AC: AC-4, AC-6
  **Hecho cuando:** con el módulo real y un blanco seguro, se registra que el láser está encendido al arrancar, sigue encendido al vencer el TTL de movimiento, al reiniciar el firmware y al detener la sesión, y solo se apaga al liberar el botón o desconectar la batería; se registra también la autonomía observada.

- [ ] **T40 — Puerta de cobertura del firmware** · B · 15 min · Depende: T30, T31, T32, T33, T34 · RF: todos · AC: —
  **Hecho cuando:** el runner nativo reporta 100 % de ramas en `control_core` y el reporte queda adjunto a la tarea.

---

## Fase 3 — Cámara ESP32-CAM

- [ ] **T41 — Configuración Wi-Fi y secretos de la cámara** · B · 25 min · Depende: T08, T11 · RF: RF-3, RF-9 · AC: AC-1, AC-6
  **Hecho cuando:** la cámara se conecta a la Wi-Fi local usando configuración no versionada, el repositorio no contiene credenciales y la conexión se mantiene cinco minutos sin caídas.

- [ ] **T42 — Stream de video** · B · 30 min · Depende: T41 · RF: RF-3 · AC: AC-1
  **Hecho cuando:** el PC abre la URL del stream y recibe frames con la resolución y FPS de ADR-005; el stream se mantiene cinco minutos sin reiniciar la cámara.

- [ ] **T43 — Salud y reconexión del stream** · B · 30 min · Depende: T42 · RF: RF-9, RF-10 · AC: AC-6
  **Hecho cuando:** al desconectar la cámara la salud pasa a caída dentro del tiempo de ADR-010 y al reconectar solo se recupera cuando el USB también está disponible; queda log.

- [ ] **T44 — Banco de stream: latencia y FPS** · B · 30 min · Depende: T42, T43 · RF: RF-3, RF-9; NFR-1 · AC: AC-1, AC-6
  **Hecho cuando:** se registran latencia extremo a extremo y FPS sostenidos durante cinco minutos, sin retraso creciente, y se comparan contra ADR-005.

---

## Fase 4 — Adaptadores de visión

- [ ] **T45 — Fuente de video con último frame y descarte** · A · 30 min · Depende: T03, T04, T19 · RF: RF-3, RF-9 · AC: AC-1, AC-6
  **Hecho cuando:** una fuente falsa publica 100 frames y el consumidor recibe solo el más reciente; el test comprueba el conteo de descartes y no hay cola creciente.

- [ ] **T46 — Detector de rostros** · A · 30 min · Depende: T03, T45 · RF: RF-1, RF-3, RF-4 · AC: AC-1, AC-2
  **Hecho cuando:** fixtures con cero, uno y dos rostros devuelven los conteos esperados y las cajas quedan dentro del frame; el test es determinista y no usa red.

- [ ] **T47 — Embedding y comparación base** · A · 30 min · Depende: T03, T45 · RF: RF-1, RF-3, RF-4 · AC: AC-1, AC-3
  **Hecho cuando:** la misma imagen produce similitud dentro de la tolerancia de identidad y una imagen distinta queda por debajo del umbral de ADR-010; el test es repetible.

- [ ] **T48 — Matcher y asociación temporal** · A · 30 min · Depende: T22, T47 · RF: RF-3, RF-4 · AC: AC-1, AC-3
  **Hecho cuando:** el matcher elige el mayor sobre umbral, abstiene si ninguno lo supera, resuelve empate de forma determinista y no cambia de candidato por un frame aislado.

- [ ] **T49 — Punto de torso y corrección de paralaje** · A · 30 min · Depende: T20, T45 · RF: RF-3, RF-5 · AC: AC-9
  **Hecho cuando:** el punto calculado queda por debajo del centro del rostro según ADR-009, se recorta al frame y aplica el offset de paralaje; los tests cubren bordes.

- [ ] **T50 — Integración de la cadena de visión** · A · 30 min · Depende: T46, T47, T48, T49 · RF: RF-1, RF-3, RF-4 · AC: AC-1, AC-2, AC-3
  **Hecho cuando:** con frames sintéticos de dos personas se selecciona la correcta, sin candidato se abstiene y no se emite comando; el escenario pasa de punta a punta.

---

## Fase 5 — Web y servicios del PC

- [ ] **T51 — Cargador de configuración** · A · 25 min · Depende: T05, T15 · RF: RF-8; NFR-9 · AC: AC-5
  **Hecho cuando:** una configuración válida carga todos los valores del PC; una clave faltante o un valor inseguro falla con mensaje claro y no aplica valores por defecto silenciosos; no existe ninguna clave de láser.

- [ ] **T52 — Servicio de sesión y foto activa** · A · 30 min · Depende: T19, T25, T51 · RF: RF-1, RF-2, RF-11, RF-12 · AC: AC-2, AC-7, AC-10
  **Hecho cuando:** los tests cubren foto válida, inválida, reemplazo, duplicado, parada y nueva foto tras parada; siempre hay una sola foto activa, el estado anterior se conserva ante rechazo y ninguna acción de sesión enciende o apaga el láser.

- [ ] **T53 — Supervisor de salud de enlaces** · A · 30 min · Depende: T25, T26, T51 · RF: RF-9, RF-10 · AC: AC-6
  **Hecho cuando:** con reloj falso, la pérdida de video o USB produce `LINK_ERROR`, la recuperación exige ambos y no se reutilizan ángulos; los tests pasan.

- [ ] **T54 — Endpoint de estado** · A · 25 min · Depende: T52, T53 · RF: RF-12 · AC: AC-7, AC-10
  **Hecho cuando:** `GET /api/status` devuelve el esquema de `SessionSnapshot`, con `laser_control: physical`, no incluye bytes de imagen y el test de cliente en proceso recibe 200 con un solo estado.

- [ ] **T55 — Endpoint de foto y validación** · A · 30 min · Depende: T03, T06, T46, T52 · RF: RF-1, RF-2 · AC: AC-1, AC-2
  **Hecho cuando:** los tests cubren foto válida, sin rostro, varios rostros, archivo corrupto, formato no admitido y tamaño excedido; cada rechazo devuelve motivo y no altera el objetivo.

- [ ] **T56 — Endpoint de parada** · A · 20 min · Depende: T06, T52 · RF: RF-12 · AC: AC-10
  **Hecho cuando:** `POST /api/stop` es idempotente, devuelve `STOPPED`, detiene el seguimiento y deja la sesión sin objetivo sin enviar ninguna orden de láser; el test de cliente en proceso pasa.

- [ ] **T57 — UI móvil de foto, estado y parada** · A · 30 min · Depende: T54, T55, T56 · RF: RF-1, RF-12 · AC: AC-1, AC-10
  **Hecho cuando:** desde un teléfono en la LAN se abre la página, se elige o captura una foto, se ve el estado y se pulsa parada; la UI muestra “láser: físico, siempre encendido” con un recordatorio de corte físico y no ofrece control de láser; el checklist manual queda firmado.

- [ ] **T58 — Varios clientes y foto duplicada** · A · 30 min · Depende: T55, T56, T57 · RF: RF-11, RF-12 · AC: AC-7
  **Hecho cuando:** dos clientes ven la misma instantánea, la última foto válida prevalece y una subida duplicada no produce cambio observable; los tests pasan.

---

## Fase 6 — Serial, pipeline e integración del PC

- [ ] **T59 — Adaptador serial y firmware simulado** · B · 30 min · Depende: T17, T18, T26, T51 · RF: RF-3, RF-8, RF-9, RF-10 · AC: AC-5, AC-6
  **Hecho cuando:** el adaptador completa handshake, envía `set_target`, recibe `state` y el firmware simulado aplica recorte, TTL y watchdog de movimiento; el mensaje no transporta láser y los tests pasan sin hardware.

- [ ] **T60 — Pruebas de contrato serial** · B · 30 min · Depende: T59 · RF: RF-3, RF-8, RF-9, RF-10 · AC: AC-5, AC-6
  **Hecho cuando:** todos los vectores de T17 pasan contra el adaptador, incluidos TTL vencido, secuencia duplicada, mensaje malformado, ángulo fuera de tope y campo `laser` obsoleto; no hay efectos laterales ni cambios de estado de láser.

- [ ] **T61 — Orquestador del pipeline** · A · 30 min · Depende: T18, T23, T24, T50, T52, T53, T59 · RF: RF-3, RF-5, RF-6, RF-8 · AC: AC-1, AC-4, AC-5
  **Hecho cuando:** con fuente y serial falsos se emite como máximo un comando por ciclo, se descarta el frame viejo, el estado del pipeline coincide con la tabla de sesión y el ciclo no calcula ni solicita láser.

- [ ] **T62 — Composition root del PC** · A · 30 min · Depende: T57, T60, T61 · RF: todos · AC: —
  **Hecho cuando:** el proceso arranca sin foto en `NO_TARGET`, con la web respondiendo, el serial conectado o en `LINK_ERROR`, y publica `laser_control: physical` sin enviar ninguna orden de láser; el smoke test pasa.

- [ ] **T63 — Integración: pérdida y reaparición** · A · 30 min · Depende: T61, T62 · RF: RF-5, RF-6, RF-7 · AC: AC-4
  **Hecho cuando:** con reloj falso, la pérdida mantiene la posición y detiene los motores de inmediato, sin ventana de gracia; el snapshot sigue reportando `laser_control: physical` antes, durante y después de la reaparición; el test marca los tiempos.

- [ ] **T64 — Integración: tope de eje** · A · 25 min · Depende: T61, T62 · RF: RF-8 · AC: AC-5
  **Hecho cuando:** un objetivo fuera de rango detiene ambos ejes, el afectado en el tope, sin alterar el estado del láser, y solo reanuda al volver a rango; el test pasa.

- [ ] **T65 — Integración: corte y recuperación de enlaces** · A · 30 min · Depende: T61, T62 · RF: RF-9, RF-10 · AC: AC-6
  **Hecho cuando:** la pérdida de video y de USB produce `LINK_ERROR` sin cambio en `laser_control`, y solo la recuperación de ambos reanuda con la foto activa sin ángulos viejos; el test pasa.

- [ ] **T66 — Integración: fotos inválidas, reemplazo y parada** · A · 30 min · Depende: T61, T62 · RF: RF-1, RF-2, RF-11, RF-12 · AC: AC-2, AC-7, AC-10
  **Hecho cuando:** una foto inválida conserva objetivo y estado, una válida reemplaza con los motores detenidos y la parada termina la sesión; los tres escenarios pasan sin que el láser cambie de estado.

- [ ] **T67 — Verificación de privacidad y no persistencia** · A · 20 min · Depende: T62 · RF: RF-9; NFR-4 · AC: AC-8
  **Hecho cuando:** una prueba y una revisión demuestran que foto y frames no se escriben a disco, no se envían fuera de la LAN y los logs no contienen imágenes ni embeddings.

---

## Fase 7 — Banco y aceptación

- [ ] **T68 — Checklist de banco reproducible** · B · 30 min · Depende: T01, T18 · RF: todos · AC: AC-1…AC-11
  **Hecho cuando:** existe un checklist con los 14 puntos del plan, incluidos láser siempre encendido, corte físico, autonomía y fiabilidad de la cinta, cada uno con valor esperado, resultado real y evidencia; A lo revisa y firma.

- [ ] **T69 — Smoke de banco: cámara, servos y láser encendido** · B · 30 min · Depende: T37, T44, T62 · RF: RF-3, RF-5, RF-8 · AC: AC-1, AC-4, AC-8
  **Hecho cuando:** con hardware real se ve el stream, ambos servos se mueven dentro de límites, el láser está encendido y el corte físico funciona; queda registro con fecha, operador y versión.

- [ ] **T70 — Banco: pérdida de objetivo y continuidad del láser** · B · 30 min · Depende: T63, T69 · RF: RF-5, RF-6, RF-7 · AC: AC-4
  **Hecho cuando:** al ocultar al objetivo los motores se detienen de inmediato, la posición se mantiene, el láser sigue encendido sin ventana de gracia y al reaparecer no hay reencendido porque nunca se apagó; los tiempos quedan registrados.

- [ ] **T71 — Banco: corte y recuperación de USB/Wi-Fi** · B · 30 min · Depende: T65, T69 · RF: RF-9, RF-10 · AC: AC-6
  **Hecho cuando:** al desconectar USB o Wi-Fi la torreta detiene los motores y el láser sigue encendido; al restablecer ambos reanuda con la foto activa y sin ángulos viejos; queda registro.

- [ ] **T72 — Banco: tope de un eje detiene ambos** · B · 30 min · Depende: T64, T69 · RF: RF-8 · AC: AC-5
  **Hecho cuando:** al llevar el objetivo fuera de rango se detienen ambos ejes, el afectado en su tope, no se fuerza el soporte y el láser sigue encendido hasta el corte físico.

- [ ] **T73 — Banco: torso, paralaje y seguridad** · B · 30 min · Depende: T49, T69 · RF: RF-3; NFR-2 · AC: AC-9
  **Hecho cuando:** con el láser siempre encendido, el haz incide en el torso de un blanco seguro por debajo del rostro a dos distancias, nunca de forma deliberada en ojos, el área está despejada y el offset de paralaje se ajusta según ADR-009.

- [ ] **T74 — Banco: suavidad, masa y vibración** · B · 30 min · Depende: T31, T38, T69 · RF: RF-3; NFR-7 · AC: AC-11
  **Hecho cuando:** el movimiento no presenta saltos ni vibración que golpee el soporte, con la cámara montada, sin que el barrido pase por el rostro, según el criterio de ADR-010; queda registro visual.

- [ ] **T75 — Demo integral y casos de fallo** · B · 30 min · Depende: T70, T71, T72, T73, T74 · RF: todos · AC: AC-1…AC-11
  **Hecho cuando:** se ejecuta la demo completa de la spec: foto válida, seguimiento, torso con láser siempre encendido, pérdida, reanudación, foto inválida, límite, corte/reconexión, reemplazo, parada y corte físico del láser; A observa y firma.

- [ ] **T76 — Cierre documental y aceptación** · A · 30 min · Depende: T75 · RF: todos · AC: AC-1…AC-11
  **Hecho cuando:** no queda ninguna casilla abierta en `tasks.md`, `spec.md` y `specs/README.md` muestran 001 en estado `hecha`, y `docs/memory/decisions.md` (incluido ADR-011), `docs/memory/lessons.md`, `docs/constitution.md`, `AGENTS.md` y `README.md` están actualizados con el láser siempre encendido y corte físico; no queda ningún `[POR VERIFICAR]` sin cierre o aceptación explícita.

---

## Reparto por persona

### Persona A — PC, visión y web

| Orden | Tareas |
|---|---|
| 1 | T01, T02, T03, T04, T05, T06, T08A, T15, T18 |
| 2 | T19, T20, T21, T22, T23, T24, T25, T26, T27 |
| 3 | T45, T46, T47, T48, T49, T50 |
| 4 | T51, T52, T53, T54, T55, T56, T57, T58 |
| 5 | T61, T62, T63, T64, T65, T66, T67 |
| 6 | T76 |

### Persona B — firmware, cámara y banco

| Orden | Tareas |
|---|---|
| 1 | T07, T08, T09, T10, T11, T12, T13, T14, T16, T17 (apoyo en T18) |
| 2 | T29, T30, T31, T32, T33, T34, T35, T36, T37, T38, T39, T40 |
| 3 | T41, T42, T43, T44 |
| 4 | T28 (auditoría), T59, T60 |
| 5 | T68, T69, T70, T71, T72, T73, T74, T75 |

### Puntos de sincronización obligatorios

- **S1 — Fin de fase 0:** A y B revisan T01–T18, incluida T08A; no se implementa con ADR abiertos.
- **S2 — Fin del core y del firmware base:** A entrega T27 y B entrega T40; se ejecutan T59–T60 contra los vectores de T17.
- **S3 — Preintegración:** A confirma T50–T58 y B confirma T37–T44; recién entonces se ejecuta T61.
- **S4 — Prebanco:** A confirma T62–T67 y B confirma T68; se agenda el banco con el módulo láser real, batería y corte físico accesible.
- **S5 — Aceptación:** T75 y T76 se firman juntos.

## Regla de bloqueo

Si una tarea de una persona queda bloqueada por una dependencia de la otra, esa persona no adelanta trabajo de una fase posterior: toma la primera tarea libre de su propio carril cuya dependencia esté cerrada. Si no existe, ayuda en revisión o en la medición de banco, pero no edita el archivo del compañero.
