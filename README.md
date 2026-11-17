# Torreta de Seguimiento Facial con Láser

Proyecto de Electrónica General. Consiste en una torreta de 2 ejes que busca a una persona específica a partir de una foto subida desde el celular, la sigue con una cámara y mantiene un láser siempre encendido sobre su torso cuando queda centrada.

## Hardware disponible
- **Microcontrolador:** ESP32 (ESP-WROOM-32) conectado al PC por USB.
- **Motores:** 2 servomotores SG90 (giro horizontal y vertical) montados en un soporte Pan-Tilt.
- **Alimentación:** Cargador de portátil conectado a un regulador de 5V 3A para dar energía a los motores.
- **Visión y apuntado:** ESP32-CAM montada en la torreta, con video por Wi-Fi local al PC. Módulo láser alimentado por batería con botón de corte mantenido; permanece encendido durante la operación y se corta solo de forma física.

> **Seguridad del láser:** el láser no se controla por software. Permanece encendido mientras la batería esté conectada y el botón presionado; el único corte es físico. Se opera con el operador presente, un blanco seguro y el área de barrido despejada. La cinta que mantiene el botón no es un corte de emergencia fiable: se recomienda sustituirla por un interruptor de retención o una seta.

## Arquitectura del software
1. **Firmware ESP32 (C++):** Recibe ángulos por USB y mueve los 2 motores suavemente. No controla el láser.
2. **Visión Artificial (Python en PC):** Compara la foto de referencia contra el video en vivo, calcula hacia dónde se movió la persona y envía los ángulos al ESP32.
3. **Interfaz Celular (Web local en Python):** Página sencilla accesible desde el navegador del teléfono en la misma red Wi-Fi para tomar o subir la foto de la persona objetivo.
# Proyecto-de-electronica
# Proyecto-de-electronica
