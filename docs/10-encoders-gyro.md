# Encoders

UltiBot utiliza encoders de cuadratura magnéticos en ambos motores para
realimentación de velocidad y posición. La lectura se realiza en la placa
FOC (STM32F103C8) mediante los timers TIM3 y TIM4 en modo encoder.

> **Nota**: Esta sección describe los encoders desde la perspectiva del robot
> completo. La documentación detallada del firmware FOC está en
> [OPRcontrolFOC — Encoders](https://docs.oprobots.org/boards/OPRcontrolFOC/06-encoders-gyro).

## Hardware

| Característica | Detalle |
|---------------|---------|
| **Modelo** | AS5600 o compatible (encoder magnético) |
| **Tipo** | Cuadratura (A, B) + índice Z |
| **Interfaz** | Timer en modo encoder (x4) |
| **Resolución** | 4096 CPR (1024 pulsos × 4) |

### Conexiones

| Señal | Motor Izquierdo | Motor Derecho |
|-------|:---------------:|:-------------:|
| Fase A | PB4 (TIM3 CH1) | PB6 (TIM4 CH1) |
| Fase B | PB5 (TIM3 CH2) | PB7 (TIM4 CH2) |
| Índice Z | PB3 (EXTI3) | PA15 (EXTI15) |

## Procesamiento en la Placa FOC

### Posición absoluta

El timer en modo encoder acumula ticks en un contador de 16 bits. La posición
absoluta se calcula módulo 586 ticks (valor calibrado para estos motores):

$$posicion\_absoluta = total\_ticks \ \% \ 586$$

El módulo maneja correctamente valores negativos.

### Velocidad (RPM)

En cada ciclo de control (~1 kHz) se calcula la diferencia de ticks y se
convierte a RPM:

$$RPM = \frac{\Delta ticks \cdot f_{control} \cdot 60}{CPR}$$

Con $f_{control} = 10$ kHz y $CPR = 4096$:

$$RPM = \frac{\Delta ticks \cdot 10000 \cdot 60}{4096} \approx \Delta ticks \cdot 146.48$$

### Filtro de velocidad

Se aplica un filtro IIR de punto fijo con factor de suavizado configurable:

```c
#define SPEED_SMOOTHING_FACTOR 4
encoder_speed = ((encoder_speed << SPEED_SMOOTHING_FACTOR) - encoder_speed
    + current_speed) >> SPEED_SMOOTHING_FACTOR;
```

Esto equivale a un filtro paso-bajo con constante de tiempo dependiente de
`SPEED_SMOOTHING_FACTOR`.

### Índice Z (Zero)

El pulso de índice Z (un pulso por revolución) se usa como referencia de cero
absoluto. Al detectarse, la posición se corrige con un offset calibrado:

| Motor | Offset |
|-------|--------|
| Izquierdo | +22 |
| Derecho | -61 |

Estos offsets compensan el desfase mecánico entre el imán del encoder y la
posición de conmutación óptima del motor. Son específicos del montaje en
UltiBot.

## Integración con FOC

La posición angular del encoder determina el ángulo eléctrico para la
conmutación sinusoidal:

$$angulo\_electrico = (posicion\_absoluta \cdot 360° / 585) + offset\_direccion$$

Donde el offset de dirección es:
- Avance motor izquierdo: +270°
- Retroceso motor izquierdo: +90°
- Avance motor derecho: +90°
- Retroceso motor derecho: +270°

---
*Documento generado el 2026-06-30. Ver también [Control](06-control-system.md), [Hardware](01-hardware.md).*
