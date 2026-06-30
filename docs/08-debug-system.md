# Sistema de Debug

UltiBot dispone de 4 modos de depuración accesibles desde el menú, más
indicadores LED integrados en el flujo normal de competición.

## Modos de Debug

| Modo | Salida | Frecuencia | Formato |
|------|--------|-----------|--------|
| `DEBUG_SENSOR_LEDS` | LEDs | Continua | LEDs reflejan estado digital de sensores |
| `DEBUG_SENSOR_RAW` | Serial (USART1) | 20 Hz | 8 valores ADC raw (0–4095) |
| `DEBUG_SENSOR_CALIBRATED` | Serial (USART1) | 20 Hz | 8 valores calibrados |
| `DEBUG_SENSOR_POSITION` | Serial (USART1) | 20 Hz | Posición estimada del rival |

## Modo 1: Sensor LEDs

Cada sensor de rival (6) tiene un LED asociado que se enciende cuando el sensor
detecta al rival (estado digital = true). Es el modo por defecto y se activa
automáticamente durante la competición.

| LED | Pin | Sensor |
|-----|-----|--------|
| Frontal derecho | PC15 | SENSOR_FRONT_RIGHT |
| Frontal izquierdo | PB1 | SENSOR_FRONT_LEFT |
| Ángulo izquierdo | PB2 | SENSOR_ANGLE_LEFT |
| Ángulo derecho | PC14 | SENSOR_ANGLE_RIGHT |
| Izquierdo | PB10 | SENSOR_LEFT |
| Derecho | PC13 | SENSOR_RIGHT |

Los sensores de línea no tienen LEDs dedicados en este modo.

## Modo 2: Raw ADC

Imprime los 8 valores raw del ADC (0–4095) a 20 Hz por USART1:

```
1234 -  567 - 2345 -  890 - 3456 - 2100 -  456 - 2789 -
   ↑      ↑      ↑      ↑      ↑      ↑      ↑      ↑
 LÍNEA  IZQ   ÁNG_I  FRT_I  FRT_D  ÁNG_D   DER   LÍNEA
  IZQ                                    .      DER
```

> **⚠️ Advertencia**: La salida de debug se envía también por USART6 a la placa
> FOC. Si los motores están activos, los datos de debug pueden interferir con
> los comandos de motor. Ver [CM-01](12-known-issues.md#cm-01).

## Modo 3: Calibrado

Igual que RAW pero con los valores tras aplicar calibración y umbralización.

## Modo 4: Posición

Imprime solo la posición estimada del rival (centroide ponderado):

```
-125
  42
 200
  -3
```

Rango típico: -250 (rival a la izquierda) a +250 (rival a la derecha).

---
## Sistema de LEDs

### LED de Menú (PA8)

| Estado | Comportamiento |
|--------|---------------|
| Competición no iniciada | Según menú activo |
| Opening | Parpadeo 50 ms |
| Running | Fijo encendido |
| Keeping Inside | Parpadeo 125 ms |

### LEDs de Sensores — Animaciones del Sistema

Además del modo debug, los LEDs de sensores se usan para animaciones de estado:

| Animación | Contexto | Descripción |
|-----------|----------|-------------|
| `set_sensor_led_waiting` | Botón start presionado | Barrido de 6 estados (ida y vuelta parcial) |
| `set_sensor_led_starting` | Cuenta atrás | 5→4→3→2→1 LEDs encendidos |
| `set_sensor_led_menu_opening` | Menú de apertura | Patrón específico por apertura |
| `set_sensor_led_menu_strat` | Menú de estrategia | Animación por estrategia |

---
## Salida Serie

La función `_write()` redirige `printf` a ambos USART simultáneamente:

```c
int _write(int file, char *ptr, int len) {
  while (*ptr && (i < len)) {
    usart_send_blocking(USART6, *ptr);  // ← FOC
    usart_send_blocking(USART1, *ptr);  // ← Consola debug
  }
}
```

Esto permite monitorizar por USB-Serial (USART1) pero tiene el efecto secundario
de enviar todo a la placa FOC (USART6).

---
## Código de Debug en Desarrollo

[`main.c:26-51`](../source_code/STM32F4CCU6/src/main.c#L26-L51) contiene bloques
de debug comentados usados durante el desarrollo:

```c
// debug_sensors_leds();
// debug_sensors_raw();
// debug_sensors_position();
// printf("%d - %d\n", ...);
```

---
*Documento generado el 2026-06-30. Ver también [Menú](07-menu-system.md), [Comunicaciones](03-communications.md), [Batería y LEDs](11-power-leds.md).*
