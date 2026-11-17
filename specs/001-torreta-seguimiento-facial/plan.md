# Plan 001 — Torreta de Seguimiento Facial con Láser

Estado: propuesto — pendiente de aprobación
Spec de referencia: `specs/001-torreta-seguimiento-facial/spec.md` (draft)
Constitución: `docs/constitution.md`

Este documento define el **cómo** sin escribir código de implementación. Ninguna versión, dependencia ni valor numérico se considera fijado: todo lo que la spec dejó como `[POR VERIFICAR]` se cierra por ADR o medición de banco antes de implementar. La estructura de carpetas es una propuesta y requiere aprobación explícita.

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
  ├─ Core puro: geometría, centrado, límites, láser, estados
  ├─ Adaptador de video ◄──── Wi-Fi local (MJPEG/HTTP) ──── ESP32-CAM sobre la torreta
  ├─ Adaptador de visión (detección, embedding, similitud)
  └─ Adaptador serial USB ◄── USB ──► ESP32 principal
                                        ├─ PWM ──► servo pan
                                        ├─ PWM ──► servo tilt
                                        └─ GPIO ─► módulo láser
```

Regla de autoridad:

- El **PC decide a quién apuntar**: qué foto es válida, qué candidato gana, qué ángulo se desea y si el apuntado está centrado.
- El **firmware del ESP32 principal decide hasta dónde es seguro moverse y cuándo apagar el láser**: aplica límites calibrados, suavizado, timeout de comando, tiempo máximo de encendido y apagado por defecto.
- La ESP32-CAM **solo captura y transmite**; no decide, no mueve y no controla el láser.

### 1.2 Principios de diseño

1. **Core puro y determinista**: toda decisión verificable (mapeo, centrado, selección, límites, política de láser, estados) vive en funciones sin I/O, sin reloj real y sin acceso a cámara, red o serial.
2. **Adaptadores finos**: video, visión, serial y web traducen datos; no contienen reglas de negocio.
3. **Dependencia unidireccional**: web/visión → core → contrato serial → firmware. La web nunca habla con el ESP32; el core nunca habla con la cámara.
4. **Seguro por defecto**: láser apagado al arrancar, timeout de comando en firmware, límites en un único punto de configuración y parada explícita desde la web.
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
| P3 Arquitectura y desacoplamiento | Cumple | Core puro sin I/O; adaptadores finos; UI no habla con firmware; pines, cámara, límites y láser en configuración única por componente (NFR-9). |
| P4 Calidad y testing | Cumple con pendiente | Pirámide definida, core con 100 % de ramas, protocolo contra stub serial, banco reproducible y tiempo inyectado. El runner nativo del firmware queda por ADR. |
| P5 Persistencia y datos | Cumple | Sin base de datos, nube ni telemetría; foto y frames solo en memoria; ángulos y estado del láser son autoridad del firmware. |
| P6 Idioma | Cumple | Documento en español; identificadores, rutas y mensajes de protocolo en inglés; UI y mensajes al usuario en español. |

**Excepción P1 registrada:** la ESP32-CAM con Wi-Fi local es un cambio de hardware aprobado en la spec (actualización 2026-11-16), pero contradice la letra de P1 mientras no exista ADR. Sin ese ADR no se implementa.

---

## 3. Arquitectura de módulos

### 3.1 Estructura propuesta (requiere aprobación)

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
      control_core/          # lógica pura: límites, suavizado, láser, watchdog
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

No se crea ninguna carpeta hasta aprobar este layout.

### 3.2 Componentes y responsabilidades

| ID | Componente | Responsabilidad | RF cubiertos |
|---|---|---|---|
| C1 | `core.models` | Tipos inmutables: foto, candidato, observación, ángulos, enlaces, estado, comandos | Todos |
| C2 | `core.geometry` | Punto de apuntado en el torso, conversión Normalizado→ángulo, suma de corrección, recorte a límites | RF-3, RF-5, RF-8 |
| C3 | `core.centering` | Zona central, banda muerta, criterio de “centrado” y “apuntado estable” | RF-5, RF-7 |
| C4 | `core.selection` | Umbral mínimo de similitud, elección del candidato de mayor similitud, abstención | RF-3, RF-4 |
| C5 | `core.laser` | Decisión de encendido: centrado, estabilidad, gracia por pérdida, tiempo máximo, bloqueos | RF-5, RF-6, RF-7 |
| C6 | `core.session` | Reductor de estado, precedencia, transiciones, sesión sin objetivo, parada | RF-1, RF-2, RF-6…RF-12 |
| C7 | `app.pipeline` | Orden del ciclo: frame → detección → selección → geometría → comando; frecuencia y descarte | RF-3, RF-5…RF-8 |
| C8 | `app.session_service` | Validación y reemplazo de foto, parada explícita, una sola sesión activa | RF-1, RF-2, RF-11, RF-12 |
| C9 | `app.health` | Salud de video y serial, tiempos de silencio, recuperación conjunta | RF-9, RF-10 |
| C10 | `adapters.video.esp32cam` | Abrir stream, decodificar frames, marcar salud y antigüedad | RF-3, RF-9, RF-10 |
| C11 | `adapters.vision` | Detectar rostros, calcular embedding, comparar con la referencia, estimar torso | RF-1, RF-3, RF-4 |
| C12 | `adapters.serial.link` | Handshake, codificar/decodificar mensajes, secuencia, telemetría | RF-3, RF-5, RF-8…RF-10 |
| C13 | `web` | Subida/captura de foto, estado único, parada explícita, mensajes de rechazo | RF-1, RF-2, RF-11, RF-12 |
| C14 | `firmware.main.control_core` | Recorte a límites, suavizado, watchdog de comando, política de láser defensiva | RF-5, RF-6, RF-8, RF-9 |
| C15 | `firmware.main.serial_link` | Framing, parser tolerante, ACK de estado, rechazo de comandos inválidos | RF-3, RF-9, RF-10 |
| C16 | `firmware.main.drivers` | PWM de servos, GPIO del láser, lectura de límites y estado | RF-3, RF-5, RF-8 |
| C17 | `firmware.camera.stream` | Cámara, Wi-Fi, stream; sin lógica de seguimiento | RF-3, RF-9 |
| C18 | `main` / composition roots | Arranque ordenado, inyección de reloj, configuración y dependencias | Todos |

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
| `LaserRequest` | `on`, `deadline`, `reason` | `on` solo si la política del core lo permite; `deadline` acota el encendido |
| `LinkHealth` | `video_ok`, `serial_ok`, `last_frame_age_ms`, `last_telemetry_age_ms`, `firmware_version` | La recuperación exige **ambos** enlaces; no hay OR |
| `SessionSnapshot` | `state`, `reason`, `photo_id`, `target_present`, `laser_on`, `angles`, `links`, `updated_at` | Instantánea inmutable para la web; un solo estado visible |

### 4.3 Estados de sesión y precedencia

`NO_TARGET`, `SEARCHING`, `TRACKING`, `LOST`, `LIMIT`, `LINK_ERROR`, `STOPPED`.

Precedencia de visualización, de mayor a menor: `STOPPED` > `LINK_ERROR` > `LOST` > `LIMIT` > `TRACKING` > `SEARCHING` > `NO_TARGET`.

| Desde | Condición | Hacia | Acción |
|---|---|---|---|
| `NO_TARGET` | Foto válida, enlaces sanos | `SEARCHING` | Láser apagado; no mover; foto activa |
| `SEARCHING` | Objetivo confirmado y dentro de rango | `TRACKING` | Enviar ángulos; láser solo si centrado y estable |
| `SEARCHING` | Objetivo confirmado fuera de rango | `LIMIT` | Detener ambos ejes; láser apagado |
| `TRACKING` | Pérdida de confirmación | `LOST` | Mantener último ángulo; iniciar gracia de láser 1–2 s |
| `LOST` | Reaparición confirmada | `TRACKING` | Reanudar; láser solo al centrar |
| `LOST` | Gracia vencida | `LOST` | Láser apagado; mantener posición |
| `TRACKING`/`LOST` | Ángulo requerido fuera de tope | `LIMIT` | Detener ambos ejes; láser apagado |
| `LIMIT` | Objetivo confirmado de nuevo en rango | `TRACKING` | Reanudar desde posición actual |
| Cualquiera salvo `STOPPED` | Video o USB caído | `LINK_ERROR` | Detener, láser apagado, avisar |
| `LINK_ERROR` | Video y USB restablecidos y foto activa | `SEARCHING` | No reutilizar ángulos; handshake y posición actual |
| Cualquiera | Parada explícita | `STOPPED` | Láser apagado; sin objetivo; sesión terminada |
| `STOPPED` | Foto válida nueva | `SEARCHING` | Nueva sesión; láser apagado |
| Cualquiera | Foto inválida | Igual | Conservar foto, objetivo y estado; mostrar motivo |

La foto válida más reciente prevalece. Una foto válida durante el seguimiento entra en `SEARCHING` con láser apagado y posición mantenida hasta confirmar el nuevo objetivo (RF-11).

### 4.4 Política de láser

| Condición | Láser |
|---|---|
| `STOPPED`, `LINK_ERROR`, `NO_TARGET`, `SEARCHING` | Apagado |
| `LIMIT` | Apagado hasta volver a rango |
| `TRACKING` con apuntado centrado y estable | Puede encenderse |
| `TRACKING` con servos en movimiento o fuera de zona | Apagado |
| `LOST` dentro de la gracia y último estado centrado | Puede permanecer encendido |
| `LOST` fuera de la gracia, o último apuntado no centrado | Apagado |
| Tiempo máximo continuo alcanzado | Apagado; rearme `[POR VERIFICAR]` |
| Firmware sin comando fresco o sin handshake | Apagado por watchdog |

La decisión del PC es una **solicitud**; el firmware puede apagarla siempre. Nunca puede encenderla si el PC no la solicita.

### 4.5 Contrato PC ↔ ESP32 principal

Transporte propuesto: **líneas de texto JSON sobre USB CDC**, UTF-8, una línea por mensaje, tamaño máximo acotado, versión de protocolo y secuencia monotónica. La selección final y el baud se fijan por ADR.

| Mensaje | Dirección | Campos mínimos | Semántica |
|---|---|---|---|
| `hello` | PC → firmware | `protocol_version`, `seq` | Inicia sesión de protocolo |
| `hello_ack` | firmware → PC | `protocol_version`, `limits`, `capabilities`, `seq` | Límites calibrados y versión aceptada |
| `set_target` | PC → firmware | `seq`, `pan_deg`, `tilt_deg`, `laser`, `ttl_ms` | Ángulo absoluto deseado y solicitud de láser con vencimiento |
| `stop` | PC → firmware | `seq`, `reason` | Parada segura: láser apagado y motores quietos |
| `state` | firmware → PC | `seq`, `applied_pan_deg`, `applied_tilt_deg`, `laser`, `flags`, `uptime_ms`, `cmd_age_ms` | Telemetría y estado real aplicado |
| `error` | firmware → PC | `code`, `detail`, `seq` | Comando inválido, versión incompatible o límite |

Reglas del contrato:

- El firmware **recorta** todo ángulo a los límites calibrados y reporta `flags` de tope.
- Un `set_target` con `ttl_ms` vencido no enciende el láser; si no llega ningún comando fresco, el firmware apaga el láser y mantiene la posición.
- `laser=true` solo se honra si hay comando fresco, handshake válido, sin tope y sin tiempo máximo alcanzado.
- Mensajes malformados, versión incompatible o secuencia no creciente se rechazan sin mover ni encender.
- Al arrancar, el firmware no acepta `laser=true` hasta completar `hello`.
- Tras reconexión, el PC parte de la posición reportada por `state`; no reutiliza ángulos guardados (RF-10).

### 4.6 Contrato web local

| Recurso | Método | Entrada | Salida |
|---|---|---|---|
| `/` | GET | — | UI mínima: cámara/archivo, estado, parada |
| `/api/photo` | POST | `multipart/form-data`, un archivo | Aceptada con `photo_id`, o rechazada con motivo |
| `/api/status` | GET | — | `SessionSnapshot` |
| `/api/stop` | POST | — | `SessionSnapshot` en `STOPPED` |

- Una sola sesión y una sola foto activa; todas las webs ven el mismo estado.
- Sin autenticación (fuera de alcance), pero el servidor escucha solo en la LAN.
- Sin vista de video en la web: la spec no la exige y evitarla elimina dependencias y exposición de frames.
- El intervalo de sondeo del estado queda `[POR VERIFICAR]`; el servidor no empuja eventos.

### 4.7 Configuración

| Componente | Claves | Fuente |
|---|---|---|
| ESP32 principal | Pines de servo y láser, límites por eje, rampa de suavizado, timeout de comando, tiempo máximo de láser, baud, versión de protocolo | Único archivo de configuración del firmware |
| ESP32-CAM | Credenciales Wi-Fi, resolución, FPS, puerto de stream, identificador del dispositivo | Configuración local no versionada; secretos fuera del repo |
| PC | Puerto serial, URL del stream, umbrales, tiempos, ganancias, FOV, paralaje, banda muerta, puerto web, nivel de log | Único archivo de configuración del PC; secretos por entorno/archivo local |

Formato, versión y manejo de secretos: ADR. Ningún pin, índice de cámara, límite o umbral se dispersa por el código.

---

## 5. Estrategia de integración

### 5.1 Arranque ordenado

1. Firmware principal arranca con láser apagado y servos en posición segura; espera `hello`.
2. PC abre el puerto serial, hace handshake y obtiene límites y versión.
3. PC abre el stream de la ESP32-CAM y valida que entrega frames.
4. PC levanta la web local y publica `NO_TARGET` o `LINK_ERROR` según la salud real.
5. El operador sube o captura la foto; solo entonces se pasa a `SEARCHING`.

Si un enlace falta, la web sigue disponible para informar y recibir foto, pero no se mueve ni se enciende el láser.

### 5.2 Ciclo de control

1. Se toma el frame más reciente; si no hay frame nuevo, no se repite cómputo.
2. Se detectan rostros y se calcula embedding de cada candidato.
3. Se elige el candidato sobre el umbral con mayor similitud; si ninguno, abstención.
4. Se calcula el punto de torso y el ángulo deseado a partir de la posición aplicada reportada por el firmware.
5. El core decide estado, centrado, estabilidad y solicitud de láser.
6. Se envía como máximo un `set_target` por ciclo, con TTL y secuencia.
7. Se lee telemetría y se actualiza la instantánea de la web.

La frecuencia de control, la de video y el presupuesto de latencia quedan `[POR VERIFICAR]`. Regla: si la visión es más lenta que el video, se descartan frames; nunca se encola.

### 5.3 Matriz de fallos

| Falla | Detección | PC | Firmware | Web |
|---|---|---|---|---|
| Pérdida de video Wi-Fi | Sin frames por el tiempo configurado | `LINK_ERROR`, mantiene posición, láser apagado | Watchdog apaga láser si dejan de llegar comandos | Aviso de error de video |
| Pérdida de USB | Sin telemetría / error de escritura | `LINK_ERROR`, deja de mandar | Watchdog apaga láser y mantiene posición | Aviso de error de enlace |
| Caída del proceso de visión | Sin comandos | La sesión termina | Watchdog apaga láser | La web deja de responder o muestra error |
| Reinicio del firmware | Handshake perdido | `LINK_ERROR` y rehandshake | Arranca con láser apagado | Aviso |
| Reinicio del PC | Sin comandos | Nueva sesión sin objetivo | Watchdog apaga láser | `NO_TARGET` tras volver |
| Tope de eje | Ángulo recortado y flag | `LIMIT`, ambos ejes detenidos | No fuerza; reporta tope | Estado límite |
| Foto inválida | Validación | Conserva objetivo y estado | Sin cambio | Motivo del rechazo |
| Foto duplicada | Mismo `photo_id` | Reemplazo válido sin cambio observable | Sin cambio | Sin cambio |
| Segundo teléfono | Estado compartido | La última foto válida prevalece | Sin cambio | Mismo estado en todas |
| Congestión Wi-Fi | Antigüedad de frame y descartes | Descarta frames, no acumula | Watchdog si el silencio supera el umbral | Estado de enlace |

### 5.4 Recuperación

La recuperación exige **video y USB restablecidos**, no uno de los dos. Al volver:

1. Se rehace handshake y se leen límites y posición aplicada.
2. Se conserva la foto activa; no se reutilizan ángulos de antes del corte.
3. El primer apuntado parte de la posición actual y respeta topes.
4. El láser permanece apagado hasta que el objetivo vuelva a estar centrado y estable.

### 5.5 Parada y ciclo de vida

- La parada explícita desde la web envía `stop`, apaga el láser, deja la sesión sin objetivo y pasa a `STOPPED`.
- Cerrar el navegador no detiene nada; la sesión vive en el proceso del PC.
- Una foto válida nueva después de `STOPPED` inicia una sesión nueva.
- Si el proceso del PC muere, el firmware apaga el láser por watchdog; no hay reanudación automática sin una nueva sesión.

### 5.6 Seguridad y secretos

- Credenciales Wi-Fi y puerto serial no se versionan; se leen de configuración local no rastreada.
- Los logs no incluyen imágenes ni embeddings; solo identificadores, estados y métricas.
- El servidor web escucha solo en la interfaz de la LAN.
- El láser real se opera únicamente en banco controlado, con el procedimiento y la clase definidos por ADR.

---

## 6. Pirámide de tests

### 6.1 Niveles

| Nivel | Alcance | Dobles | Criterio de salida |
|---|---|---|---|
| Unitario de core Python | Geometría, mapeo, centrado, selección, límites, láser, estados, parada | Reloj falso; datos sintéticos | 100 % de ramas en `core` |
| Unitario de `control_core` C++ | Recorte, suavizado, watchdog, política defensiva de láser | Reloj y GPIO falsos; runner nativo por ADR | 100 % de ramas |
| Contrato de protocolo | Serialización, versión, secuencia, TTL, mensajes malformados, topes | Stub de puerto serial + vectores comunes | Todos los vectores pasan en Python y C++ |
| Integración del PC | Pipeline completo con video, visión, serial y web simulados | Fuente de frames falsa, firmware simulado, reloj falso, cliente web en proceso | Escenarios de AC-1…AC-10 pasan sin hardware |
| Banco | ESP32-CAM real, ESP32 principal, servos y láser real | Checklist reproducible y evidencia registrada | Todos los ítems en verde |
| Aceptación manual | Demo de flujo y fallos de la spec | Hardware real | AC-1…AC-11 verificados |

Distribución objetivo aproximada: 60 % unitario de core, 15 % unitario de firmware, 15 % contrato, 8 % integración y 2 % banco. La cobertura y el runner se configuran por ADR; hoy no existe ninguno.

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
| Ningún candidato sobre umbral; abstención, sin movimiento y láser apagado | RF-4 / AC-3 | Unitario + integración |
| Láser solo al centrar y estabilizar; se apaga en la gracia | RF-5, RF-6, RF-7 / AC-4 | Unitario + integración + banco |
| Pérdida y reaparición; no se reenciende hasta centrar | RF-6, RF-7 / AC-4 | Integración |
| Tope de un eje; se detienen ambos y el láser queda apagado | RF-8 / AC-5 | Unitario + integración + banco |
| Corte de video, corte de USB y recuperación conjunta | RF-9, RF-10 / AC-6 | Integración + banco |
| Foto nueva durante el seguimiento; transición con láser apagado | RF-11 / AC-7 | Integración |
| Foto duplicada y segundo teléfono | RF-11, RF-12 / CL-6 | Integración |
| Parada explícita; cerrar navegador no detiene | RF-12 / AC-10 | Integración + banco |
| Proceso de visión caído; watchdog apaga el láser | RF-9 / AC-6 | Banco |
| Torso como punto de apuntado y paralaje calibrado | NFR-2 / AC-9 | Unitario + banco |
| Suavidad sin saltos ni vibración | NFR-7 / AC-11 | Banco |

### 6.4 Checklist de banco

1. Stream de la ESP32-CAM estable, con resolución, FPS y latencia medidos.
2. Sentido de giro de ambos servos, topes calibrados y ausencia de forzado.
3. Movimiento suave, sin oscilación ni vibración con la cámara montada.
4. Láser apagado al arrancar, al detener y ante cualquier pérdida.
5. Láser encendido solo con apuntado centrado y estable.
6. Apagado del láser dentro de la ventana de 1–2 s tras la pérdida.
7. Tope de un eje detiene ambos y apaga el láser.
8. Desconexión de USB: watchdog, parada y apagado.
9. Desconexión de Wi-Fi: parada y recuperación solo con ambos enlaces.
10. Foto desde el teléfono: válida, inválida, duplicada y con dos clientes.
11. Paralaje cámara-láser y punto de torso; verificación sobre un blanco seguro, nunca hacia ojos.
12. Alimentación, masa y fijación de la ESP32-CAM; comportamiento del regulador.
13. Muerte del proceso del PC: el firmware apaga el láser.
14. Evidencia registrada: fecha, operador, versión, resultado y observaciones.

El LED sustituto solo sirve para pruebas parciales del lazo; la aceptación final exige el módulo láser real.

---

## 7. Trazabilidad RF → componente → verificación

| RF | Componentes | Verificación principal |
|---|---|---|
| RF-1 Foto válida inicia sesión | C8, C11, C13, C6, C17 | Unitario de validación; integración de subida; banco con teléfono |
| RF-2 Foto inválida se rechaza sin alterar | C8, C11, C13, C6 | Unitario e integración de motivos y estado previo |
| RF-3 Seguimiento del objetivo en video | C7, C10, C11, C12, C16 | Integración con video y serial simulados; banco en lazo cerrado |
| RF-4 Mayor similitud o abstención | C4, C11, C7 | Unitario de umbral, empate y abstención; integración |
| RF-5 Láser solo al centrar y estable | C3, C5, C12, C14, C16 | Unitario de política; integración; banco de seguridad |
| RF-6 Pérdida, parada y gracia 1–2 s | C5, C6, C7, C12, C14 | Unitario con reloj falso; integración; banco |
| RF-7 Reaparición y reencendido al centrar | C3, C5, C6, C7 | Unitario e integración de secuencia |
| RF-8 Tope de eje detiene ambos | C2, C6, C12, C14, C16 | Unitario de recorte; contrato; banco de topes |
| RF-9 Corte de video o USB | C9, C10, C12, C14, C15, C17 | Integración de salud; contrato; banco de desconexión |
| RF-10 Recuperación con ambos enlaces | C9, C12, C14, C15 | Integración de reconexión; banco |
| RF-11 Reemplazo de objetivo | C6, C8, C13 | Integración de subida durante seguimiento |
| RF-12 Estado visible y parada | C6, C13, C18 | Unitario de precedencia; integración; banco |

| NFR | Componentes | Verificación |
|---|---|---|
| NFR-1 Rendimiento | C7, C10, C9 | Medición de latencia y FPS en banco; descarte de frames en integración |
| NFR-2 Seguridad del láser | C5, C14 | Unitario de política, contrato defensivo y checklist con láser real |
| NFR-3 Seguridad mecánica | C2, C14, configuración | Unitario de límites y banco de topes |
| NFR-4 Privacidad | C8, C10, web, logs | Revisión de memoria y ausencia de escritura/red externa |
| NFR-5 Plataforma | C10, C17, C12 | Checklist de banco con teléfono, PC y ESP32-CAM |
| NFR-6 Idioma | UI y documentación | Revisión de textos y nombres |
| NFR-7 Movimiento | C14, C16 | Unitario de rampa y checklist de vibración |
| NFR-8 Verificación | Toda la pirámide | Cobertura, contrato y banco registrados |
| NFR-9 Configuración | Configuración de los tres componentes | Revisión de punto único y ausencia de pines dispersos |

---

## 8. Decisiones técnicas y trade-offs

| # | Decisión | Alternativa descartada | Justificación del trade-off |
|---|---|---|---|
| D1 | ESP32-CAM sobre la torreta envía video por Wi-Fi local; ESP32 principal por USB para motores y láser | Webcam del PC; cámara USB montada con cable al PC | La cámara debe girar con la torreta para que el encuadre coincida con el apuntado. La webcam del PC no cumple eso; una cámara USB con cable añade arrastre y un slip ring. El costo es una excepción a P1 y un enlace Wi-Fi con latencia y jitter que hay que medir. |
| D2 | Visión y decisión en el PC | Detección/embedding en la ESP32-CAM | El PC tiene más cómputo y permite tests puros; la ESP32-CAM queda como sensor. Se pierde autonomía, pero el sistema ya depende del PC para la web y el USB. |
| D3 | Video MJPEG sobre HTTP como opción base | RTSP; UDP crudo; WebRTC | MJPEG es simple, ampliamente soportado por ESP32-CAM y fácil de diagnosticar; el costo es mayor ancho de banda y decodificación. RTSP/UDP/WebRTC se descartan por complejidad y por no necesitar audio ni bidireccionalidad. La elección final se confirma por ADR tras medir latencia. |
| D4 | Hilos con buzón de último frame y descarte | Asyncio puro; multiproceso; cola FIFO | OpenCV y serial son bloqueantes; los hilos evitan envoltorios artificiales. El buzón de tamaño 1 garantiza latencia acotada. Asyncio complica el cómputo síncrono; multiproceso añade IPC y duplicación de la foto. |
| D5 | Core Python puro + `control_core` C++ puro, con vectores comunes | Una sola librería compartida por FFI; lógica solo en firmware; lógica solo en PC | Cada lado se testea en su toolchain sin hardware. Se acepta una duplicación pequeña y controlada; la conformidad se mide con vectores, no con código compartido. FFI acopla builds y complica el banco. |
| D6 | Protocolo serial de líneas JSON con versión y secuencia | Struct binario; COBS+CRC; ROS/serial ROS | JSON es legible, fácil de testear y suficiente para la carga; el costo es bytes y parseo, irrelevante a la tasa de control. Un binario ahorra ancho de banda pero fragiliza el contrato y el diagnóstico. COBS+CRC se reserva si las mediciones muestran corrupción. |
| D7 | Ángulos absolutos deseados + recorte y suavizado en firmware | Enviar deltas; control PID en el PC; dejar la autoridad al PC | Los absolutos son idempotentes y toleran un comando perdido. El firmware conserva la última palabra mecánica y de seguridad (P5). Los deltas acumulan error si se pierde un mensaje. |
| D8 | Reductor de sesión en el PC; seguridad defensiva en firmware | Estado solo en firmware; estado solo en el PC | La spec exige que el firmware sea autoridad de ángulos y láser, pero la sesión, la foto y la web viven en el PC. Separar responsabilidades evita duplicar la máquina de estados completa. |
| D9 | Política de láser calculada en el core y **revalidada** en firmware | Láser comandado directamente por el PC; temporizador solo en el PC | Dos barreras independientes: una decide, la otra impide un encendido inseguro por comando viejo, tope o reinicio. Cuesta duplicar la regla mínima; el beneficio es seguridad ante caída del PC. |
| D10 | La web consulta el estado por sondeo HTTP | WebSocket; SSE; streaming de video en la web | Sondeo es suficiente para estados de baja frecuencia, no añade dependencias ni conexiones persistentes y es trivial de testear. WebSocket/SSE se descartan por complejidad; el video no se muestra porque la spec no lo pide. |
| D11 | Servidor HTTP de la biblioteca estándar como opción base | Framework web externo (Flask/FastAPI) | P1 exige preferir la biblioteca estándar. Si el manejo de `multipart` o la robustez del servidor lo justifican, se añade una dependencia con ADR; no se decide ahora. |
| D12 | Configuración única por componente | Configuración global compartida; constantes dispersas | NFR-9 y P3 exigen un punto por componente. Una configuración global acopla firmware y PC y facilita filtrar secretos; constantes dispersas impiden calibrar topes y umbrales. |
| D13 | Detección de rostros + embedding + similitud coseno, con abstención | Template matching; Haar solo; tracker profundo de persona | El embedding tolera cambios de pose e iluminación mejor que el template matching y permite un umbral verificable. Un tracker profundo añade dependencia y opacidad. La biblioteca y el modelo exactos se fijan por ADR. |
| D14 | Reloj, frames, serial y aleatoriedad inyectados; vectores contractuales | Pruebas con cámara real y esperas; mocks ad hoc por test | P4 exige determinismo y cobertura. Los dobles permiten probar pérdida, timeout y reconexión sin hardware; el banco cubre lo físico. |
| D15 | Foto y frames solo en memoria | Caché en disco; base de datos local | P5 prohíbe persistencia no decidida y simplify el ciclo de vida. Una caché facilitaría depuración pero crea riesgo de almacenar imágenes de personas. |
| D16 | Una sesión y una foto activa por proceso | Varias sesiones o varias torretas | La spec limita el alcance a una persona y una torreta. La última foto válida prevalece y todas las webs ven lo mismo; multi-sesión multiplica estados y autoridad. |
| D17 | Watchdog de comando y tiempo máximo de láser en firmware | Confiar solo en el PC; apagado solo por software | El PC puede colgarse o desconectarse. El watchdog es la última barrera física; el costo es un timeout que debe superar el periodo normal de comandos y medirse en banco. |
| D18 | Punto de torso derivado del rostro y corregido por paralaje | Apuntar al centro del rostro; apuntar a la caja completa de la persona | La spec exige torso y descarta ojos. Derivarlo del rostro es posible con una sola detección; la corrección de paralaje se calibra en banco. Apuntar al rostro viola NFR-2; la caja completa exige un detector de persona adicional. |
| D19 | Coordenadas normalizadas y mapeo por FOV medido | Píxeles absolutos con supuestos de resolución | Las coordenadas normalizadas sobreviven a cambios de resolución del stream. El FOV real se mide; asumirlo introduce error de apuntado. |

---

## 9. Riesgos y mitigaciones

| Riesgo | Impacto | Mitigación | Verificación |
|---|---|---|---|
| Latencia y jitter del Wi-Fi | Seguimiento oscilante o láser tarde | Descarte de frames, control acotado, medir latencia y FPS | NFR-1 en banco |
| Cámara solidaria a la torreta | Realimentación, blur y oscilación | Banda muerta, rampa de servo, límite de velocidad | NFR-7 y estabilidad en banco |
| Paralaje cámara-láser | El haz no cae en el torso | Calibración y offset configurables | AC-9 en banco |
| Masa y par de la ESP32-CAM | Caída del soporte o pérdida de topes | Fijación rígida, recalibrar límites y rampa | Checklist de banco |
| Alimentación insuficiente | Reinicios de la cámara o del ESP32 | Regulación separada y medición de consumo | Checklist de banco |
| Falsos positivos de similitud | Seguir a otra persona | Umbral, abstención y confirmación por frames | AC-3 y banco |
| Falso negativo por baja luz | Pérdida constante | Tratarlo como pérdida, no forzar detección | CL-3 |
| Caída del proceso del PC | Láser encendido | Watchdog y tiempo máximo en firmware | RF-9 y banco |
| Comando viejo o duplicado | Movimiento o láser incorrecto | Secuencia, TTL y último comando válido | Contrato |
| Saturación del serial | Comandos atrasados | Mensajes pequeños, tasa acotada y telemetría resumida | Medición |
| Credenciales en el repo | Exposición de la LAN | Configuración local no versionada | Revisión |
| Cambio de resolución del stream | Mapeo incorrecto | Coordenadas normalizadas y FOV configurable | Unitario |
| Rearme del láser tras el máximo | Comportamiento indefinido | Se deja `[POR VERIFICAR]`; no se inventa | ADR de seguridad |

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
| ADR-010 | Parámetros de seguridad y control: tiempo máximo de láser, TTL, gracia, confirmación, banda muerta, ganancias y rearme | C3, C5, C14 y AC-4 |

Datos `[POR VERIFICAR]` que deben cerrarse con medición o decisión antes de la aceptación: tolerancia de centrado, umbral de similitud, criterios de rostro detectable, frames de confirmación, tiempo de interrupción, topes, formatos y tamaño de foto, latencia/FPS, navegadores y requisitos de PC/cámara, clase y potencia del láser, timeout del firmware, estabilidad del lazo, alimentación/masa/fijación, paralaje y rearme tras el tiempo máximo.

---

## 11. Fases de implementación

| Fase | Entregable | Salida |
|---|---|---|
| F0 | ADR y mediciones de banco, sin código de producto | Todos los `[POR VERIFICAR]` cerrados o con método de medición aprobado; `README.md` y constitución actualizados |
| F1 | Core Python puro + contrato serial con vectores | Cobertura de ramas 100 %; protocolo con vectores en verde |
| F2 | Firmware principal: serial, servos, láser, watchdog | `control_core` al 100 %; banco básico de movimiento y apagado |
| F3 | Stream de la ESP32-CAM y salud de video | Stream estable medido; corte y recuperación detectados |
| F4 | Adaptador de visión y geometría de torso | AC-1, AC-3 y AC-9 en integración/banco |
| F5 | Web de foto, estado y parada | AC-2, AC-7 y AC-10 en integración y teléfono |
| F6 | Lazo cerrado completo y matriz de fallos | AC-4, AC-5, AC-6 y AC-8 en integración/banco |
| F7 | Aceptación y checklist final | AC-1…AC-11 con evidencia y láser real |

Cada fase termina con sus tests en verde; no se avanza con tests fallidos o no ejecutados (P4).

---

## 12. Aprobación requerida

Este plan queda propuesto hasta que se aprueben:

1. La estructura de `src/` y `tests/` de la sección 3.1.
2. Las decisiones D1–D19, en especial MJPEG como base, hilos con descarte, JSON por líneas, ángulos absolutos con recorte en firmware y sondeo web.
3. El orden de fases y la exigencia de ADR antes de instalar cualquier dependencia.
4. La actualización de `README.md` y de la constitución para reflejar la ESP32-CAM y el enlace Wi-Fi.

No se ha escrito código de implementación, no se han creado carpetas y no se han ejecutado tests: el repositorio aún no tiene runner ni toolchain.
