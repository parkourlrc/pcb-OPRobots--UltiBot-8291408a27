# Problemas Conocidos

> **Fecha de análisis**: 2026-06-30
> **Última actualización**: 2026-06-30
> **Total**: 12 issues — 2 críticos, 5 moderados, 5 leves

---

## Resumen por Prioridades

| Prioridad | Cantidad | Issues |
|-----------|:--------:|--------|
| 🔴 Críticos | 2 | CM-01, SW-01 |
| 🟡 Moderados | 5 | SS-01, SS-02, MV-01, CT-01, EE-01 |
| 🟢 Leves | 5 | SW-02, SW-03, BL-01, SS-03, SW-04 |
| **TOTAL** | **12** | |

---

## Correcciones Recientes ✅

| ID | Fecha | Descripción |
|----|-------|-------------|
| — | — | — |

---

## 🔴 Críticos

<a id="cm-01"></a>
#### CM-01 — printf transmite por USART1 y USART6 simultáneamente

<table>
<tr><td><strong>Archivos</strong></td><td><a href="../source_code/STM32F4CCU6/src/usart.c#L3"><code>usart.c:3-21</code></a></td></tr>
<tr><td><strong>Descripción</strong></td><td>La función <code>_write()</code> redirige todo <code>printf</code> a ambos USART simultáneamente: USART1 (consola debug) y USART6 (placa FOC). Cualquier salida de debug — como la de los modos RAW, CALIBRATED o POSITION — se transmite también a la placa FOC. El parser de comandos de la placa FOC recibe estos bytes e intenta interpretarlos como comandos de motor (<code>Lnn</code>, <code>Rnn</code>, <code>E</code>, <code>D</code>).</td></tr>
<tr><td><strong>Impacto</strong></td><td><strong>Alto</strong> — Si el modo debug está activo durante la competición, la salida serial puede: (a) causar desbordamiento del buffer de 8 bytes en la ISR de USART3 del FOC, (b) ser interpretada como comandos espurios de motor si contienen los caracteres D/E/R/L seguidos de dígitos. Durante la competición normal solo se usa <code>DEBUG_SENSOR_LEDS</code> (sin salida serial), por lo que el riesgo se limita a tener un modo debug serial activo accidentalmente.</td></tr>
<tr><td><strong>Mitigación</strong></td><td>El debug serial solo se activa manualmente desde el menú. Durante la competición normal se ejecuta <code>debug_sensors_leds()</code> que no genera salida serial. El parser del FOC descarta comandos con primer carácter no reconocido (aunque el buffer puede desbordarse).</td></tr>
<tr><td><strong>Solución</strong></td><td>Separar la transmisión: <code>_write()</code> debe enviar solo a USART1. Crear una función separada para enviar comandos a USART6 (p.ej., <code>usart6_send()</code>) o usar <code>usart_send_blocking(USART6, ...)</code> directamente en <code>send_command()</code> sin pasar por <code>printf</code>.</td></tr>
</table>

<a id="sw-01"></a>
#### SW-01 — Bucle principal limitado a ~10 Hz con delay(100) incondicional

<table>
<tr><td><strong>Archivos</strong></td><td><a href="../source_code/STM32F4CCU6/src/main.c#L53"><code>main.c:53</code></a>, <a href="../source_code/STM32F4CCU6/src/control.c#L133"><code>control.c:133</code></a></td></tr>
<tr><td><strong>Descripción</strong></td><td>El bucle principal contiene un <code>delay(100)</code> incondicional en la línea 53. El <code>continue</code> de la línea 55 está comentado, por lo que el flujo siempre pasa por el delay. Esto limita la frecuencia del bucle principal a ~10 Hz. La estrategia PID en <code>control.c:133</code> verifica <code>if (get_clock_ticks() - strat_pid_last_ms > 1)</code>, pero esta condición siempre es cierta porque han pasado ≥100 ms desde la última iteración. El comentario del desarrollador en <code>control.c:267</code> indica la intención de mover <code>control_main_loop()</code> a una ISR de 1 ms.</td></tr>
<tr><td><strong>Impacto</strong></td><td><strong>Alto</strong> — La frecuencia de control efectiva es ~10 Hz en lugar de los ~1 kHz potenciales. Para un robot de sumo esto puede ser aceptable (el rival no se mueve a altísima velocidad), pero limita la capacidad de reacción. La comprobación de 1 ms en <code>strat_pid()</code> es superflua y confusa.</td></tr>
<tr><td><strong>Mitigación</strong></td><td>El control a 10 Hz es funcional para minisumo, donde las dinámicas son relativamente lentas. La detección de línea (en la ISR de SysTick a 1 kHz) no se ve afectada.</td></tr>
<tr><td><strong>Solución</strong></td><td>Opción A: Eliminar el <code>delay(100)</code> y añadir un control de tiempo no bloqueante en el bucle principal para ejecutar la estrategia a la frecuencia deseada. Opción B (preferida según el TODO del desarrollador): Mover <code>control_main_loop()</code> a una ISR de timer independiente a 1 kHz, manteniendo el bucle principal solo para menú y debug. Esto requiere añadir protección de concurrencia para variables compartidas.</td></tr>
</table>

---

## 🟡 Moderados

### Sensores

<a id="ss-01"></a>
#### SS-01 — Posición del rival usa detección binaria en vez de intensidad analógica

<table>
<tr><td><strong>Archivos</strong></td><td><a href="../source_code/STM32F4CCU6/src/sensors.c#L84"><code>sensors.c:84-114</code></a></td></tr>
<tr><td><strong>Descripción</strong></td><td>La función <code>update_sensors_position()</code> usa <code>get_sensor_digital(sensor)</code> (booleano) para decidir si un sensor contribuye al centroide, y pondera por <code>RIVAL_SENSOR_MAX</code> (constante 2600) en lugar de usar <code>get_sensor_calibrated(sensor)</code>. El código original con valores calibrados está comentado (<code>/* get_sensor_calibrated(sensor) */</code>). Esto reduce la estimación de posición a ~6 valores discretos, perdiendo la información de intensidad de la señal.</td></tr>
<tr><td><strong>Impacto</strong></td><td><strong>Medio</strong> — La persecución PID solo puede distinguir ~6 posiciones discretas del rival. Con valores analógicos, el control podría ser más suave y preciso, reaccionando proporcionalmente a la intensidad de la señal.</td></tr>
<tr><td><strong>Mitigación</strong></td><td>El debounce de 75 muestras evita fluctuaciones. Para sumo, 6 posiciones discretas pueden ser suficientes dado el tamaño del dohyo y la velocidad del robot.</td></tr>
<tr><td><strong>Solución</strong></td><td>Descomentar <code>get_sensor_calibrated(sensor)</code> y eliminar <code>RIVAL_SENSOR_MAX</code> en el cálculo de <code>sensor_avg</code> y <code>sensor_sum</code> en las líneas 90-91. Verificar que la posición resultante tiene el rango esperado y ajustar las ganancias PID si es necesario.</td></tr>
</table>

<a id="ss-02"></a>
#### SS-02 — Debounce asimétrico en detección de rival

<table>
<tr><td><strong>Archivos</strong></td><td><a href="../source_code/STM32F4CCU6/src/sensors.c#L158"><code>sensors.c:158-159</code></a></td></tr>
<tr><td><strong>Descripción</strong></td><td><code>is_rival_detected()</code> requiere 150 ms de presencia continua del rival para activarse (<code>rival_detected && get_clock_ticks() - rival_detected_ms > 150</code>), pero la desactivación es inmediata cuando <code>rival_detected</code> se pone a false (línea 112). Esto crea un debounce asimétrico: 150 ms para detectar, 0 ms para perder.</td></tr>
<tr><td><strong>Impacto</strong></td><td><strong>Medio</strong> — Breves pérdidas de señal (p.ej., por ruido o movimiento rápido del rival) causan que el robot "pierda" al rival inmediatamente. Las aperturas que dependen de <code>is_rival_detected()</code> pueden terminar prematuramente. La estrategia PID no usa <code>is_rival_detected()</code> para el seguimiento continuo (usa directamente <code>get_sensors_position()</code>), pero las aperturas y transiciones de estrategia sí dependen de esta función.</td></tr>
<tr><td><strong>Mitigación</strong></td><td>El debounce de 75 muestras en la detección digital de cada sensor reduce el ruido aguas arriba. La estrategia PID no se ve afectada porque usa directamente la posición.</td></tr>
<tr><td><strong>Solución</strong></td><td>Añadir un debounce simétrico en la desactivación: mantener <code>rival_detected = true</code> durante N ms adicionales tras la última detección. Un valor de 50-100 ms evitaría desconexiones espurias sin hacer al robot insensible a la pérdida real del rival.</td></tr>
</table>

### Movimiento

<a id="mv-01"></a>
#### MV-01 — Velocidades de apertura superan el clamp de set_motors_speed

<table>
<tr><td><strong>Archivos</strong></td><td><a href="../source_code/STM32F4CCU6/src/control.c#L63"><code>control.c:63</code></a>, <a href="../source_code/STM32F4CCU6/src/control.c#L91"><code>control.c:91</code></a>, <a href="../source_code/STM32F4CCU6/src/motors.c#L4"><code>motors.c:4-13</code></a></td></tr>
<tr><td><strong>Descripción</strong></td><td><code>opening_right_arc()</code> solicita velocidad 90 (<code>set_motors_speed(90, 40)</code>) y <code>opening_left_arc()</code> solicita velocidad 90 (<code>set_motors_speed(40, 90)</code>). Sin embargo, <code>set_motors_speed()</code> satura a ±80. La velocidad efectiva es 80, no 90, sin ninguna indicación al desarrollador.</td></tr>
<tr><td><strong>Impacto</strong></td><td><strong>Medio</strong> — Las aperturas RIGHT_ARC y LEFT_ARC no alcanzan la velocidad prevista en la fase inicial. La diferencia entre 90 y 80 puede afectar la trayectoria esperada de la apertura.</td></tr>
<tr><td><strong>Mitigación</strong></td><td>Los tiempos de las aperturas se calibraron empíricamente con el clamp ya activo, por lo que el comportamiento observado es el correcto. El problema es de mantenimiento: si alguien aumenta el clamp a 100, estas aperturas se comportarán de forma diferente.</td></tr>
<tr><td><strong>Solución</strong></td><td>Opción A: Cambiar 90 → 80 en <code>control.c</code> para que el código refleje el valor real. Opción B: Aumentar el límite de clamp a 100 y verificar que la placa FOC puede manejar ese rango.</td></tr>
</table>

### Control

<a id="ct-01"></a>
#### CT-01 — Estrategia STEPS usa bucles busy-wait bloqueantes

<table>
<tr><td><strong>Archivos</strong></td><td><a href="../source_code/STM32F4CCU6/src/control.c#L162"><code>control.c:162-168</code></a></td></tr>
<tr><td><strong>Descripción</strong></td><td><code>strat_steps()</code> contiene dos bucles <code>while</code> bloqueantes que detienen todo el sistema: 50 ms de avance (<code>while (get_clock_ticks() - ms < 50)</code>) seguidos de 10 ms de parada (<code>while (get_clock_ticks() - ms < 60)</code>). Durante estos períodos no se ejecuta ninguna otra lógica (detección de línea, LEDs, etc.).</td></tr>
<tr><td><strong>Impacto</strong></td><td><strong>Medio</strong> — Durante los pulsos de movimiento (~10% del tiempo en STEPS), la detección de línea no se procesa en el bucle principal. La ISR del SysTick sigue ejecutándose (lee sensores y actualiza posición), por lo que la información se acumula, pero no se actúa sobre ella hasta que el bucle while termina. Si el robot pisa la línea durante un pulso de avance, no reaccionará hasta que el bloqueo termine (hasta 50 ms después).</td></tr>
<tr><td><strong>Mitigación</strong></td><td>La estrategia STEPS está diseñada para combate conservador. La probabilidad de alcanzar la línea en una ráfaga de 50 ms es baja. Si se detecta rival cercano, la estrategia cambia a PID inmediatamente.</td></tr>
<tr><td><strong>Solución</strong></td><td>Reescribir <code>strat_steps()</code> con una máquina de estados no bloqueante usando <code>strat_steps_last_ms</code> como referencia temporal, similar al patrón usado en las aperturas.</td></tr>
</table>

### Almacenamiento

<a id="ee-01"></a>
#### EE-01 — eeprom_save bloquea el sistema durante ~500 ms

<table>
<tr><td><strong>Archivos</strong></td><td><a href="../source_code/STM32F4CCU6/src/eeprom.c#L6"><code>eeprom.c:6-25</code></a></td></tr>
<tr><td><strong>Descripción</strong></td><td><code>eeprom_save()</code> ejecuta un bucle busy-wait de 500 ms para el parpadeo de confirmación, seguido de borrado y escritura de flash. Durante este tiempo (~500 ms total) el sistema está completamente bloqueado. Las interrupciones no se deshabilitan, pero la ejecución del bucle principal se detiene. La escritura en flash se realiza con interrupciones activas; si la ISR del SysTick (que llama a <code>update_sensors_readings()</code>) o cualquier otra ISR ocurre durante el borrado/escritura, podría leer datos de la flash generando un bus fault o datos corruptos.</td></tr>
<tr><td><strong>Impacto</strong></td><td><strong>Medio</strong> — La operación de guardado solo ocurre durante la configuración del mando RC5 (modo IDLE), no en competición. El riesgo de bus fault por acceso a flash durante escritura es bajo en STM32F4 (el core queda en espera durante las operaciones flash), pero las lecturas de DMA desde el bus AHB podrían verse afectadas.</td></tr>
<tr><td><strong>Mitigación</strong></td><td>La operación solo se ejecuta en IDLE. El tiempo de bloqueo es aceptable para una operación de configuración puntual. El ADC usa DMA para transferencias autónomas, por lo que no debería producirse pérdida de datos.</td></tr>
<tr><td><strong>Solución</strong></td><td>Deshabilitar interrupciones durante la operación flash (<code>__disable_irq()</code> / <code>__enable_irq()</code>). Reemplazar el bucle de parpadeo bloqueante por una máquina de estados que use el contador de ticks para parpadear sin bloquear.</td></tr>
</table>

---

## 🟢 Leves

<a id="sw-02"></a>
#### SW-02 — Buffer estático de 8 bytes en ISR de USART6 sin protección de desbordamiento

<table>
<tr><td><strong>Archivos</strong></td><td><a href="../source_code/STM32F4CCU6/src/setup.c#L130"><code>setup.c:130-148</code></a></td></tr>
<tr><td><strong>Descripción</strong></td><td>La ISR de USART6 usa un buffer estático de 8 bytes (<code>static char command[8]</code>) con un índice <code>i</code> que se incrementa sin verificar límites. Si se reciben más de 7 caracteres sin un <code>\n</code>, se produce desbordamiento del buffer. Este es el mismo patrón documentado en OPRcontrolFOC como SW-01.</td></tr>
<tr><td><strong>Impacto</strong></td><td><strong>Bajo</strong> — La placa FOC no envía respuestas largas por USART. Los comandos del FOC al main son confirmaciones cortas o inexistentes. El riesgo real es bajo en la práctica.</td></tr>
<tr><td><strong>Mitigación</strong></td><td>La placa FOC no genera tráfico significativo hacia la placa principal. El buffer de 8 bytes es suficiente para las respuestas esperadas.</td></tr>
<tr><td><strong>Solución</strong></td><td>Añadir verificación de límites: <code>if (i < sizeof(command) - 1) { command[i++] = data; }</code>.</td></tr>
</table>

<a id="sw-03"></a>
#### SW-03 — Debounce de botones usa delay bloqueante de 50 ms

<table>
<tr><td><strong>Archivos</strong></td><td><a href="../source_code/STM32F4CCU6/src/buttons.c#L9"><code>buttons.c:9-13</code></a>, <a href="../source_code/STM32F4CCU6/src/buttons.c#L21"><code>buttons.c:21-25</code></a></td></tr>
<tr><td><strong>Descripción</strong></td><td><code>get_start_btn()</code> y <code>get_menu_mode_btn()</code> implementan debounce mediante doble lectura separada por <code>delay(50)</code>. Cada llamada bloquea durante 50 ms, y en el caso del botón de menú se llama repetidamente desde bucles while anidados.</td></tr>
<tr><td><strong>Impacto</strong></td><td><strong>Bajo</strong> — Los botones solo se consultan en estado IDLE (competición no iniciada), donde la latencia no es crítica. El retraso de 50 ms no afecta a la experiencia de usuario en la configuración.</td></tr>
<tr><td><strong>Mitigación</strong></td><td>El bucle principal ya tiene un <code>delay(100)</code>, por lo que el delay adicional de 50 ms está dentro de un contexto ya lento.</td></tr>
<tr><td><strong>Solución</strong></td><td>Implementar debounce por timestamp: almacenar el momento del último cambio de estado y solo confirmar el nuevo estado tras N ms de estabilidad, sin bloquear.</td></tr>
</table>

<a id="bl-01"></a>
#### BL-01 — LED de sensor de línea sin apagado en set_sensor_led_debug

<table>
<tr><td><strong>Archivos</strong></td><td><a href="../source_code/STM32F4CCU6/src/leds.c#L177"><code>leds.c:177-185</code></a></td></tr>
<tr><td><strong>Descripción</strong></td><td>Los casos <code>LED_SENSOR_LINE_LEFT</code> y <code>LED_SENSOR_LINE_RIGHT</code> en <code>set_sensor_led_debug()</code> solo manejan <code>state == true</code> (encienden LEDs), pero no tienen cláusula <code>else</code> para apagarlos. Una vez encendidos, no hay código que los apague individualmente. Adicionalmente, estos casos nunca se ejecutan en el flujo actual porque <code>debug_sensors_leds()</code> salta los sensores de línea.</td></tr>
<tr><td><strong>Impacto</strong></td><td><strong>Bajo</strong> — Código muerto en la práctica actual. Si en el futuro se activa el debug de sensores de línea (líneas comentadas 36-41 en debug.c), los LEDs se encenderían pero no se apagarían, quedando permanentemente iluminados.</td></tr>
<tr><td><strong>Mitigación</strong></td><td>Las líneas que activarían el debug de línea están comentadas. La función <code>set_sensor_led(false)</code> apaga todos los LEDs de sensor como grupo.</td></tr>
<tr><td><strong>Solución</strong></td><td>Añadir cláusulas <code>else</code> para apagar los LEDs de línea, o eliminar los casos no utilizados si no se planea activar el debug de línea.</td></tr>
</table>

<a id="ss-03"></a>
#### SS-03 — Include de stdint.h con comillas en lugar de ángulos

<table>
<tr><td><strong>Archivos</strong></td><td><a href="../source_code/STM32F4CCU6/include/sensors.h#L7"><code>sensors.h:7</code></a></td></tr>
<tr><td><strong>Descripción</strong></td><td><code>sensors.h</code> incluye <code>"stdint.h"</code> con comillas en lugar de <code>&lt;stdint.h&gt;</code> con ángulos. Aunque funciona (el compilador busca en los include paths del sistema como fallback), es técnicamente incorrecto y puede causar confusión.</td></tr>
<tr><td><strong>Impacto</strong></td><td><strong>Bajo</strong> — Sin impacto funcional. El compilador resuelve correctamente el archivo.</td></tr>
<tr><td><strong>Mitigación</strong></td><td>No requiere acción inmediata.</td></tr>
<tr><td><strong>Solución</strong></td><td>Cambiar <code>#include "stdint.h"</code> por <code>#include &lt;stdint.h&gt;</code> en <code>sensors.h:7</code>.</td></tr>
</table>

<a id="sw-04"></a>
#### SW-04 — Múltiples TODOs sobre funcionalidad incompleta

<table>
<tr><td><strong>Archivos</strong></td><td><a href="../source_code/STM32F4CCU6/src/control.c#L8"><code>control.c:8</code></a>, <a href="../source_code/STM32F4CCU6/src/control.c#L38"><code>control.c:38</code></a>, <a href="../source_code/STM32F4CCU6/src/control.c#L267"><code>control.c:267</code></a>, <a href="../source_code/STM32F4CCU6/src/control.c#L287"><code>control.c:287-289</code></a>, <a href="../source_code/STM32F4CCU6/src/control.c#L315"><code>control.c:315-316</code></a></td></tr>
<tr><td><strong>Descripción</strong></td><td>El código contiene varios TODOs sin resolver: (1) estrategia por defecto hardcodeada a PID con nota de actualizar, (2) preocupación sobre referencias circulares en <code>set_state(last_state)</code>, (3) migración de <code>control_main_loop()</code> a ISR independiente, (4) estrategia por defecto en caso de error no definida, (5) apertura por defecto en caso de error no definida. Además hay código de ajuste dinámico de <code>sine_step</code> comentado (líneas 143-156).</td></tr>
<tr><td><strong>Impacto</strong></td><td><strong>Bajo</strong> — El comportamiento actual es funcional y seguro (motores se detienen en caso de error). Los TODOs representan deuda técnica y funcionalidad potencial no implementada.</td></tr>
<tr><td><strong>Mitigación</strong></td><td>Los casos <code>default</code> en los switches ponen velocidad 0, lo que es seguro. La estrategia PID por defecto es razonable para minisumo.</td></tr>
<tr><td><strong>Solución</strong></td><td>Priorizar y resolver los TODOs según necesidad. El ajuste dinámico de <code>sine_step</code> podría proporcionar más par a baja velocidad (rival cercano) y más velocidad punta cuando el rival está lejos.</td></tr>
</table>

---

## Issues en OPRcontrolFOC

La placa FOC tiene 13 issues documentados en su propia documentación:
[OPRcontrolFOC — Problemas Conocidos](https://docs.oprobots.org/boards/OPRcontrolFOC/08-known-issues/).

Destacamos los más relevantes para UltiBot:

| ID | Descripción | Impacto en UltiBot |
|----|-------------|-------------------|
| HW-01 | SYSCLK declarado 84 MHz, real 72 MHz | Los cálculos de velocidad pueden estar desfasados |
| CT-01 | Control solo PI (KD=0) | Respuesta de velocidad sin término derivativo |
| EG-01 | Offsets de encoder +22/-61 hardcodeados | Específicos del montaje en UltiBot |
| SW-01 | Buffer USART3 de 8 bytes sin protección | Misma vulnerabilidad que SW-02 en placa principal |

---
*Documento generado el 2026-06-30. Ver también [Arquitectura Software](02-software-architecture.md), [Control](06-control-system.md), [Sensores](04-sensors.md).*
