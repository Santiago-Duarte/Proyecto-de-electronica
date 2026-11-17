# Spec 001 — Torreta de Seguimiento Facial con Láser

Estado: draft

## Contexto y objetivo
La torreta Pan-Tilt debe reconocer a una única persona a partir de una foto subida desde el navegador del celular y seguirla con una ESP32-CAM montada en la torreta, que transmite video en vivo por Wi-Fi local al PC; el ESP32 principal, conectado por USB, mueve los servos y controla el láser. El sistema apunta con el láser solo cuando el objetivo queda centrado. Resuelve la necesidad de demostrar, con hardware de bajo costo y sin servicios externos, un lazo completo de visión, control y actuación en tiempo real, manteniendo la seguridad del usuario y del soporte mecánico como condición de operación.

## Usuarios / actores
- **Operador:** persona con el teléfono que abre la web local, elige la foto y observa el estado del seguimiento.
- **Persona objetivo:** persona que aparece en la foto y a quien la torreta debe seguir; no interactúa con el sistema.
- **Torreta:** conjunto ESP32-CAM + PC + ESP32 principal + servos + láser que ejecuta el seguimiento.

## Historias de usuario
- H1: Como operador quiero subir o capturar una foto desde el navegador del móvil para indicar a la torreta a quién debe seguir.
- H2: Como operador quiero que la torreta siga solo a la persona de la foto e ignore al resto para que el apuntado no se confunda con otras personas.
- H3: Como operador quiero que el movimiento sea suave y que el láser se encienda solo al centrar para no dañar el soporte ni apuntar fuera del objetivo.
- H4: Como operador quiero ver en el móvil el estado de la sesión para saber si la torreta busca, sigue, perdió al objetivo o sufrió un fallo.
- H5: Como operador quiero que ante pérdida del objetivo, fallo de enlace o límite de giro la torreta se detenga y apague el láser para operar con seguridad.

## Requisitos funcionales (criterios de aceptación en EARS)
- RF-1: CUANDO el operador envía o captura una foto desde el navegador móvil, EL SISTEMA la valida y, si contiene exactamente un rostro detectable (frontal o semilateral, sin oclusión total y con tamaño suficiente para comparar; umbrales [POR VERIFICAR]), la establece como único objetivo activo e inicia el seguimiento; si la persona no está en el encuadre, permanece en la posición actual, con el láser apagado y el estado "buscando".
- RF-2: SI la foto no contiene exactamente un rostro detectable, está corrupta o su formato no es admitido (formatos, tamaño y resolución admitidos [POR VERIFICAR]), ENTONCES EL SISTEMA la rechaza, muestra el motivo en la web y conserva sin cambios el objetivo y el estado anteriores; si no había objetivo, permanece sin objetivo y no se mueve.
- RF-3: CUANDO el objetivo activo aparece en el video en vivo recibido de la ESP32-CAM por Wi-Fi local, EL SISTEMA lo identifica y envía los ángulos necesarios para mantener centrado el punto de apuntado (centro del torso, por debajo del rostro), sin seguir a las demás personas del encuadre.
- RF-4: SI hay candidatos en el encuadre, ENTONCES EL SISTEMA sigue al de mayor similitud que supere el umbral mínimo, sin pedir confirmación; SI ninguno supera el umbral, ENTONCES no sigue a ninguno, mantiene la posición y apaga el láser.
- RF-5: EL SISTEMA mantiene el láser encendido únicamente mientras el punto de apuntado del objetivo está centrado o dentro de la ventana de gracia por pérdida definida en RF-6; en cualquier otro estado el láser permanece apagado. Se considera centrado cuando el punto de apuntado está dentro de la zona central del encuadre; la tolerancia se fija en el plan [POR VERIFICAR]. El láser no se enciende mientras los servos estén en movimiento: solo con el apuntado estable y dentro de la zona central.
- RF-6: CUANDO se pierde la detección del objetivo, EL SISTEMA detiene los motores en el último ángulo conocido y apaga el láser entre 1 y 2 s después del último instante con objetivo confirmado. Se considera objetivo confirmado cuando la detección se mantiene durante un mínimo de frames consecutivos [POR VERIFICAR].
- RF-7: CUANDO el objetivo reaparece, EL SISTEMA reanuda el seguimiento y solo vuelve a encender el láser cuando el objetivo esté centrado.
- RF-8: SI el objetivo requiere un ángulo fuera de los límites calibrados de un eje, ENTONCES EL SISTEMA detiene ambos ejes, con el eje afectado en su tope, sin forzar el soporte y mantiene el láser apagado hasta que el objetivo vuelva a estar dentro de rango.
- RF-9: SI el video de la ESP32-CAM por Wi-Fi o el enlace USB con el ESP32 principal se interrumpe, ENTONCES EL SISTEMA detiene los motores y apaga el láser; la autoridad del estado del láser y de los ángulos permanece en el firmware del ESP32 principal y la web informa el fallo. Se considera interrupción cuando el dispositivo deja de estar disponible o no entrega frames durante el tiempo configurado [POR VERIFICAR]. SI el proceso de visión se detiene, ENTONCES la sesión termina y el firmware apaga el láser por timeout.
- RF-10: CUANDO se restablecen el video de la ESP32-CAM y el enlace USB con el ESP32 principal, EL SISTEMA reanuda el seguimiento con la foto de referencia activa, sin reutilizar ángulos de una sesión anterior, y avisa de la recuperación en la web. El primer apuntado parte de la posición actual y respeta los topes.
- RF-11: CUANDO el operador envía una foto nueva válida durante el seguimiento, EL SISTEMA reemplaza el objetivo en el acto, sin reiniciar la sesión ni exigir pasos adicionales. Durante la transición apaga el láser y mantiene la posición hasta confirmar el nuevo objetivo; si el enlace está caído, la foto válida más reciente queda como objetivo activo y se usa al reanudar.
- RF-12: EL SISTEMA muestra en la web el estado actual de la sesión (sin objetivo, buscando, siguiendo, objetivo perdido, límite alcanzado, error de video o enlace USB, o detenido) y permite al operador detener la sesión; al detenerla, apaga el láser, termina el seguimiento y queda sin objetivo. Cerrar el navegador no detiene el seguimiento. La precedencia es: detenido, error de video o enlace USB, objetivo perdido, límite alcanzado, siguiendo, buscando, sin objetivo; se muestra un solo estado.

## Requisitos no funcionales
- NFR-1 Rendimiento: la cadena de seguimiento debe sostener la tasa de frames del video de la ESP32-CAM sin acumular retraso creciente; se mide como latencia extremo a extremo y tasa de frames sostenida, y los valores máximos se fijan antes de aprobar el plan [POR VERIFICAR].
- NFR-2 Seguridad del láser: apagado por defecto y en todo estado no centrado, salvo la ventana de gracia de RF-6; el punto de apuntado es el centro del torso, por debajo del rostro, y el haz nunca se dirige a los ojos; el apagado por pérdida de objetivo se verifica en banco. El láser tiene un tiempo máximo de encendido continuo [POR VERIFICAR] y se apaga al alcanzarlo.
- NFR-3 Seguridad mecánica: los topes de cada eje viven en un único punto de configuración y ningún ángulo enviado los excede.
- NFR-4 Privacidad: sin servicios en la nube ni telemetría; foto y frames viven solo en memoria y no salen de la LAN; el video de la ESP32-CAM viaja únicamente por la Wi-Fi local.
- NFR-5 Plataforma: navegador web de un teléfono en la misma red Wi-Fi que el PC; ESP32-CAM con video por Wi-Fi local; ESP32 principal por USB; versiones, modelos y navegadores admitidos [POR VERIFICAR].
- NFR-6 Idioma: interfaz, mensajes y documentación en español; identificadores y código en inglés.
- NFR-7 Movimiento: los servos se mueven de forma suavizada, sin saltos que golpeen o vibren el soporte; el criterio numérico se fija en el plan [POR VERIFICAR].
- NFR-8 Verificación: core puro con cobertura de ramas completa, protocolo PC↔ESP32 principal probado contra un stub de puerto serial, video Wi-Fi de la ESP32-CAM validado en banco y hardware validado con checklist reproducible.
- NFR-9 Configuración: pines, identificador de la ESP32-CAM, límites de servo y estado del láser viven en un único punto de configuración por componente.

## Casos límite
- **CL-1 Foto inválida o ambigua:** sin rostro, varios rostros, archivo corrupto o formato no admitido — se rechaza con motivo y el objetivo y el estado anteriores quedan intactos (RF-2).
- **CL-2 Personas parecidas:** dos o más candidatos sobre el umbral, incluso con similitud cercana — se sigue al de mayor similitud sin intervención del operador; si ninguno supera el umbral, no se sigue a nadie y el láser queda apagado (RF-4).
- **CL-3 Pérdida y reaparición:** oclusión breve, persona agachada, salida del encuadre o baja luz que impide la detección mientras la ESP32-CAM entrega frames — se mantiene el último ángulo, el láser se apaga entre 1 y 2 s y el seguimiento se reanuda al reaparecer (RF-6, RF-7).
- **CL-4 Fuera del rango de giro:** el objetivo exige un ángulo mayor que el tope calibrado — se detienen ambos ejes, el afectado en su límite, sin forzar el soporte, y el láser queda apagado hasta que el objetivo vuelva a rango (RF-8).
- **CL-5 Corte de video Wi-Fi o USB:** desconexión del video de la ESP32-CAM o del enlace USB con el ESP32 principal — parada segura, láser apagado, aviso en la web y reanudación automática al restablecerse ambos enlaces (RF-9, RF-10).
- **CL-6 Nueva foto o varios teléfonos:** subida válida durante el seguimiento o una segunda persona abriendo la web — una sola sesión y una sola foto activa; la última foto válida recibida prevalece, todas las webs muestran el mismo estado y una foto inválida no altera el objetivo activo. Una subida duplicada (mismo archivo) se trata como reemplazo válido sin cambio observable (RF-11, RF-12).

## Fuera de alcance
- Aplicación móvil nativa para Android o iOS.
- Análisis de video en la ESP32-CAM; solo captura y transmisión por Wi-Fi local.
- Seguimiento simultáneo de varias personas o control de más de una torreta.
- Persistencia, historial o grabación de fotos, frames o video.
- Reconocimiento de la persona entre sesiones o cualquier identificación biométrica más allá de comparar con la foto activa.
- Control de los motores o del láser por Wi-Fi; el ESP32 principal sigue exclusivamente por USB y la ESP32-CAM solo transmite video.
- Autenticación de la web o acceso desde fuera de la LAN.
- Calibración o ajuste manual de ángulos desde la web.
- Reconocimiento garantizado en condiciones de baja luz, con varias cámaras o con cambio de cámara en caliente.

## Criterios de aceptación
- AC-1 (RF-1, RF-3): con una foto válida y dos personas en escena, la torreta sigue únicamente a la persona de la foto.
- AC-2 (RF-2): una foto sin rostro, con varios rostros o corrupta se rechaza con mensaje y no altera el objetivo activo.
- AC-3 (RF-4): con dos personas sobre el umbral, la torreta sigue a la de mayor similitud sin intervención del operador; si ninguna supera el umbral, no sigue a ninguna y apaga el láser.
- AC-4 (RF-5, RF-6, RF-7): el láser solo está encendido con el punto de apuntado centrado, se apaga entre 1 y 2 s tras la pérdida y no se reenciende hasta volver a centrar.
- AC-5 (RF-8): al alcanzar el tope de un eje se detienen ambos ejes, el afectado en su tope, sin exceder el valor calibrado, sin forzar el soporte y con el láser apagado hasta que el objetivo vuelva a rango.
- AC-6 (RF-9, RF-10): un corte de video Wi-Fi o USB produce parada y aviso; cuando ambos enlaces se restablecen, el seguimiento reanuda con la foto activa; si el proceso de visión termina, la sesión termina y el láser se apaga.
- AC-7 (RF-11, RF-12): una foto nueva válida reemplaza al objetivo en el acto, con el láser apagado durante la transición, y el estado de la sesión es visible en la web.
- AC-8 (NFR-2, NFR-4): el láser inicia apagado y ninguna imagen sale de la LAN ni se persiste; el video de la ESP32-CAM circula solo por la Wi-Fi local.
- AC-9 (NFR-2): el punto de apuntado se mantiene en el torso, por debajo del rostro, y el haz no se dirige a los ojos.
- AC-10 (RF-12): la parada explícita apaga el láser y deja la sesión sin objetivo; cerrar el navegador no detiene el seguimiento.
- AC-11 (NFR-7): el movimiento no presenta saltos que golpeen o vibren el soporte, según el checklist de banco; el criterio numérico queda [POR VERIFICAR].

## Criterios de finalización
- Todos los RF con prueba automatizada o checklist de banco en verde; cobertura de ramas completa en el core y protocolo validado contra el stub serial.
- Demo manual del flujo completo y de los casos de fallo: foto válida → seguimiento → láser al centrar → pérdida → reanudación; foto inválida; límite de giro; corte y reconexión; reemplazo de objetivo; parada explícita.
- Checklist de banco registrado para video Wi-Fi de la ESP32-CAM, servos y láser con el módulo láser real; el LED solo se usa para pruebas parciales del resto del lazo.
- `plan.md` con dependencias y versiones fijadas por ADR; las dudas de decisión se resuelven antes de aprobar y los datos externos [POR VERIFICAR] se cierran antes de implementar, con criterio de medición.

## Dudas abiertas
- [POR VERIFICAR] Tiempo máximo de encendido continuo del láser; NFR-2 exige que exista y se apague al alcanzarlo.
- [POR VERIFICAR] Tolerancia numérica de centrado, umbral mínimo de similitud y criterios numéricos de rostro detectable.
- [POR VERIFICAR] Tiempo para declarar interrupción de video Wi-Fi o USB, mínimo de frames consecutivos para confirmar objetivo y estabilidad previa al encendido.
- [POR VERIFICAR] Valores concretos de los topes de cada eje tras la calibración en banco.
- [POR VERIFICAR] Formatos, tamaño y resolución admitidos para la foto.
- [POR VERIFICAR] Latencia máxima aceptable y tasa de frames mínima para que el seguimiento se perciba continuo.
- [POR VERIFICAR] Navegadores y versiones móviles soportadas, y requisitos mínimos del PC y de la ESP32-CAM.
- [POR VERIFICAR] Clase, potencia, distancia de seguridad y comportamiento del módulo láser real.
- [POR VERIFICAR] Timeout de apagado autónomo del firmware al perderse el enlace o el proceso de visión.
- [POR VERIFICAR] Latencia, resolución, tasa de frames y estabilidad del video Wi-Fi de la ESP32-CAM.
- [POR VERIFICAR] Estabilidad del lazo de control con la ESP32-CAM solidaria a la torreta: oscilación, latencia y criterio de ganancia/tolerancia.
- [POR VERIFICAR] Alimentación, fijación, masa de la ESP32-CAM y su efecto en los topes y en la suavidad del movimiento.
- [POR VERIFICAR] Calibración del paralaje entre la ESP32-CAM y el láser para que el punto de apuntado coincida con el torso y no con los ojos.
- [POR VERIFICAR] Modelo exacto de ESP32-CAM, framework de su firmware y ADR que justifique el enlace Wi-Fi frente a la constitución P1.

## Clarificaciones de auditoría

Procesados los 31 hallazgos del reporte; la tabla mantiene el orden de severidad. Decisiones nuevas que deben incorporarse al contexto base: punto de apuntado en el torso; abstención bajo el umbral de similitud; espera sin barrido si el objetivo no aparece; parada explícita con sesión independiente de la web; parada de ambos ejes al alcanzar un tope. No queda ningún hallazgo pendiente de decisión.

| Hallazgo | Estado | Cómo se cerró |
|---|---|---|
| H-01 Láser vs ventana de gracia | Cerrado | RF-5 admite la gracia de RF-6; RF-6 fija 1–2 s (entrevista 2). |
| H-02 Punto de apuntado | Cerrado | Mira en el centro del torso (decisión); clase y potencia [POR VERIFICAR]. |
| H-03 Umbral de similitud | Dudas abiertas | Abstención bajo el umbral (decisión); valor [POR VERIFICAR]. |
| H-04 Autoridad con USB caído | Cerrado | RF-9: el firmware conserva la autoridad de ángulos y láser (P5). |
| H-05 Recuperación OR | Cerrado | RF-10 exige video Wi-Fi y USB restablecidos. |
| H-06 Margen 1–2 s | Cerrado | RF-6 y AC-4 fijan 1–2 s (entrevista 2). |
| H-07 Centrado | Dudas abiertas | Zona central definida; tolerancia [POR VERIFICAR]. |
| H-08 Rostro detectable | Dudas abiertas | Criterio cualitativo; umbrales [POR VERIFICAR]. |
| H-09 Interrupción | Dudas abiertas | Dispositivo no disponible o sin frames; tiempo [POR VERIFICAR]. |
| H-10 Sin forzar soporte | Dudas abiertas | AC-5 exige no exceder el tope calibrado; topes [POR VERIFICAR]. |
| H-11 Objetivo ausente al inicio | Cerrado | Espera sin moverse, láser apagado (decisión). |
| H-12 Proceso o navegador | Cerrado | RF-9 y RF-12: proceso caído termina la sesión; cerrar la web no la detiene. |
| H-13 Reemplazo de objetivo | Cerrado | RF-11: láser apagado y posición mantenida; la foto válida más reciente prevalece. |
| H-14 Fin de sesión | Cerrado | RF-12 y AC-10: parada explícita y estado detenido. |
| H-15 Objetivo confirmado | Dudas abiertas | Mínimo de frames consecutivos [POR VERIFICAR]. |
| H-16 Láser en movimiento | Cerrado | RF-5: no enciende con los servos en movimiento. |
| H-17 AC y dudas abiertas | Cerrado | La finalización exige resolver dudas antes de aprobar (P2). |
| H-18 LED sustituto | Cerrado | La finalización exige el módulo láser real; LED solo pruebas parciales (P4). |
| H-19 Modelo de estados | Cerrado | RF-12: estados y precedencia definidos. |
| H-20 Tope de un eje | Cerrado | RF-8 y AC-5: se detienen ambos ejes (decisión). |
| H-21 Estado previo tras foto inválida | Cerrado | RF-2 conserva objetivo y estado. |
| H-22 Reanudación y servos | Cerrado | RF-10 no reutiliza ángulos; parte de la posición actual. |
| H-23 Varios teléfonos | Cerrado | CL-6: la última foto válida prevalece; estado único. |
| H-24 NFR-1 | Dudas abiertas | Medición definida; valores [POR VERIFICAR]. |
| H-25 Suavidad | Dudas abiertas | AC-11; criterio numérico [POR VERIFICAR]. |
| H-26 Formatos de foto | Dudas abiertas | RF-2 y AC: formatos y límites [POR VERIFICAR]. |
| H-27 P3 configuración | Cerrado | NFR-9: pines, ESP32-CAM, límites y láser en un punto por componente. |
| H-28 Demo de finalización | Cerrado | Incluye foto inválida, límite, corte, reemplazo y parada. |
| H-29 Baja luz | Cerrado | CL-3: se trata como pérdida si la ESP32-CAM entrega frames. |
| H-30 Subida duplicada | Cerrado | CL-6: mismo archivo; reemplazo sin cambio observable. |
| H-31 Captura | Cerrado | RF-1 incluye envío o captura. |

## Actualización de hardware (2026-11-16)

La cámara pasa de la webcam del PC a una ESP32-CAM montada en la torreta, con video por Wi-Fi local; el ESP32 principal sigue por USB y controla motores y láser. Afecta RF-3, RF-9, RF-10, RF-12, NFR-4, NFR-5, NFR-9, CL-5, el fuera de alcance y AC-6/AC-8. Requiere ADR y actualización del `README.md`, que aún describe la cámara del PC.
