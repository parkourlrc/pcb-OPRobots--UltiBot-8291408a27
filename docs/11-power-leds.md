# Batería y LEDs

UltiBot utiliza un sistema de indicación visual mediante 7 LEDs para comunicar
el estado del robot. La alimentación proviene de dos baterías LiPo 3S sin
monitorización por firmware.

## Batería

| Característica | Detalle |
|---------------|---------|
| **Tipo** | 2× LiPo 3S |
| **Voltaje nominal** | 11.1V (3 × 3.7V) |
| **Voltaje máximo** | 12.6V (3 × 4.2V) |
| **Capacidad** | ~180 mAh cada una |
| **Regulador** | Mini360 (step-down a 5V/3.3V) |
| **Monitorización** | No implementada |

> **⚠️ Advertencia**: El firmware no incluye lectura del voltaje de batería.
> No hay protección por batería baja ni indicación de nivel de carga.

---
## Sistema de LEDs

### LEDs de Sensores (6 unidades)

Cada sensor de rival tiene un LED indicador asociado:

| LED | Pin | Sensor asociado |
|-----|-----|-----------------|
| Frontal derecho | PC15 | `SENSOR_FRONT_RIGHT` |
| Frontal izquierdo | PB1 | `SENSOR_FRONT_LEFT` |
| Ángulo izquierdo | PB2 | `SENSOR_ANGLE_LEFT` |
| Ángulo derecho | PC14 | `SENSOR_ANGLE_RIGHT` |
| Izquierdo | PB10 | `SENSOR_LEFT` |
| Derecho | PC13 | `SENSOR_RIGHT` |

Los LEDs de sensores tienen múltiples funciones según el contexto:

| Contexto | Función | Frecuencia |
|----------|---------|-----------|
| Debug | Reflejan estado digital de sensores | Continua |
| Esperando inicio | Animación de barrido | 100 ms/estado |
| Cuenta atrás | Encendido progresivo 5→4→3→2→1 | Actualizado cada iteración |
| Menú apertura | Patrón fijo según apertura | Estático |
| Menú estrategia | Animación según estrategia | 50–150 ms |

### LED de Menú (1 unidad)

| Pin | Función |
|-----|---------|
| PA8 | Indicador de menú y estado |

| Contexto | Comportamiento |
|----------|---------------|
| MENU_NONE | Apagado |
| MENU_OPENING | Parpadeo 350 ms |
| MENU_STRAT | Parpadeo 150 ms |
| MENU_DEBUG | Fijo encendido |
| Opening | Parpadeo 50 ms |
| Running | Fijo encendido |
| Keeping Inside | Parpadeo 125 ms |
| Guardando EEPROM | Parpadeo 50 ms (500 ms) |

---
## Animaciones LED

### Barrido de espera (set_sensor_led_waiting)

Mientras se mantiene presionado el botón de inicio, 6 LEDs ejecutan una
animación de ida y vuelta parcial con 6 estados:

```
Estado 0: ■ □ □ □ □ □
Estado 1: ■ ■ ■ □ □ □
Estado 2: ■ ■ ■ ■ ■ ■
Estado 3: □ □ ■ ■ ■ ■
Estado 4: □ □ □ □ ■ ■
Estado 5: □ □ □ □ □ □
```

### Cuenta atrás (set_sensor_led_starting)

Durante los 5 segundos previos al combate:

| Segundos restantes | LEDs encendidos |
|:------------------:|-----------------|
| 5→2 | 6 LEDs |
| 1 | Frontal der. + Frontal izq. (2 LEDs) |
| 0 | Ninguno |

La función usa `ceil()` con aritmética de punto flotante para calcular los
segundos restantes.

---
## LEDs en la Placa FOC

La placa OPRcontrolFOC incluye 2 LEDs adicionales en el STM32F103:

| LED | Pin | Función |
|-----|-----|---------|
| Saturación derecha | PC13 | Motor derecho en saturación PID |
| Saturación izquierda | PC14 | Motor izquierdo en saturación PID |

---
*Documento generado el 2026-06-30. Ver también [Hardware](01-hardware.md), [Debug](08-debug-system.md), [Menú](07-menu-system.md).*
