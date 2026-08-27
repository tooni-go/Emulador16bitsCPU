# Documentación: Desarrollo de Snake en RTM32

Este documento sirve como bitácora y referencia técnica sobre las decisiones arquitectónicas tomadas para implementar el juego "Snake" sobre el microprocesador STX4 y el emulador RTM32.

## 1. Descubrimiento de Hardware (I/O)
*Fecha: Agosto 2026*

### El Problema
El manual de la arquitectura detalla que la Entrada/Salida (I/O) de dispositivos se maneja mediante *Memory-Mapped I/O (MMIO)* a partir de la dirección `0xFFFFF000`. Sabemos que escribir en la dirección `-0x100` (`0xFFFFFF00`) emite caracteres por la consola serie. Sin embargo, no hay documentación oficial sobre cómo leer el teclado o los joysticks (Gamepad) ya que el Capítulo 9 (Entrada/Salida) figura como "PRONTO...".

### Pruebas de Hardware
Se diseñó un script exploratorio (`io_test.rtm`) para realizar *polling* sobre las direcciones `0xFFFFFF00` y `0xFFFFFF04`. El objetivo era capturar pulsaciones de teclado enviadas desde una terminal serial bidireccional (`picocom`).

### Resultados y Limitaciones Físicas
Los ensayos arrojaron resultados negativos: el simulador ignoró por completo las pulsaciones de teclado enviadas desde `picocom`. Esto nos llevó a la conclusión de que la versión actual del emulador **carece de implementación para la lectura de datos desde un teclado o consola (RX)**, teniendo únicamente habilitada la transmisión (TX) para imprimir resultados por pantalla.
Debido a esta limitación de hardware, es imposible desarrollar un juego interactivo en tiempo real controlado por el usuario.

---

## 2. Pivote Arquitectónico: Auto-Snake (IA)
Para cumplir con el requerimiento del trabajo práctico, se decidió pivotar la arquitectura del juego hacia una versión **completamente autónoma (Auto-Snake)**. El procesador jugará al Snake por sí mismo utilizando una Inteligencia Artificial rudimentaria programada en lenguaje ensamblador.

### Motor de Renderizado (Consola ANSI)
Dado que la plataforma no posee salida VGA, el renderizado se realiza sobre la terminal serial. Para evitar hacer "scroll" infinito y crear la ilusión de un cuadro de video estático, utilizamos secuencias de escape ANSI enviadas secuencialmente al MMIO `0xFFFFFF00`:
- **Limpiar pantalla:** `\x1b[2J`
- **Mover cursor:** `\x1b[<fila>;<columna>H` (Esto nos permite redibujar sólo la cabeza o la manzana sin tener que limpiar toda la pantalla en cada *frame*, optimizando brutalmente los ciclos de reloj).

### Gestión de Memoria (Estructuras de Datos)
El estado del juego reside en la sección `.data`:
1. **Arrays de Cuerpo (`snake_x`, `snake_y`):** Se reservan 100 palabras de memoria para cada eje.
2. **Buffer Circular:** En lugar de desplazar un arreglo de 100 posiciones cada vez que la serpiente avanza (lo cual tomaría miles de ciclos de reloj inútilmente), utilizamos dos punteros (`head_ptr` y `tail_ptr`). Cuando la serpiente avanza, la nueva cabeza sobreescribe la posición más antigua de la cola, y los punteros avanzan de forma circular.

### Inteligencia Artificial (Toma de Decisiones)
En cada frame, el juego invoca a la subrutina de IA antes de realizar el movimiento.
La IA implementa una heurística del tipo *Greedy Best-First Search* modificada:
1. **Análisis de Ruta:** Calcula el vector de distancia hacia las coordenadas actuales de la manzana (`apple_x`, `apple_y`).
2. **Preferencia Cardinal:** Decide intentar moverse prioritariamente en el eje donde la distancia sea mayor (si está lejos en X, intentará moverse horizontalmente primero).
3. **Evasión de Obstáculos:** Simula hipotéticamente la colisión del movimiento deseado. Si choca contra la pared o contra sí misma, descarta ese movimiento y evalúa el siguiente movimiento más favorable.
4. **Decisión:** Finalmente escribe la nueva dirección en memoria y le cede el control al motor de físicas.

### Generador de Manzanas (PRNG)
Puesto que no existe el concepto de "aleatoriedad" en un procesador sin hardware adicional, hemos implementado un *Linear Congruential Generator* (LCG). La fórmula algorítmica utilizada es la clásica propuesta en el libro *Numerical Recipes*:
`semilla = (semilla * 1664525 + 1013904223) mod 2^32`.
Los bits intermedios de este cálculo se extraen usando desplazamientos lógicos para obtener coordenadas de `apple_x` y `apple_y` pseudo-aleatorias dentro de los límites de la pantalla (e.g., 20x20).
