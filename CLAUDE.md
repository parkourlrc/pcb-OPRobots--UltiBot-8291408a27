# UltiBot

Robot de minisumo con 6 sensores SHARP GP2Y0E03 y control FOC de motores
brushless. Arquitectura de doble MCU: STM32F4 para estrategia y sensores,
STM32F1 para control de motores.

## Stack tecnológico
- Microcontrolador principal: STM32F401CCU6 (BlackPill) @ 84 MHz
- Microcontrolador FOC: STM32F103C8T6 (BluePill) @ 72 MHz
- Framework: libopencm3 (bare-metal, sin HAL)
- Entorno: PlatformIO + VS Code
- Lenguaje: C11
- Sensores: 6× SHARP GP2Y0E03 (IR distancia) + 2× QRE1113 (línea)
- Actuadores: 2× BLDC gimbal 12V 80T con SimpleFOCMini

## Estructura del repositorio
- `source_code/STM32F4CCU6/` — firmware placa principal (PlatformIO + libopencm3)
- `source_code/OPRcontrolFOC/` — submódulo: firmware placa FOC (con doc propia)
- `pcb_files/` — diseño electrónico (KiCad 7, gerbers, CubeMX)
- `3d_model/` — piezas mecánicas (SketchUp, STL)
- `images/` — fotos del robot
- `docs/` — documentación técnica (MkDocs Material)

## Convenciones
- Lenguaje: C11 con libopencm3
- Comentarios en español e inglés
- ISRs en `setup.c` (handlers de interrupción junto a la inicialización)
- Variables compartidas ISR↔main loop declaradas `volatile`
- Bucle principal ~10 Hz con `delay(100)` (el código tiene un TODO para mover
  `control_main_loop()` a ISR de 1 ms)
- `printf()` transmite por ambos USART1 (debug) y USART6 (FOC)

## Arquitectura
- **Placa principal (STM32F4)**: sensores ADC+DMA → SysTick 1 kHz →
  filtrado → bucle principal ~10 Hz → comandos de velocidad por USART6
- **Placa FOC (STM32F1)**: recibe comandos por USART3 → PI velocidad →
  conmutación sinusoidal → PWM trifásico
- **Comunicación**: USART 115200 8N1, protocolo ASCII (`Lnn`, `Rnn`, `E`, `D`)
- **Control remoto**: RC5 IR en PB3 (EXTI3)
- **EEPROM**: sector 5 de flash (0x08020000) para códigos RC5

## Documentación
- Usa `/doc-init` para regenerar documentación y `/doc-review` para revisar
  cambios y actualizar known issues
- La documentación sigue el estándar OPRobots (MkDocs Material + estilo ZoroBot3)
- Despliegue en https://docs.oprobots.org/robots/UltiBot
- El submódulo OPRcontrolFOC tiene su propia doc en
  https://docs.oprobots.org/boards/OPRcontrolFOC

## Notas
- El submódulo `OPRcontrolFOC` tiene su propio `CLAUDE.md` y documentación.
  No duplicar issues ni contenido entre ambos.
- `commands.h` se comparte entre ambos proyectos (include path `-I` en
  platformio.ini de STM32F4CCU6).
- La frecuencia real del FOC es 72 MHz aunque `config.h` declare 84 MHz
  (issue documentado en OPRcontrolFOC HW-01).
- Los offsets de encoder (+22, -61) son específicos del montaje en UltiBot.
- La función `_write()` envía a USART1 y USART6 simultáneamente. Si se añade
  debug serial, verificar que no interfiere con los comandos de motor.
