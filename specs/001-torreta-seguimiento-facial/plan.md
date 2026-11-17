# Plan 001 — Torreta de Seguimiento Facial con Láser

Estado: aprobado el 2026-11-17 (T01) — layout de `src/`/`tests/`, decisiones D1–D20 y excepción de ADR-011
Spec de referencia: `specs/001-torreta-seguimiento-facial/spec.md` (aprobada)
Constitución: `docs/constitution.md`

Este documento define el **cómo** sin escribir código de implementación. Ninguna versión, dependencia ni valor numérico se considera fijado: todo lo que la spec dejó como `[POR VERIFICAR]` se cierra por ADR o medición de banco antes de implementar. La estructura de carpetas de §3.1 está aprobada; cualquier carpeta adicional exige ADR.

---

## 1. Enfoque técnico

### 1.1 Topología

```text
Teléfono (navegador)
      │ HTTP LAN
      ▼
PC (Python, un proceso)
  ├─ Web local: foto, estado, parada
  ├─ Servicio de sesión y salud de enlaces
  ├─ Core puro: geometría, centrado, límites, estados
  ├─ Adaptador de video ◄──── Wi-Fi local (MJPEG/HTTP) ──── ESP32-CAM sobre la torreta
  ├─ Adaptador de visión (detección, embedding, similitud)
  └─ Adaptador serial USB ◄── USB ──► ESP32 principal
                                        ├─ PWM ──► servo pan
                                        └─ PWM ──► servo tilt

Batería ──► botón físico mantenido ──► módulo láser (siempre encendido)
```

Regla de autoridad:

- El **PC decide a quién apuntar**: qué foto es válida, qué candidato gana, qué ángulo se desea y si el apuntado está centrado.
- El **firmware del ESP32 principal decide hasta dónde es seguro moverse**: aplica límites calibrados, suavizado y watchdog de comando.
- La ESP32-CAM **solo captura y transmite**; no decide y no mueve.
- El **láser no tiene control software**: se alimenta por batería y se corta solo con el botón físico. Ni el PC ni el firmware lo encienden ni lo apagan.

### 1.2 Principios de diseño

1. **Core puro y determinista**: toda decisión verificable (mapeo, centrado, selección, límites, estados) vive en funciones sin I/O, sin reloj real y sin acceso a cámara, red o serial.
2. **Adaptadores finos**: video, visión, serial y web traducen datos; no contienen reglas de negocio.
3. **Dependencia unidireccional**: web/visión → core → contrato serial → firmware. La web nunca habla con el ESP32; el core nunca habla con la cámara.
4. **Corte físico y apuntado seguro**: el láser no es un actuador de software; se alimenta por batería y se corta solo con un botón físico. En software la seguridad se limita a watchdog de comando, límites en un único punto de configuración y parada explícita desde la web.
5. **Sin acumulación de latencia**: se procesa el frame más reciente y se descartan los atrasados; nunca se forma una cola creciente.
6. **Todo dato de personas vive en memoria**: foto de referencia y frames no se escriben a disco ni salen de la LAN.

### 1.3 Modelo de ejecución del PC

Un solo proceso Python con cuatro responsabilidades concurrentes:

| Responsabilidad | Modelo | Regla |
|---|---|---|
| Lectura de video | Hilo dedicado de I/O bloqueante | Publica solo el último frame en un buzón de tamaño 1; descarta el anterior |
| Visión y control | Hilo de cómputo a ritmo acotado | Lee el último frame, ejecuta core puro y emite como máximo un comando por ciclo |
| Serial | Hilo lector de telemetría + escritor desde control | Último comando válido gana; secuencia monotónica |
| Web | Servidor HTTP local con hilos | Solo lee instantáneas inmutables del estado y encola acciones |

El estado compartido se publica como **instantáneas inmutables**; cada campo tiene un único escritor. Esto evita carreras sin introducir un bus de eventos.

---

## 2. Verificación contra la constitución

| Principio | Cumplimiento | Observación |
|---|---|---|
| P1 Simplicidad y stack | **Excepción justificada** | Se mantiene Python 3 en PC y C++ en ESP32, pero se añade una ESP32-CAM con enlace Wi-Fi local, no prevista en la redacción original de P1. Exige ADR y actualización de `README.md` y de la constitución. El ESP32 principal sigue exclusivamente por USB. Versiones y frameworks: ADR. |
| P2 Gobernanza de specs | Cumple | Cada componente y transición se deriva de RF-1…RF-12 y NFR-1…NFR-9. Lo no definido queda `[POR VERIFICAR]`; no se inventa comportamiento. |
| P3 Arquitectura y desacoplamiento | Cumple | Core puro sin I/O; adaptadores finos; UI no habla con firmware; pines de servo, cámara y límites en configuración única por componente (NFR-9); el láser no tiene pin ni estado software. |
| P4 Calidad y testing | Cumple con pendiente | Pirámide definida, core con 100 % de ramas, protocolo contra stub serial, banco reproducible y tiempo inyectado. Para el láser se verifica siempre encendido, corte físico y apuntado al torso cuando el objetivo está centrado. El runner nativo del firmware queda por ADR. |
| P5 Persistencia y datos | Cumple | Sin base de datos, nube ni telemetría; foto y frames solo en memoria; los ángulos son autoridad del firmware y el láser no tiene estado software. |
| P6 Idioma | Cumple | Documento en español; identificadores, rutas y mensajes de protocolo en inglés; UI y mensajes al usuario en español. |

**Excepción P1 registrada:** la ESP32-CAM con Wi-Fi local es un cambio de hardware aprobado en la spec (actualización 2026-11-16), pero contradice la letra de P1 mientras no exista ADR. Sin ese ADR no se implementa.

**Excepción de seguridad registrada (ADR-011):** el láser permanece encendido mientras la batería esté conectada y el botón físico esté presionado; no hay apagado por software, timeout ni apagado por pérdida. El sistema no puede garantizar que el haz no alcance ojos durante arranque, búsqueda, movimiento o fallo. La seguridad depende del operador presente, de un blanco seguro, de la clase/potencia y de un corte físico accesible.

---

## 3. Arquitectura de módulos

### 3.1 Estructura aprobada (2026-11-17)

```text
src/
  pc/
    turret/
      core/                  # Python puro, sin I/O ni reloj real
      app/                   # orquestación, configuración, salud
      adapters/
        video/               # ESP32-CAM
        vision/              # detección, embedding, similitud
        serial/              # enlace USB
      web/                   # HTTP local y UI
      main.py                # composition root
    (manifiesto y venv según ADR)
  firmware/
    main_esp32/
      control_core/          # lógica pura: límites, suavizado, watchdog
      serial_link/
      drivers/
      config/                # único punto de configuración
      main
    camera_esp32/
      stream/
      config/                # único punto de configuración; secretos fuera del repo
      main
tests/
  unit/
  contract/
  integration/
  fixtures/
  bench/
```

No se crea ninguna carpeta fuera de este layout sin ADR. El layout quedó aprobado el 2026-11-17 (T01).

### 3.2 Componentes y responsabilidades

| ID | Componente | Responsabilidad | RF cubiertos |
|---|---|---|---|
| C1 | `core.models` | Tipos inmutables: foto, candidato, observación, ángulos, enlaces, estado, comandos | Todos |
| C2 | `core.geometry` | Punto de apuntado en el torso, conversión Normalizado→ángulo, suma de corrección, recorte a límites | RF-3, RF-5, RF-8 |
| C3 | `core.centering` | Zona central, banda muerta, criterio de “centrado” y “apuntado estable” | RF-5, RF-7 |
| C4 | `core.selection` | Umbral mínimo de similitud, elección del candidato de mayor similitud, abstención | RF-3, RF-4 |
| C6 | `core.session` | Reductor de estado, precedencia, transiciones, sesión sin objetivo, parada | RF-1, RF-2, RF-6…RF-12 |
| C7 | `app.pipeline` | Orden del ciclo: frame → detección → selección → geometría → comando; frecuencia y descarte | RF-3, RF-5…RF-8 |
| C8 | `app.session_service` | Validación y reemplazo de foto, parada explícita, una sola sesión activa | RF-1, RF-2, RF-11, RF-12 |
| C9 | `app.health` | Salud de video y serial, tiempos de silencio, recuperación conjunta | RF-9, RF-10 |
| C10 | `adapters.video.esp32cam` | Abrir stream, decodificar frames, marcar salud y antigüedad | RF-3, RF-9, RF-10 |
| C11 | `adapters.vision` | Detectar rostros, calcular embedding, comparar con la referencia, estimar torso | RF-1, RF-3, RF-4 |
| C12 | `adapters.serial.link` | Handshake, codificar/decodificar mensajes, secuencia, telemetría | RF-3, RF-8…RF-10 |
| C13 | `web` | Subida/captura de foto, estado único (incluye “láser: físico, siempre encendido”), parada explícita, mensajes de rechazo | RF-1, RF-2, RF-11, RF-12 |
| C14 | `firmware.main.control_core` | Recorte a límites, suavizado, watchdog de comando | RF-8, RF-9 |
| C15 | `firmware.main.serial_link` | Framing, parser tolerante, ACK de estado, rechazo de comandos inválidos | RF-3, RF-9, RF-10 |
| C16 | `firmware.main.drivers` | PWM de servos, lectura de límites y estado | RF-3, RF-8 |
| C17 | `firmware.camera.stream` | Cámara, Wi-Fi, stream; sin lógica de seguimiento | RF-3, RF-9 |
| C18 | `main` / composition roots | Arranque ordenado, inyección de reloj, configuración y dependencias | Todos |

> **C5 retirado por ADR-011:** `core.laser` desaparece. El láser no es un actuador de software: se alimenta por batería y se corta solo con el botón físico. Los IDs no se renumeran para no romper la trazabilidad histórica.

### 3.3 Dirección de dependencias

```text
web ─┐
visión ─┼─► app ─► core (puro)
serial ─┘            ▲
                     │ modelos y puertos
firmware control_core (puro, C++) ◄── serial_link ◄── USB ── PC
```

- `core` no importa `app`, `adapters`, `web`, serial, OpenCV ni sockets.
- `adapters` implementan puertos declarados por `core`/`app`; no contienen umbrales ni reglas.
- `firmware.control_core` replica las decisiones de seguridad del firmware y se prueba de forma nativa, sin hardware.
- La duplicación pequeña entre core Python y `control_core` C++ es deliberada: evita FFI, toolchain compartido y acoplamiento de builds. La conformidad se garantiza con vectores contractuales comunes, no compartiendo código.

---

## 4. Modelo de datos

### 4.1 Convenciones

- Ángulos en **grados**; `pan` positivo a la derecha, `tilt` positivo hacia arriba. Resolución y rango efectivos: `[POR VERIFICAR]`.
- Coordenadas de imagen **normalizadas [0,1]**, origen arriba-izquierda, independientes de la resolución del stream.
- Tiempo siempre **monotónico en milisegundos**, inyectado; nunca reloj de pared para control.
- Secuencias de protocolo `uint64` estrictamente crecientes.
- Identificador de foto: hash criptográfico del contenido; sirve para detectar duplicados sin comparar imágenes.
- Enums e identificadores en inglés; textos de UI en español.

### 4.2 Entidades en memoria

| Entidad | Campos principales | Invariantes |
|---|---|---|
| `ReferencePhoto` | `photo_id`, `mime`, `width`, `height`, `image_bytes`, `face_bbox`, `embedding`, `created_at` | Vive solo en memoria; una sola activa; `face_bbox` contiene exactamente un rostro |
| `FaceCandidate` | `bbox`, `detection_score`, `embedding` | Bbox dentro del frame; embedding calculado con el mismo modelo que la referencia |
| `Observation` | `frame_id`, `captured_at`, `received_at`, `candidate`, `similarity`, `aim_point` | `aim_point` es el centro del torso, no el centro del rostro |
| `TrackSnapshot` | `photo_id`, `last_observation`, `last_confirmed_at`, `hits`, `misses`, `last_angles`, `centered`, `settled` | `hits`/`misses` consecutivos; se reinician al confirmar o perder |
| `AngleTarget` | `pan_deg`, `tilt_deg`, `source_frame_id`, `requested_at` | Valores absolutos deseados; el firmware los recorta; nunca se envían ángulos de una sesión anterior |
| `LinkHealth` | `video_ok`, `serial_ok`, `last_frame_age_ms`, `last_telemetry_age_ms`, `firmware_version` | La recuperación exige **ambos** enlaces; no hay OR |
| `SessionSnapshot` | `state`, `reason`, `photo_id`, `target_present`, `laser_control`, `angles`, `links`, `updated_at` | Instantánea inmutable para la web; un solo estado visible; `laser_control` es siempre `physical` (ADR-011) |

### 4.3 Estados de sesión y precedencia

`NO_TARGET`, `SEARCHING`, `TRACKING`, `LOST`, `LIMIT`, `LINK_ERROR`, `STOPPED`.

Precedencia de visualización, de mayor a menor: `STOPPED` > `LINK_ERROR` > `LOST` > `LIMIT` > `TRACKING` > `SEARCHING` > `NO_TARGET`.

| Desde | Condición | Hacia | Acción |
|---|---|---|---|
| `NO_TARGET` | Foto válida, enlaces sanos | `SEARCHING` | No mover; foto activa; el láser no cambia |
| `SEARCHING` | Objetivo confirmado y dentro de rango | `TRACKING` | Enviar ángulos; el láser no cambia |
| `SEARCHING` | Objetivo confirmado fuera de rango | `LIMIT` | Detener ambos ejes; el láser no cambia |
| `TRACKING` | Pérdida de confirmación | `LOST` | Mantener último ángulo; el láser no cambia |
| `LOST` | Reaparición confirmada | `TRACKING` | Reanudar desde la posición actual; el láser no cambia |
| `TRACKING`/`LOST` | Ángulo requerido fuera de tope | `LIMIT` | Detener ambos ejes; el láser no cambia |
| `LIMIT` | Objetivo confirmado de nuevo en rango | `TRACKING` | Reanudar desde posición actual |
| Cualquiera salvo `STOPPED` | Video o USB caído | `LINK_ERROR` | Detener motores, avisar; el láser no cambia |
| `LINK_ERROR` | Video y USB restablecidos y foto activa | `SEARCHING` | No reutilizar ángulos; handshake y posición actual |
| Cualquiera | Parada explícita | `STOPPED` | Sin objetivo; sesión terminada; el láser no cambia |
| `STOPPED` | Foto válida nueva | `SEARCHING` | Nueva sesión; el láser no cambia |
| Cualquiera | Foto inválida | Igual | Conservar foto, objetivo y estado; mostrar motivo |

La foto válida más reciente prevalece. Una foto válida durante el seguimiento entra en `SEARCHING` con los motores detenidos y la posición mantenida hasta confirmar el nuevo objetivo (RF-11); el láser sigue encendido.

### 4.4 Láser

El láser **no tiene política software ni estado en el modelo**. Se alimenta por batería y permanece encendido mientras el botón físico mantenido esté presionado; el único corte es físico (ADR-011).

| Condición | Láser |
|---|---|
| Cualquier estado software (`NO_TARGET`, `SEARCHING`, `TRACKING`, `LOST`, `LIMIT`, `LINK_ERROR`, `STOPPED`) | Encendido: el software no lo apaga |
| Arranque, reinicio del firmware o del PC, caída del proceso | Encendido: no hay apagado por software ni watchdog de láser |
| Pérdida de objetivo, tope, error de enlace o parada | Encendido: solo se detienen los motores |
| Botón físico liberado o batería desconectada | Apagado: único corte admitido |

La web se limita a informar `laser_control: physical` y a recordar al operador que debe accionar el corte físico. Ni el PC ni el firmware envían órdenes de encendido o apagado del láser.

### 4.5 Contrato PC ↔ ESP32 principal

Transporte propuesto: **líneas de texto JSON sobre USB CDC**, UTF-8, una línea por mensaje, tamaño máximo acotado, versión de protocolo y secuencia monotónica. La selección final y el baud se fijan por ADR.

| Mensaje | Dirección | Campos mínimos | Semántica |
|---|---|---|---|
| `hello` | PC → firmware | `protocol_version`, `seq` | Inicia sesión de protocolo |
| `hello_ack` | firmware → PC | `protocol_version`, `limits`, `capabilities`, `seq` | Límites calibrados y versión aceptada |
| `set_target` | PC → firmware | `seq`, `pan_deg`, `tilt_deg`, `ttl_ms` | Ángulo absoluto deseado con vencimiento |
| `stop` | PC → firmware | `seq`, `reason` | Parada segura: motores quietos |
| `state` | firmware → PC | `seq`, `applied_pan_deg`, `applied_tilt_deg`, `flags`, `uptime_ms`, `cmd_age_ms` | Telemetría y estado real aplicado |
| `error` | firmware → PC | `code`, `detail`, `seq` | Comando inválido, versión incompatible o límite |

Reglas del contrato:

- El firmware **recorta** todo ángulo a los límites calibrados y reporta `flags` de tope.
- Un `set_target` con `ttl_ms` vencido no se aplica; si no llega ningún comando fresco, el firmware mantiene la posición. El `ttl_ms` controla solo la frescura del movimiento, nunca el láser.
- El protocolo no tiene campo `laser`; un mensaje que lo incluya se rechaza como desconocido o se ignora explícitamente [POR VERIFICAR en ADR-006].
- Mensajes malformados, versión incompatible o secuencia no creciente se rechazan sin mover los servos.
- Al arrancar, el firmware solo aplica movimiento tras completar `hello`; el láser ya está encendido por hardware.
- Tras reconexión, el PC parte de la posición reportada por `state`; no reutiliza ángulos guardados (RF-10).

### 4.6 Contrato web local

| Recurso | Método | Entrada | Salida |
|---|---|---|---|
| `/` | GET | — | UI mínima: cámara/archivo, estado, parada |
| `/api/photo` | POST | `multipart/form-data`, un archivo | Aceptada con `photo_id`, o rechazada con motivo |
| `/api/status` | GET | — | `SessionSnapshot` |
| `/api/stop` | POST | — | `SessionSnapshot` en `STOPPED` |

- Una sola sesión y una sola foto activa; todas las webs ven el mismo estado.
- `SessionSnapshot.laser_control` es siempre `physical`: la web informa “láser físico, siempre encendido” y no ofrece encender ni apagar.
- La parada (`/api/stop`) no apaga el láser; solo detiene los motores y deja la sesión sin objetivo.
- Sin autenticación (fuera de alcance), pero el servidor escucha solo en la LAN.
- Sin vista de video en la web: la spec no la exige y evitarla elimina dependencias y exposición de frames.
- El intervalo de sondeo del estado queda `[POR VERIFICAR]`; el servidor no empuja eventos.

### 4.7 Configuración

| Componente | Claves | Fuente |
|---|---|---|
| ESP32 principal | Pines de servo, límites por eje, rampa de suavizado, timeout de comando, baud, versión de protocolo | Único archivo de configuración del firmware |
| ESP32-CAM | Credenciales Wi-Fi, resolución, FPS, puerto de stream, identificador del dispositivo | Configuración local no versionada; secretos fuera del repo |
| PC | Puerto serial, URL del stream, umbrales, tiempos, ganancias, FOV, paralaje, banda muerta, puerto web, nivel de log | Único archivo de configuración del PC; secretos por entorno/archivo local |

Formato, versión y manejo de secretos: ADR. Ningún pin, índice de cámara, límite o umbral se dispersa por el código. El láser no aparece en ninguna configuración software: no tiene pin, estado ni timeout.

---

## 5. Estrategia de integración

### 5.1 Arranque ordenado

1. Firmware principal arranca con servos en posición segura; espera `hello`. El láser ya está encendido por hardware si el operador energizó la batería.
2. PC abre el puerto serial, hace handshake y obtiene límites y versión.
3. PC abre el stream de la ESP32-CAM y valida que entrega frames.
4. PC levanta la web local y publica `NO_TARGET` o `LINK_ERROR` según la salud real.
5. El operador sube o captura la foto; solo entonces se pasa a `SEARCHING`.

Si un enlace falta, la web sigue disponible para informar y recibir foto, pero no se mueve. El láser permanece encendido en todos estos pasos; el operador debe energizarlo solo con el área despejada y un blanco seguro.

### 5.2 Ciclo de control

1. Se toma el frame más reciente; si no hay frame nuevo, no se repite cómputo.
2. Se detectan rostros y se calcula embedding de cada candidato.
3. Se elige el candidato sobre el umbral con mayor similitud; si ninguno, abstención.
4. Se calcula el punto de torso y el ángulo deseado a partir de la posición aplicada reportada por el firmware.
5. El core decide estado, centrado y estabilidad; **no decide nada sobre el láser**.
6. Se envía como máximo un `set_target` por ciclo, con TTL y secuencia.
7. Se lee telemetría y se actualiza la instantánea de la web.

La frecuencia de control, la de video y el presupuesto de latencia quedan `[POR VERIFICAR]`. Regla: si la visión es más lenta que el video, se descartan frames; nunca se encola.

### 5.3 Matriz de fallos

| Falla | Detección | PC | Firmware | Web |
|---|---|---|---|---|
| Pérdida de video Wi-Fi | Sin frames por el tiempo configurado | `LINK_ERROR`, mantiene posición | Watchdog de movimiento: mantiene posición | Aviso de error de video |
| Pérdida de USB | Sin telemetría / error de escritura | `LINK_ERROR`, deja de mandar | Watchdog de comando: mantiene posición | Aviso de error de enlace |
| Caída del proceso de visión | Sin comandos | La sesión termina | Watchdog de comando: mantiene posición | La web deja de responder o muestra error |
| Reinicio del firmware | Handshake perdido | `LINK_ERROR` y rehandshake | Arranca con servos en posición segura | Aviso |
| Reinicio del PC | Sin comandos | Nueva sesión sin objetivo | Watchdog de comando: mantiene posición | `NO_TARGET` tras volver |
| Tope de eje | Ángulo recortado y flag | `LIMIT`, ambos ejes detenidos | No fuerza; reporta tope | Estado límite |
| Foto inválida | Validación | Conserva objetivo y estado | Sin cambio | Motivo del rechazo |
| Foto duplicada | Mismo `photo_id` | Reemplazo válido sin cambio observable | Sin cambio | Sin cambio |
| Segundo teléfono | Estado compartido | La última foto válida prevalece | Sin cambio | Mismo estado en todas |
| Congestión Wi-Fi | Antigüedad de frame y descartes | Descarta frames, no acumula | Watchdog si el silencio supera el umbral | Estado de enlace |
| Cinta o botón del láser falla | Observación del operador | Sin detección software | Sin efecto: el láser no es actuador software | Recordatorio de corte físico |

Todas las filas comparten la misma consecuencia: **el láser sigue encendido** salvo que el operador accione el corte físico o se desconecte la batería.

### 5.4 Recuperación

La recuperación exige **video y USB restablecidos**, no uno de los dos. Al volver:

1. Se rehace handshake y se leen límites y posición aplicada.
2. Se conserva la foto activa; no se reutilizan ángulos de antes del corte.
3. El primer apuntado parte de la posición actual y respeta topes.
4. El láser nunca cambió de estado: sigue encendido y solo el corte físico lo apaga.

### 5.5 Parada y ciclo de vida

- La parada explícita desde la web envía `stop`, detiene los motores, deja la sesión sin objetivo y pasa a `STOPPED`. **No apaga el láser.**
- Cerrar el navegador no detiene nada; la sesión vive en el proceso del PC.
- Una foto válida nueva después de `STOPPED` inicia una sesión nueva.
- Si el proceso del PC muere, el firmware mantiene la posición por watchdog; el láser sigue encendido.

### 5.6 Seguridad y secretos

- Credenciales Wi-Fi y puerto serial no se versionan; se leen de configuración local no rastreada.
- Los logs no incluyen imágenes ni embeddings; solo identificadores, estados y métricas.
- El servidor web escucha solo en la interfaz de la LAN.
- El láser se opera únicamente con el operador presente, un blanco seguro, el área de barrido despejada y un corte físico accesible; clase y potencia bajas `[POR VERIFICAR]`.
- La cinta que mantiene presionado el botón es un riesgo documentado: se recomienda un interruptor de retención o una seta.
- Prohibido apuntar deliberadamente a ojos y prohibido dejar el sistema energizado sin supervisión.

---

## 6. Pirámide de tests

### 6.1 Niveles

| Nivel | Alcance | Dobles | Criterio de salida |
|---|---|---|---|
| Unitario de core Python | Geometría, mapeo, centrado, selección, límites, estados, parada | Reloj falso; datos sintéticos | 100 % de ramas en `core` |
| Unitario de `control_core` C++ | Recorte, suavizado, watchdog de comando | Reloj y GPIO falsos; runner nativo por ADR | 100 % de ramas |
| Estático de ausencia de control de láser | Búsqueda de GPIO, relé, campo `laser` y estado de láser en PC y firmware | Analizador de texto/repositorio | Cero coincidencias |
| Contrato de protocolo | Serialización, versión, secuencia, TTL, mensajes malformados, topes, ausencia de `laser` | Stub de puerto serial + vectores comunes | Todos los vectores pasan en Python y C++ |
| Integración del PC | Pipeline completo con video, visión, serial y web simulados | Fuente de frames falsa, firmware simulado, reloj falso, cliente web en proceso | Escenarios de AC-1…AC-10 pasan sin hardware |
| Banco | ESP32-CAM real, ESP32 principal, servos y láser real (batería + corte físico) | Checklist reproducible y evidencia registrada | Todos los ítems en verde |
| Aceptación manual | Demo de flujo y fallos de la spec | Hardware real | AC-1…AC-11 verificados |

Distribución objetivo aproximada: 55 % unitario de core, 15 % unitario de firmware, 15 % contrato, 10 % integración y 5 % banco/estático. La cobertura y el runner se configuran por ADR; hoy no existe ninguno.

### 6.2 Reglas de determinismo

- Nada de `sleep`, esperas arbitrarias ni dependencia de red en pruebas automatizadas.
- Reloj, fuente de frames, aleatoriedad y estado del firmware se inyectan.
- Los escenarios usan observaciones sintéticas y embeddings falsos; no se usan fotos de personas reales.
- Los tests de web usan un cliente en proceso, no un puerto de red externo.
- Los vectores de protocolo son archivos de datos versionados, compartidos por ambas suites.
- Cualquier prueba que necesite hardware se marca como banco y no se ejecuta en la suite automática.

### 6.3 Escenarios mínimos

| Escenario | RF/AC | Nivel |
|---|---|---|
| Foto válida con dos personas; se sigue solo a la de la foto | RF-1, RF-3 / AC-1 | Integración |
| Foto sin rostro, varios rostros, corrupta o formato inválido | RF-2 / AC-2 | Unitario + integración |
| Dos candidatos sobre umbral; gana el de mayor similitud | RF-4 / AC-3 | Unitario |
| Ningún candidato sobre umbral; abstención, sin movimiento y láser sin cambios | RF-4 / AC-3 | Unitario + integración |
| Láser siempre encendido, sin transición por estado ni por centrado | RF-5, RF-6, RF-7 / AC-4 | Unitario (ausencia de lógica) + banco |
| Pérdida y reaparición; motores se detienen y el láser sigue encendido | RF-6, RF-7 / AC-4 | Integración + banco |
| Tope de un eje; se detienen ambos ejes y el láser sigue encendido | RF-8 / AC-5 | Unitario + integración + banco |
| Corte de video, corte de USB y recuperación conjunta; el láser no cambia | RF-9, RF-10 / AC-6 | Integración + banco |
| Foto nueva durante el seguimiento; transición con motores detenidos y láser encendido | RF-11 / AC-7 | Integración |
| Foto duplicada y segundo teléfono | RF-11, RF-12 / CL-6 | Integración |
| Parada explícita; cerrar navegador no detiene; el láser sigue encendido | RF-12 / AC-10 | Integración + banco |
| Proceso de visión caído; el firmware mantiene posición y el láser sigue encendido | RF-9 / AC-6 | Banco |
| Ausencia de control de láser en código y protocolo | NFR-2 / AC-4 | Estático + contrato |
| Corte físico del láser accionado por el operador | NFR-2 / AC-4 | Banco |
| Torso como punto de apuntado y paralaje calibrado | NFR-2 / AC-9 | Unitario + banco |
| Suavidad sin saltos ni vibración | NFR-7 / AC-11 | Banco |

### 6.4 Checklist de banco

1. Stream de la ESP32-CAM estable, con resolución, FPS y latencia medidos.
2. Sentido de giro de ambos servos, topes calibrados y ausencia de forzado.
3. Movimiento suave, sin oscilación ni vibración con la cámara montada, sin barrido deliberado del rostro.
4. Láser siempre encendido con la batería conectada y el botón presionado; verificado antes de mover los servos.
5. Corte físico accesible que apaga el láser de inmediato; comprobar también qué ocurre si la cinta se suelta.
6. Autonomía y temperatura del módulo láser durante una sesión completa.
7. Tope de un eje detiene ambos ejes y el láser sigue encendido.
8. Desconexión de USB: watchdog de movimiento y parada; el láser sigue encendido.
9. Desconexión de Wi-Fi: parada y recuperación solo con ambos enlaces; el láser sigue encendido.
10. Foto desde el teléfono: válida, inválida, duplicada y con dos clientes.
11. Paralaje cámara-láser y punto de torso; verificación sobre un blanco seguro, nunca hacia ojos ni sobre personas sin consentimiento.
12. Alimentación, masa y fijación de la ESP32-CAM; comportamiento del regulador.
13. Muerte del proceso del PC: el firmware mantiene posición y el láser sigue encendido.
14. Evidencia registrada: fecha, operador, versión, clase/potencia del láser, resultado y observaciones.

El LED sustituto ya no representa el comportamiento final: la aceptación exige el módulo real con batería y corte físico, y verificar que no existe apagado por software.

---

## 7. Trazabilidad RF → componente → verificación

| RF | Componentes | Verificación principal |
|---|---|---|
| RF-1 Foto válida inicia sesión | C8, C11, C13, C6, C17 | Unitario de validación; integración de subida; banco con teléfono |
| RF-2 Foto inválida se rechaza sin alterar | C8, C11, C13, C6 | Unitario e integración de motivos y estado previo |
| RF-3 Seguimiento del objetivo en video | C7, C10, C11, C12, C16 | Integración con video y serial simulados; banco en lazo cerrado |
| RF-4 Mayor similitud o abstención | C4, C11, C7 | Unitario de umbral, empate y abstención; integración |
| RF-5 Láser siempre encendido con corte físico | C3, C13, C18 | Estático de ausencia de control; banco con corte físico y torso |
| RF-6 Pérdida; motores se detienen y el láser sigue | C6, C7, C12, C14 | Unitario con reloj falso; integración; banco |
| RF-7 Reaparición sin reencendido por software | C3, C6, C7 | Unitario e integración de secuencia |
| RF-8 Tope de eje detiene ambos | C2, C6, C12, C14, C16 | Unitario de recorte; contrato; banco de topes |
| RF-9 Corte de video o USB | C9, C10, C12, C14, C15, C17 | Integración de salud; contrato; banco de desconexión |
| RF-10 Recuperación con ambos enlaces | C9, C12, C14, C15 | Integración de reconexión; banco |
| RF-11 Reemplazo de objetivo | C6, C8, C13 | Integración de subida durante seguimiento |
| RF-12 Estado visible y parada | C6, C13, C18 | Unitario de precedencia; integración; banco |

| NFR | Componentes | Verificación |
|---|---|---|
| NFR-1 Rendimiento | C7, C10, C9 | Medición de latencia y FPS en banco; descarte de frames en integración |
| NFR-2 Seguridad del láser | C13, C14, C18, hardware | Estático de ausencia de control, contrato sin campo `laser` y checklist con módulo real, batería y corte físico |
| NFR-3 Seguridad mecánica | C2, C14, configuración | Unitario de límites y banco de topes |
| NFR-4 Privacidad | C8, C10, web, logs | Revisión de memoria y ausencia de escritura/red externa |
| NFR-5 Plataforma | C10, C17, C12 | Checklist de banco con teléfono, PC y ESP32-CAM |
| NFR-6 Idioma | UI y documentación | Revisión de textos y nombres |
| NFR-7 Movimiento | C14, C16 | Unitario de rampa y checklist de vibración |
| NFR-8 Verificación | Toda la pirámide | Cobertura, contrato y banco registrados |
| NFR-9 Configuración | Configuración de los tres componentes | Revisión de punto único, ausencia de pines dispersos y ausencia de configuración de láser |

---

## 8. Decisiones técnicas y trade-offs

| # | Decisión | Alternativa descartada | Justificación del trade-off |
|---|---|---|---|
| D1 | ESP32-CAM sobre la torreta envía video por Wi-Fi local; ESP32 principal por USB para motores | Webcam del PC; cámara USB montada con cable al PC | La cámara debe girar con la torreta para que el encuadre coincida con el apuntado. La webcam del PC no cumple eso; una cámara USB con cable añade arrastre y un slip ring. El costo es una excepción a P1 y un enlace Wi-Fi con latencia y jitter que hay que medir. |
| D2 | Visión y decisión en el PC | Detección/embedding en la ESP32-CAM | El PC tiene más cómputo y permite tests puros; la ESP32-CAM queda como sensor. Se pierde autonomía, pero el sistema ya depende del PC para la web y el USB. |
| D3 | Video MJPEG sobre HTTP como opción base | RTSP; UDP crudo; WebRTC | MJPEG es simple, ampliamente soportado por ESP32-CAM y fácil de diagnosticar; el costo es mayor ancho de banda y decodificación. RTSP/UDP/WebRTC se descartan por complejidad y por no necesitar audio ni bidireccionalidad. La elección final se confirma por ADR tras medir latencia. |
| D4 | Hilos con buzón de último frame y descarte | Asyncio puro; multiproceso; cola FIFO | OpenCV y serial son bloqueantes; los hilos evitan envoltorios artificiales. El buzón de tamaño 1 garantiza latencia acotada. Asyncio complica el cómputo síncrono; multiproceso añade IPC y duplicación de la foto. |
| D5 | Core Python puro + `control_core` C++ puro, con vectores comunes | Una sola librería compartida por FFI; lógica solo en firmware; lógica solo en PC | Cada lado se testea en su toolchain sin hardware. Se acepta una duplicación pequeña y controlada; la conformidad se mide con vectores, no con código compartido. FFI acopla builds y complica el banco. |
| D6 | Protocolo serial de líneas JSON con versión y secuencia | Struct binario; COBS+CRC; ROS/serial ROS | JSON es legible, fácil de testear y suficiente para la carga; el costo es bytes y parseo, irrelevante a la tasa de control. Un binario ahorra ancho de banda pero fragiliza el contrato y el diagnóstico. COBS+CRC se reserva si las mediciones muestran corrupción. |
| D7 | Ángulos absolutos deseados + recorte y suavizado en firmware | Enviar deltas; control PID en el PC; dejar la autoridad al PC | Los absolutos son idempotentes y toleran un comando perdido. El firmware conserva la última palabra mecánica y de seguridad (P5). Los deltas acumulan error si se pierde un mensaje. |
| D8 | Reductor de sesión en el PC; seguridad de movimiento en firmware | Estado solo en firmware; estado solo en el PC | La spec exige que el firmware sea autoridad de ángulos, pero la sesión, la foto y la web viven en el PC. Separar responsabilidades evita duplicar la máquina de estados completa. El láser queda fuera de ambos: no tiene estado software. |
| D9 | Láser siempre encendido por batería con corte físico; sin control software | GPIO o relé con apagado por software; láser comandado por el PC | Elimina circuito, pin, driver, protocolo y política. A cambio, el sistema no puede apagar el haz ante pérdida, parada, tope, error o caída del proceso: la seguridad depende del operador, de un blanco seguro y de un corte accesible. Se acepta como excepción registrada en ADR-011. |
| D10 | La web consulta el estado por sondeo HTTP | WebSocket; SSE; streaming de video en la web | Sondeo es suficiente para estados de baja frecuencia, no añade dependencias ni conexiones persistentes y es trivial de testear. WebSocket/SSE se descartan por complejidad; el video no se muestra porque la spec no lo pide. |
| D11 | Servidor HTTP de la biblioteca estándar como opción base | Framework web externo (Flask/FastAPI) | P1 exige preferir la biblioteca estándar. Si el manejo de `multipart` o la robustez del servidor lo justifican, se añade una dependencia con ADR; no se decide ahora. |
| D12 | Configuración única por componente | Configuración global compartida; constantes dispersas | NFR-9 y P3 exigen un punto por componente. Una configuración global acopla firmware y PC y facilita filtrar secretos; constantes dispersas impiden calibrar topes y umbrales. El láser no figura en la configuración: no tiene pin ni estado. |
| D13 | Detección de rostros + embedding + similitud coseno, con abstención | Template matching; Haar solo; tracker profundo de persona | El embedding tolera cambios de pose e iluminación mejor que el template matching y permite un umbral verificable. Un tracker profundo añade dependencia y opacidad. La biblioteca y el modelo exactos se fijan por ADR. |
| D14 | Reloj, frames, serial y aleatoriedad inyectados; vectores contractuales | Pruebas con cámara real y esperas; mocks ad hoc por test | P4 exige determinismo y cobertura. Los dobles permiten probar pérdida, timeout y reconexión sin hardware; el banco cubre lo físico. |
| D15 | Foto y frames solo en memoria | Caché en disco; base de datos local | P5 prohíbe persistencia no decidida y simplify el ciclo de vida. Una caché facilitaría depuración pero crea riesgo de almacenar imágenes de personas. |
| D16 | Una sesión y una foto activa por proceso | Varias sesiones o varias torretas | La spec limita el alcance a una persona y una torreta. La última foto válida prevalece y todas las webs ven lo mismo; multi-sesión multiplica estados y autoridad. |
| D17 | Watchdog de comando en firmware, solo para movimiento | Confiar solo en el PC para detener los servos | El PC puede colgarse o desconectarse. El watchdog detiene el movimiento ante comandos viejos; el costo es un timeout que debe superar el periodo normal de comandos y medirse en banco. No aplica al láser, que no tiene estado software. |
| D18 | Punto de torso derivado del rostro y corregido por paralaje | Apuntar al centro del rostro; apuntar a la caja completa de la persona | La spec exige torso y descarta ojos. Derivarlo del rostro es posible con una sola detección; la corrección de paralaje se calibra en banco. Apuntar al rostro viola NFR-2; la caja completa exige un detector de persona adicional. |
| D19 | Coordenadas normalizadas y mapeo por FOV medido | Píxeles absolutos con supuestos de resolución | Las coordenadas normalizadas sobreviven a cambios de resolución del stream. El FOV real se mide; asumirlo introduce error de apuntado. |
| D20 | Batería + botón mantenido con cinta como único corte del láser | Interruptor de retención; seta de emergencia; relé controlado | Es la opción de menor circuito y cumple lo pedido. El botón mantenido con cinta no es un corte de emergencia fiable y puede soltarse o dejar de hacer contacto; se documenta como riesgo residual y se recomienda migrar a un interruptor de retención o una seta. |

---

## 9. Riesgos y mitigaciones

| Riesgo | Impacto | Mitigación | Verificación |
|---|---|---|---|
| Latencia y jitter del Wi-Fi | Seguimiento oscilante | Descarte de frames, control acotado, medir latencia y FPS | NFR-1 en banco |
| Cámara solidaria a la torreta | Realimentación, blur y oscilación | Banda muerta, rampa de servo, límite de velocidad | NFR-7 y estabilidad en banco |
| Paralaje cámara-láser | El haz no cae en el torso | Calibración y offset configurables | AC-9 en banco |
| Masa y par de la ESP32-CAM | Caída del soporte o pérdida de topes | Fijación rígida, recalibrar límites y rampa | Checklist de banco |
| Alimentación insuficiente | Reinicios de la cámara o del ESP32 | Regulación separada y medición de consumo | Checklist de banco |
| Falsos positivos de similitud | Seguir a otra persona | Umbral, abstención y confirmación por frames | AC-3 y banco |
| Falso negativo por baja luz | Pérdida constante | Tratarlo como pérdida, no forzar detección | CL-3 |
| Caída del proceso del PC | Motores quietos pero láser encendido | Watchdog de movimiento; el operador debe cortar el láser | RF-9 y banco |
| Comando viejo o duplicado | Movimiento incorrecto | Secuencia, TTL y último comando válido | Contrato |
| Saturación del serial | Comandos atrasados | Mensajes pequeños, tasa acotada y telemetría resumida | Medición |
| Credenciales en el repo | Exposición de la LAN | Configuración local no versionada | Revisión |
| Cambio de resolución del stream | Mapeo incorrecto | Coordenadas normalizadas y FOV configurable | Unitario |
| Exposición ocular durante arranque, búsqueda, movimiento o fallo | Daño ocular o deslumbramiento | Clase/potencia bajas, blanco seguro, área despejada, operador presente y corte accesible; prohibido apuntar a ojos | NFR-2 y checklist de banco |
| Fallo de la cinta o del botón mantenido | Láser imposible de cortar sin desconectar la batería | Documentado; se recomienda interruptor de retención o seta; verificar corte antes de energizar | Checklist de banco |
| Autonomía o fallo de la batería | Corte inesperado o descarga | Medir autonomía y usar batería adecuada | Checklist de banco |
| Calor del módulo láser | Degradación o daño del módulo | Régimen de uso y disipación; medir temperatura | Checklist de banco |
| Ausencia de apagado por software | No se puede mitigar por firmware ante un fallo | Se acepta como excepción ADR-011; la mitigación es física y humana | Revisión de ADR |

---

## 10. ADR y datos que deben cerrarse antes de implementar

| ADR | Decisión | Bloquea |
|---|---|---|
| ADR-001 | Versión de Python, venv, pytest, cobertura y herramientas de calidad | Todo el PC |
| ADR-002 | Modelo exacto de ESP32-CAM, framework de su firmware y excepción Wi-Fi a P1; actualización de `README.md` y constitución | C17 y NFR-5 |
| ADR-003 | Framework del ESP32 principal, versión y runner nativo de tests | C14, C15 y P4 |
| ADR-004 | Biblioteca de visión, modelo de embedding, licencia y tamaño | C11 |
| ADR-005 | Transporte de video, resolución, FPS y presupuesto de latencia | C10 y NFR-1 |
| ADR-006 | Protocolo serial: versión, baud, tamaño máximo y política de errores | C12 y C15 |
| ADR-007 | Formato de configuración y manejo de secretos | NFR-9 y P5 |
| ADR-008 | Servidor web: biblioteca estándar o dependencia justificada | C13 |
| ADR-009 | Calibración mecánica y óptica: topes, FOV, paralaje y offset de torso | C2, C14 y AC-9/AC-5 |
| ADR-010 | Parámetros de seguridad y control: TTL de movimiento, confirmación de objetivo, banda muerta y ganancias | C3, C14 y AC-4 |
| ADR-011 | Láser siempre encendido: batería, botón mantenido con corte físico, sin GPIO ni software; enmienda de P3/P4/P5, `AGENTS.md` y `README.md` | C13, C18 y NFR-2 |

Datos `[POR VERIFICAR]` que deben cerrarse con medición o decisión antes de la aceptación: tolerancia de centrado, umbral de similitud, criterios de rostro detectable, frames de confirmación, tiempo de interrupción, topes, formatos y tamaño de foto, latencia/FPS, navegadores y requisitos de PC/cámara, clase y potencia del láser, autonomía y tipo de batería, fiabilidad del botón mantenido con cinta, zona segura de barrido, riesgo de exposición ocular durante búsqueda o fallo, timeout del firmware, estabilidad del lazo, alimentación/masa/fijación y paralaje.

Cada ADR vive en su propio archivo `docs/memory/decisions/ADR-00X-titulo-corto.md`; `docs/memory/decisions.md` es solo el índice. Un archivo por decisión evita que los dos responsables editen el mismo bloque.

---

## 11. Fases de implementación

| Fase | Entregable | Salida |
|---|---|---|
| F0 | ADR y mediciones de banco, sin código de producto | Todos los `[POR VERIFICAR]` cerrados o con método de medición aprobado; `README.md` y constitución actualizados |
| F1 | Core Python puro + contrato serial con vectores | Cobertura de ramas 100 %; protocolo con vectores en verde |
| F2 | Firmware principal: serial, servos, watchdog | `control_core` al 100 %; banco básico de movimiento |
| F3 | Stream de la ESP32-CAM y salud de video | Stream estable medido; corte y recuperación detectados |
| F4 | Adaptador de visión y geometría de torso | AC-1, AC-3 y AC-9 en integración/banco |
| F5 | Web de foto, estado y parada | AC-2, AC-7 y AC-10 en integración y teléfono |
| F6 | Lazo cerrado completo y matriz de fallos | AC-4, AC-5, AC-6 y AC-8 en integración/banco |
| F7 | Aceptación y checklist final | AC-1…AC-11 con evidencia, módulo láser real, batería y corte físico verificado |

Cada fase termina con sus tests en verde; no se avanza con tests fallidos o no ejecutados (P4).

---

## 12. Aprobación

Aprobado el 2026-11-17 (T01) por el responsable del proyecto:

1. La estructura de `src/` y `tests/` de la sección 3.1.
2. Las decisiones D1–D20, en especial MJPEG como base, hilos con descarte, JSON por líneas, ángulos absolutos con recorte en firmware, sondeo web y láser siempre encendido con corte físico.
3. El orden de fases y la exigencia de ADR antes de instalar cualquier dependencia.
4. La actualización de `README.md` y de la constitución para reflejar la ESP32-CAM, el enlace Wi-Fi y el láser siempre encendido.
5. La **excepción de seguridad** de ADR-011: se acepta que el láser no se apague por software y que el haz pueda alcanzar zonas no previstas durante arranque, búsqueda, movimiento o fallo; la mitigación es física y humana.

No se ha escrito código de implementación, no se han creado carpetas y no se han ejecutado tests: el repositorio aún no tiene runner ni toolchain.
