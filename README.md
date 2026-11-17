# Torreta de Seguimiento Facial con Láser

Proyecto de Electrónica General. Consiste en una torreta de 2 ejes que busca a una persona específica a partir de una foto subida desde el celular, la sigue con una cámara y le apunta con un láser cuando queda centrada.

## Hardware disponible
- **Microcontrolador:** ESP32 (ESP-WROOM-32) conectado al PC por USB.
- **Motores:** 2 servomotores SG90 (giro horizontal y vertical) montados en un soporte Pan-Tilt.
- **Alimentación:** Cargador de portátil conectado a un regulador de 5V 3A para dar energía a los motores.
- **Visión y apuntado:** Cámara web conectada al PC (mientras llega la externa se usa la del portátil) y un módulo láser de 5V (mientras llega se prueba con el LED de la placa ESP32).

## Arquitectura del software
1. **Firmware ESP32 (C++):** Recibe ángulos por USB y mueve los 2 motores suavemente + prende/apaga el láser.
2. **Visión Artificial (Python en PC):** Compara la foto de referencia contra el video en vivo, calcula hacia dónde se movió la persona y envía los ángulos al ESP32.
3. **Interfaz Celular (Web local en Python):** Página sencilla accesible desde el navegador del teléfono en la misma red Wi-Fi para tomar o subir la foto de la persona objetivo.
# Proyecto-de-electronica
# Proyecto-de-electronica
