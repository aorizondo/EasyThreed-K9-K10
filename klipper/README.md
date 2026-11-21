# Klipper para EasyThreed K9 y K10

Klipper divide el trabajo: la placa de la impresora solo mueve motores y lee sensores, y un
ordenador aparte (Raspberry Pi o PC con Linux) hace los cálculos y sirve la interfaz web
(Mainsail o Fluidd). Instalación del host: [KIAUH](https://github.com/dw-0/kiauh).

| | K9 | K10 |
|---|---|---|
| Placa | ET4000+ (clon de MKS Robin Lite) | Propia de EasyThreed |
| MCU | STM32F103RC (⚠️ hay una revisión con **AT32F403**: no soportada) | GD32F303CBT6 (como STM32F103) |
| Conexión con el host | USB de serie (CH340), sin modificar nada | **Hay que soldar** (el USB-C solo alimenta) |
| Archivo en la SD | `mksLite.bin` (cifrado XOR), arranque en 20 KiB | `p10_printer.bin` (sin cifrar), arranque en 16 KiB |
| Finales de carrera | X, Y, Z de serie | Uno solo encontrado (`PA3`) |
| Estado | Firmware probado en la placa; `printer.cfg` sin probar en una K9 completa | Comunicación probada; `printer.cfg` pendiente de pines |
| Guía | [k9/README.md](k9/README.md) | [k10/README.md](k10/README.md) |

Ninguna de las dos tiene acelerómetro: el input shaping solo se puede hacer a ojo
([torre de resonancia](https://www.klipper3d.org/Resonance_Compensation.html)).

## Contenido

```
k9/        printer.cfg, klipper.config (menuconfig), parche del linker, guía
k10/       printer.cfg (plantilla), guía del hack USB
docs/      K10 en detalle: hardware (02), soldadura/UART (04), flasheo (05), reversing (10)
diagrams/  K10: esquema de soldadura USB y conexión UART
firmware/k10/   klipper.config (lo compila el CI), reversing del firmware de fábrica, puntero al OEM
reference/      K10: alimentación USB independiente y fotos de la placa
```

El workflow [`build-k10-klipper.yml`](../.github/workflows/build-k10-klipper.yml) compila el
firmware de la K10 en GitHub Actions.
