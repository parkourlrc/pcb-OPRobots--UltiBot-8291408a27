# UltiBot

Robot de Minisumo con 6 sensores SHARP GP2Y0E03

<table>
<tr>
<td  colspan="2"> <img src="./images/UltiBot_finish_1.jpg"/> </td>
</tr>
<tr>
<td> <img src="./images/UltiBot_finish_2.jpg" /> </td>
<td> <img src="./images/UltiBot_finish_3.jpg" /> </td>
</tr>
</table>

## ⚙️ Hardware
- STM32F401CCU6 (Principal) y STM32F103C8T6 (FOC)
- 2× SimpleFOCMini
- 2× Motores gimbal 12V 80T
- 2× LiPo 3S ~180 mAh
- 6× Sensores SHARP GP2Y0E03
- 2× Sensores QRE1113
- Ruedas JSUMO JS2622
- Chasis impreso en PLA

![UltiBot Chasis](./images/UltiBot_3d_model.png "UltiBot - Chasis")

## 💻 Software
- Programado con VSCode y PlatformIO
- 3 estrategias de combate: PID pursuit, Keeping Inside, Steps
- 6 aperturas configurables
- Control FOC sinusoidal con lazo PI de velocidad
- Menú de configuración con botón y mando IR RC5
- 4 modos de debug (LEDs, raw ADC, calibrado, posición)

## 📚 Documentación

- [Hardware](docs/01-hardware.md) — MCUs, sensores, motores, PCB, batería, chasis y pinout
- [Arquitectura Software](docs/02-software-architecture.md) — Bucle principal, ISRs, máquina de estados
- [Comunicaciones](docs/03-communications.md) — Protocolo USART entre MCUs y control remoto RC5
- [Sensores](docs/04-sensors.md) — Array de 8 sensores IR, ADC+DMA, filtrado y detección de rival
- [Movimiento](docs/05-movement.md) — Aperturas, estrategias de combate y comandos de motor
- [Control](docs/06-control-system.md) — PD pursuit + PI velocidad + FOC sinusoidal
- [Menú](docs/07-menu-system.md) — Sistema de menú con botones y mando IR
- [Debug](docs/08-debug-system.md) — Modos de depuración, salida serie y visualización LED
- [Almacenamiento](docs/09-storage.md) — EEPROM emulada en flash para códigos RC5
- [Encoders](docs/10-encoders-gyro.md) — Encoders de cuadratura, RPM e indexado Z
- [Batería y LEDs](docs/11-power-leds.md) — Sistema de indicación LED y alimentación
- [Problemas Conocidos](docs/12-known-issues.md) — 12 issues detectados en auditoría (2 críticos, 5 moderados, 5 leves)

Documentación completa en [docs.oprobots.org/robots/UltiBot](https://docs.oprobots.org/robots/UltiBot)

## 🔧 Stack Tecnológico

| Capa | Tecnología |
|------|-----------|
| MCU principal | STM32F401CCU6 (Cortex-M4F, 84 MHz) |
| MCU FOC | STM32F103C8T6 (Cortex-M3, 72 MHz) |
| Framework | libopencm3 |
| Build | PlatformIO |
| Lenguaje | C11 |
| PCB | KiCad 7 |

## KiCAD

### PCB UltiBot v1.0
![UltiBot PCB Front](./images/UltiBot_pcb_front.png "UltiBot - PCB Front")![UltiBot PCB Back](./images/UltiBot_pcb_back.png "UltiBot - PCB Back")

### Librerías

Las librerías necesarias para este proyecto se encuentran en [OPRobots/KiCADLibraries](https://github.com/OPRobots/KiCADLibraries)

La estructura tiene que quedar de la siguiente forma:

```bash
└── kicad_project
    └── custom_libraries
        ├── custom_footprints
        │   ├── *.kicad_mod
        │   └── ...
        └── custom_symbols.kicad_sym
```
