# Decisiones (ADR)

<!-- Una entrada por decision: fecha, contexto, decision, consecuencias, spec relacionada. -->

## ADR-011 — Láser siempre encendido con corte físico

- **Fecha:** 2026-11-17
- **Contexto:** se simplifica el circuito del láser. El módulo funciona con batería y un botón físico mantenido presionado con cinta; no se añade GPIO, relé ni circuito de control.
- **Decisión:** el láser permanece encendido mientras la batería esté conectada y el botón esté presionado. El firmware y la web no controlan su encendido ni su apagado; el único corte es físico. En pérdida, parada, tope o error de enlace el láser sigue encendido y solo se detienen los motores.
- **Consecuencias:** se eliminan RF-5, RF-6, RF-7, RF-9 y RF-12 en lo relativo a encendido/apagado por software, la ventana de gracia de 1–2 s, el tiempo máximo de encendido y el timeout de apagado. Se enmiendan P3, P4 y P5 de la constitución, la regla de seguridad de `AGENTS.md` y `README.md`. La seguridad depende del operador, de un blanco seguro, de la clase/potencia y de un corte físico accesible. El sistema no puede garantizar que el haz no alcance ojos durante búsqueda, movimiento o fallo.
- **Alternativa descartada:** GPIO o relé con apagado por software, con timeout y apagado por pérdida. Se descarta por el circuito extra; se recomienda sustituir la cinta por un interruptor de retención o una seta.
- **Spec relacionada:** `specs/001-torreta-seguimiento-facial/spec.md`.
