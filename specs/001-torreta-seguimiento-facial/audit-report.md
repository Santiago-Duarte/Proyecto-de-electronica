# Reporte de auditoría — Spec 001

**Objeto:** `specs/001-torreta-seguimiento-facial/spec.md` (Estado: draft)
**Insumos:** `docs/constitution.md`, `README.md`, `AGENTS.md`, respuestas de la entrevista (6), `specs/README.md`.
**Fecha:** 2026-11-16
**Alcance:** auditoría documental; no existen `plan.md`, `tasks.md` ni código que auditar.

**Veredicto:** la especificación no es apta para aprobación en su estado actual. Se registran 4 hallazgos críticos, 14 altos, 11 medios y 2 bajos. Los críticos afectan la seguridad del láser, la selección de la persona y la autoridad de parada ante fallo de enlace.

| Severidad | Nº | IDs |
|---|---|---|
| Crítica | 4 | H-01–H-04 |
| Alta | 14 | H-05–H-18 |
| Media | 11 | H-19–H-29 |
| Baja | 2 | H-30–H-31 |

## Hallazgos críticos

### H-01 — El invariante del láser y la ventana de gracia se contradicen
- **Ubicación:** RF-5, RF-6, RF-7, AC-4.
- **Tipo:** Contradicción lógica / estado inconsistente.
- **Descripción:** RF-5 exige láser apagado en "cualquier otro estado" distinto de objetivo centrado; RF-6 permite hasta 2 s encendido tras perder la detección. Durante esa gracia el objetivo no está confirmado como centrado y la spec no declara RF-6 como excepción de RF-5.
- **Impacto:** dos lecturas incompatibles del mismo estado; AC-4 solo fija el límite superior y no resuelve cuál regla prevalece.

### H-02 — El punto de apuntado no está definido y NFR-2 afirma una garantía no derivable
- **Ubicación:** NFR-2, RF-3, RF-5, objetivo.
- **Tipo:** Ambigüedad / seguridad.
- **Descripción:** el sistema apunta a la persona cuando está centrada, pero no se define a qué zona del cuerpo o del rostro apunta el haz. NFR-2 declara "nunca apunta a los ojos" sin que ningún RF fije una zona de mira ni una distancia. La clase y potencia del láser no están registradas.
- **Impacto:** la garantía de seguridad no es verificable y puede ser incompatible con el apuntado al rostro que describe el objetivo.

### H-03 — No hay umbral mínimo de similitud y el sistema puede seguir a un desconocido
- **Ubicación:** RF-4, CL-2, H2, objetivo.
- **Tipo:** Ambigüedad lógica / alcance.
- **Descripción:** RF-4 ordena seguir al candidato de mayor similitud sin confirmación cuando "varias personas se parecen"; CL-2 llama "compatibles" a los candidatos sin definir el término. Si la persona objetivo no está y hay alguien con similitud baja pero la más alta, la regla selecciona a esa persona.
- **Impacto:** contradice "una única persona seleccionada" y H2; AC-3 no distingue "parecido" de "no parecido".

### H-04 — La autoridad de parada con el USB caído no está definida; posible conflicto con P3/P5
- **Ubicación:** RF-9, RF-6, constitución P3 y P5.
- **Tipo:** Estado inconsistente / constitución.
- **Descripción:** si el enlace USB se interrumpe, el PC no puede enviar la orden de detener motores ni de apagar el láser. RF-9 atribuye ambas acciones a "EL SISTEMA" sin indicar si las ejecuta el firmware o el PC. P5 sitúa la autoridad de ángulos y láser en el firmware y P3 separa el control del framing USB.
- **Impacto:** el requisito puede ser inejecutable tal como está redactado; no se sabe qué componente garantiza la parada segura.

## Hallazgos altos

### H-05 — La recuperación se activa con una condición OR y produce un estado imposible
- **Ubicación:** RF-9, RF-10, AC-6.
- **Tipo:** Estado inconsistente.
- **Descripción:** "CUANDO se restablece la cámara o el enlace USB" permite reanudar con solo uno de los dos restablecido. Sin cámara no hay detección; sin USB no hay envío de ángulos. El estado "reanudado" no es alcanzable en ninguno de los dos casos parciales.
- **Impacto:** AC-6 no se puede verificar en la recuperación parcial y no existe estado definido para esa ventana.

### H-06 — La decisión de margen 1–2 s quedó reducida a un máximo de 2 s
- **Ubicación:** RF-6, AC-4, respuesta de entrevista 2.
- **Tipo:** Contradicción con una decisión cerrada.
- **Descripción:** la respuesta fue "margen de 1 a 2 segundos"; la spec solo impone "como máximo 2 s", sin mínimo. Un apagado inmediato (0 s) cumple RF-6 y AC-4, y la tolerancia a una oclusión breve no queda obligada.
- **Impacto:** el comportamiento decidido no es el que la spec exige; una implementación que viola la intención pasa la aceptación.

### H-07 — "Centrado" carece de tolerancia, referencia e histéresis
- **Ubicación:** RF-5, RF-7, AC-4.
- **Tipo:** Ambigüedad / verificabilidad.
- **Descripción:** no se define la zona de centrado (píxeles, porcentaje del encuadre o ángulo), si se mide sobre el rostro o la persona, ni histéresis. Un umbral de 0 produce parpadeo del láser; uno amplio lo enciende antes de tiempo.
- **Impacto:** RF-5 y AC-4 no son verificables; el encendido del láser queda a criterio de la implementación.

### H-08 — "Rostro detectable" y "compatible" no tienen criterio; "detectable" no está marcado como pendiente
- **Ubicación:** RF-1, RF-2, RF-4, CL-1, CL-2, duda abierta 2.
- **Tipo:** Ambigüedad / verificabilidad.
- **Descripción:** "exactamente un rostro detectable" no define tamaño mínimo, pose, oclusión ni resolución; "compatible" en CL-2 tampoco. El umbral de similitud está en dudas abiertas, pero "detectable" no lleva [POR VERIFICAR]. Una foto con un rostro pequeño al fondo se rechaza y una con el rostro objetivo ocluido también.
- **Impacto:** RF-1, RF-2 y RF-4 no son verificables; el comportamiento depende de la implementación.

### H-09 — "Se interrumpe" no distingue pérdida transitoria de desconexión
- **Ubicación:** RF-9, RF-10, AC-6.
- **Tipo:** Ambigüedad / caso límite.
- **Descripción:** no se define si un frame perdido, una racha de frames o la desaparición del dispositivo cuentan como interrupción, ni el tiempo para declarar el fallo y la recuperación. La frontera con RF-6 (pérdida de detección) queda difusa: una caída de cámara puede leerse como pérdida de 2 s o como interrupción con parada.
- **Impacto:** paradas y reanudaciones no deterministas; AC-6 no es reproducible.

### H-10 — "Sin forzar el soporte" no es medible
- **Ubicación:** RF-8, AC-5, NFR-3.
- **Tipo:** Ambigüedad / verificabilidad.
- **Descripción:** no hay criterio observable (ángulo máximo, corriente, tope mecánico, ausencia de bloqueo) que permita decidir si el soporte se está forzando.
- **Impacto:** AC-5 no es verificable y el riesgo mecánico que motiva RF-8 no queda cubierto por una prueba.

### H-11 — Los estados iniciales no están definidos
- **Ubicación:** RF-1, RF-3, RF-12, respuesta de entrevista 1.
- **Tipo:** Caso límite / estado inconsistente.
- **Descripción:** RF-1 inicia el seguimiento aunque la persona no esté en el encuadre; RF-12 ofrece "buscando" pero no define si los motores se mueven. La entrevista descartó el barrido para la pérdida, no para el inicio. Tampoco se define la posición inicial de los servos ni el estado si faltan cámara o ESP32 al abrir la web.
- **Impacto:** el primer estado del sistema y el comportamiento motor inicial quedan indeterminados; "buscando" sugiere una acción no especificada.

### H-12 — La caída del proceso de visión o el cierre del navegador no están contemplados
- **Ubicación:** NFR-4, RF-10, RF-12.
- **Tipo:** Caso límite / estado inconsistente / constitución P5.
- **Descripción:** RF-10 reanuda "con la misma foto" tras un fallo de enlace, pero NFR-4 prohíbe persistirla. Si el proceso del PC se reinicia, la foto en memoria se pierde. No se define si el seguimiento depende de la web abierta ni qué pasa con ángulos y láser si el proceso muere.
- **Impacto:** contradicción potencial entre "reanudar con la misma foto" y "sin persistencia"; estado final del láser y los servos indeterminado.

### H-13 — El reemplazo de objetivo no define transición ni carrera con la recuperación
- **Ubicación:** RF-11, RF-5, RF-10, CL-6.
- **Tipo:** Estado inconsistente.
- **Descripción:** al reemplazar la foto "en el acto" no se define si el láser se apaga durante la transición, si se mantiene el último ángulo o se recentra, ni qué pasa si la nueva foto llega con el enlace caído y luego RF-10 reanuda "con la misma foto" (¿la anterior o la nueva?). "Sin reiniciar la sesión" no aclara el estado intermedio.
- **Impacto:** el láser puede quedar encendido apuntando a un objetivo ya reemplazado; el resultado depende del orden de eventos.

### H-14 — No existe forma definida de detener o terminar la sesión
- **Ubicación:** RF-11, RF-12, CL-6, fuera de alcance.
- **Tipo:** Caso límite / seguridad.
- **Descripción:** la entrevista 6 eligió reemplazo inmediato en lugar de detención y la spec no incluye un control de parada. No se define qué termina una sesión, qué pasa si el operador quiere dejar de apuntar ni si el láser puede quedar habilitado indefinidamente mientras el objetivo esté centrado. RF-12 no tiene estado "detenido".
- **Impacto:** no hay salida operativa segura ni estado final; el operador no puede cesar el apuntado dentro de la spec.

### H-15 — "Objetivo confirmado" no tiene criterio temporal y el timeout de 2 s es evadible
- **Ubicación:** RF-6, RF-7.
- **Tipo:** Ambigüedad / caso límite.
- **Descripción:** no se define si "confirmado" exige un frame, varios consecutivos o una similitud mínima. Una detección aislada cada menos de 2 s reinicia el plazo y el láser puede permanecer encendido de forma intermitente sin objetivo estable.
- **Impacto:** el apagado garantizado a los 2 s no se cumple en la práctica; la prueba depende de la definición oculta.

### H-16 — El láser se enciende sin exigir estabilidad del apuntado
- **Ubicación:** RF-5, NFR-7.
- **Tipo:** Caso límite / seguridad.
- **Descripción:** RF-5 enciende al alcanzar "centrado", sin exigir que los servos estén detenidos o que el apuntado esté estable. El haz puede encenderse durante la corrección y barrer la escena.
- **Impacto:** encendido en movimiento y riesgo de apuntar fuera del objetivo.

### H-17 — Varios AC dependen de decisiones abiertas y la finalización permite "aceptarlas"
- **Ubicación:** AC-3, AC-4, AC-5, dudas abiertas, criterios de finalización, constitución P2.
- **Tipo:** Constitución / gobernanza.
- **Descripción:** AC-3, AC-4 y AC-5 no son verificables sin umbrales que siguen abiertos. La finalización admite dudas "resueltas o aceptadas explícitamente", lo que permite cerrar sin definirlas. P2 exige criterios verificables y detener el desarrollo si falta una decisión.
- **Impacto:** se podría aprobar e implementar con AC no verificables; la gobernanza de la spec queda debilitada.

### H-18 — La finalización con LED sustituto no valida el láser
- **Ubicación:** Criterios de finalización, NFR-2, AC-8, `README.md`.
- **Tipo:** Constitución P4 / verificación.
- **Descripción:** la finalización acepta "el módulo láser o el LED sustituto". Con LED no se verifican el apuntado, la clase, la distancia ni "nunca a los ojos". La spec no distingue qué AC quedan pendientes hasta tener el módulo real.
- **Impacto:** se puede declarar terminada la spec sin validar el actuador de seguridad; P4 exige checklist de banco del láser.

## Hallazgos medios

### H-19 — El modelo de estados de RF-12 no es exhaustivo, excluyente ni jerárquico
- **Ubicación:** RF-12, RF-2, RF-8, RF-9, RF-10, RF-11.
- **Tipo:** Estado inconsistente.
- **Descripción:** la lista "sin objetivo, buscando, siguiendo, objetivo perdido o error de enlace" no define transiciones ni precedencia. No hay estado para foto rechazada, límite alcanzado, reemplazo o recuperación. "Sin objetivo" y "buscando" se solapan, y con pérdida y error de enlace simultáneos no se sabe cuál mostrar.
- **Impacto:** RF-12 no es verificable y la UI puede mostrar estados contradictorios.

### H-20 — RF-8 no define la salida del límite y la duda abierta 4 contradice su texto
- **Ubicación:** RF-8, AC-5, duda abierta 4.
- **Tipo:** Estado inconsistente / duda.
- **Descripción:** se define la entrada al tope pero no la salida: si el objetivo vuelve dentro de rango, ¿se reanuda y se reenciende el láser? ¿Se considera perdido? RF-8 dice "ese eje" (singular), mientras la duda 4 plantea detener todo el apuntado, como si no estuviera decidido.
- **Impacto:** comportamiento indeterminado justo en el límite, donde existe riesgo mecánico.

### H-21 — RF-2 no define qué se conserva cuando el estado previo no es "siguiendo"
- **Ubicación:** RF-2, RF-12.
- **Tipo:** Estado inconsistente.
- **Descripción:** "conserva el objetivo activo anterior (o el estado sin objetivo)" asume dos casos, pero el estado previo puede ser perdido, error de enlace o límite. No se define si la foto rechazada congela ese estado, vuelve a "buscando" o mantiene el objetivo y sigue operando.
- **Impacto:** estado inconsistente tras una subida inválida.

### H-22 — La reanudación asume un estado de servos que puede no ser válido
- **Ubicación:** RF-7, RF-10, RF-11.
- **Tipo:** Estado inconsistente.
- **Descripción:** no se define si el firmware conserva la posición, si el ESP32 se reinició ni si el último ángulo conocido sigue siendo válido. Tampoco se distingue ángulo comandado de ángulo aplicado.
- **Impacto:** movimientos bruscos o apuntado a un ángulo obsoleto tras la reconexión.

### H-23 — La concurrencia entre varios teléfonos no está resuelta más allá del reemplazo
- **Ubicación:** CL-6, RF-11, RF-12.
- **Tipo:** Caso límite / concurrencia.
- **Descripción:** "una sola sesión" no define el orden ante dos subidas simultáneas, qué cliente gana, si todos ven el mismo estado ni si una subida inválida de un cliente afecta al objetivo de otro.
- **Impacto:** estado no determinista con varios clientes.

### H-24 — NFR-1 no es medible ni tiene método de observación
- **Ubicación:** NFR-1, RF-3.
- **Tipo:** Ambigüedad / verificabilidad.
- **Descripción:** "sin acumular retraso creciente" no fija tasa de frames, latencia, ventana de medición ni tolerancia. El valor [POR VERIFICAR] no cubre la falta de criterio de medición.
- **Impacto:** NFR no verificable y sin AC de rendimiento asociado.

### H-25 — NFR-7 y H3 no tienen criterio de aceptación trazable
- **Ubicación:** NFR-7, H3, RF-6.
- **Tipo:** Ambigüedad / trazabilidad.
- **Descripción:** NFR-7 deja el criterio "por fijar" y no hay AC que cubra la suavidad; H3 no tiene RF/AC trazable. "Sin saltos que golpeen o vibren" es cualitativo. RF-6 ordena detener motores sin decir si la parada debe ser suavizada.
- **Impacto:** uno de los objetivos del alcance ("movimiento suavizado") no es verificable.

### H-26 — Los formatos y límites de la foto no están definidos ni marcados
- **Ubicación:** RF-2, CL-1.
- **Tipo:** Ambigüedad / verificabilidad.
- **Descripción:** se menciona "formato no admitido" sin enumerar formatos, tamaño máximo, resolución mínima ni orientación EXIF, y sin marca [POR VERIFICAR].
- **Impacto:** RF-2 no es verificable en el límite; la validación queda a criterio de la implementación.

### H-27 — P3 exige punto único de configuración para pines, cámara y láser; solo los límites están cubiertos
- **Ubicación:** NFR-3, constitución P3.
- **Tipo:** Constitución.
- **Descripción:** NFR-3 solo hace verificable el punto único de los límites de servo. Pines, índice de cámara y estado del láser no aparecen como requisito ni como exclusión.
- **Impacto:** cobertura incompleta de P3; el plan podría cumplirlo, pero la spec no lo exige.

### H-28 — La demo de finalización no cubre todos los AC
- **Ubicación:** Criterios de finalización, AC-2, AC-5, AC-6, AC-7.
- **Tipo:** Cobertura de aceptación.
- **Descripción:** la demo listada solo recorre el flujo feliz y la pérdida; no incluye foto inválida, límite, corte/reconexión ni reemplazo.
- **Impacto:** se podría cerrar con ACs no demostrados.

### H-29 — Los límites ambientales excluidos no tienen comportamiento de fallo definido
- **Ubicación:** Fuera de alcance, RF-6, RF-9.
- **Tipo:** Caso límite / alcance.
- **Descripción:** se excluye baja luz, varias cámaras y cambio en caliente, pero no se define cómo se manifiesta ese límite si ocurre: ¿pérdida de objetivo (2 s), error o simple ausencia de detección?
- **Impacto:** comportamiento indefinido en condiciones reales; la exclusión de alcance no define el estado resultante.

## Hallazgos bajos

### H-30 — "Subida duplicada" es ambigua
- **Ubicación:** CL-6.
- **Tipo:** Ambigüedad.
- **Descripción:** no se aclara si "duplicada" es el mismo archivo byte a byte, la misma persona en otra foto o un reintento de red. "No cambia el comportamiento observable" no es comprobable sin esa definición.
- **Impacto:** criterio de caso límite no verificable.

### H-31 — La captura desde el navegador no está cubierta por RF-1
- **Ubicación:** RF-1, H1, alcance base.
- **Tipo:** Cobertura / alcance.
- **Descripción:** el alcance y H1 dicen "subir o capturar", pero RF-1 solo habla de "envía una foto". No se define si la captura con la cámara del teléfono es un camino equivalente ni qué pasa si se deniega el permiso.
- **Impacto:** cobertura funcional incompleta del alcance declarado.

## Mapa por categoría

- **Contradicciones lógicas:** H-01, H-06, H-17.
- **Estados inconsistentes:** H-01, H-04, H-05, H-11, H-12, H-13, H-19, H-20, H-21, H-22.
- **Casos límite no documentados:** H-09, H-11, H-12, H-14, H-15, H-16, H-23, H-29, H-30, H-31.
- **Ambigüedades y verificabilidad:** H-02, H-03, H-07, H-08, H-10, H-15, H-24, H-25, H-26.
- **Constitución:** P2 en H-17; P3 en H-04 y H-27; P4 en H-18 y H-28; P5 en H-04 y H-12. P1 (stack/versiones) y P6 (idioma) no presentan hallazgos en el texto.

## Límites de la auditoría

- No existen `plan.md`, `tasks.md`, código, manifiestos ni runner: no se ejecutaron pruebas ni se validó comportamiento real.
- La auditoría es documental y se limita a la spec 001, la constitución, el README y las respuestas de la entrevista.
- Los hallazgos sobre dudas abiertas se valoran como bloqueo de aprobación, no como incumplimiento consumado, dado que la spec está en estado draft.



