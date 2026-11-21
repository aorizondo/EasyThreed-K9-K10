# EasyThreed K10 con Klipper (hack USB)

La K10 de fábrica **no se puede controlar desde un ordenador**: su USB-C solo alimenta (controlador
USB-PD PW6606) y el firmware de fábrica es cerrado, sin protocolo de G-code por serie. Este hack
da a la placa un enlace de datos con el host (USB nativo del MCU o UART) y le pone **Klipper**.

| Estado | |
|---|---|
| ✅ Probado | Soldadura USB (PA11/PA12 + pull-up 1.5 kΩ), flasheo por microSD, Klipper enumera por USB y responde |
| ⚠️ Pendiente | Mapa de pines completo (pin del **calentador** sin resolver, eje de cada motor sin asignar) → [`printer.cfg`](printer.cfg) es una **plantilla**, todavía no imprime |

> **No hay finales de carrera ni acelerómetro.** El reversing solo encontró **una** entrada de
> final de carrera (`PA3`). El homing de X/Y/Z será manual (`SET_KINEMATIC_POSITION`) hasta
> identificar qué eje usa `PA3`.

## Hardware (resumen)

| Pieza | Dato |
|---|---|
| MCU | **GD32F303CBT6** (LQFP48, 128 KB flash, 48 KB RAM, cristal 8 MHz). Klipper lo trata como STM32F103 |
| Drivers | 4× HR4988 (clon de A4988) soldados: sin UART, sin sensorless |
| Alimentación | USB-C **solo PD** o jack DC; regulador AMS1117 a 3.3 V |
| Bootloader | Propio (no MKS): lee **`p10_printer.bin`** de la microSD, lo graba en `0x08004000` (**16 KiB**) y lo renombra a `.CUR` |
| Mecánica (fábrica) | 100×100×100 mm, pasos/mm X606 Y606 Z600 E1040, PID 22.2/1.08/114 |

Detalle: [docs/02](../docs/02-hardware-k10.md) · reversing: [firmware/k10/reversing.md](../firmware/k10/reversing.md).

## 0. Antes de empezar: billete de vuelta

El firmware de fábrica lo publica palmarci (el fabricante se lo envía por email a quien lo pida en
`info@easythreed.com`). Restaurar = poner ese `p10_printer.bin` en la SD, igual que Klipper.
Hashes y procedimiento: [firmware/k10/stock/README.md](../firmware/k10/stock/README.md). **No lo
redistribuimos** (es propietario).

## 1. Enlace de datos: soldar

Tres vías, todas a **3.3 V** ([docs/04](../docs/04-soldadura-uart-k10.md)):

| Vía | Qué soldar | Resultado |
|---|---|---|
| **A. USB nativo** | `PA12` (D+), `PA11` (D−), GND a un cable USB + **resistencia 1.5 kΩ de D+ a 3.3 V** (el GD32 no tiene pull-up interna; sin ella no enumera). VBUS (rojo) **sin conectar** | `/dev/serial/by-id/usb-Klipper_stm32f103xe_…` |
| B. UART a la Pi | `PA9` (TX) → pin 10, `PA10` (RX) → pin 8, GND → pin 6 | `/dev/ttyAMA0` |
| C. UART + adaptador USB-TTL 3.3 V | `PA9`/`PA10`/GND al adaptador | `/dev/serial/by-id/…` |

Esquema de soldadura (pines del LQFP48, colores, fotos): [diagrams/k10-soldadura-usb.md](../diagrams/k10-soldadura-usb.md).
Soldador de punta fina + flux (paso 0.5 mm), **no** aire caliente.

> ⚠️ **Conflicto posible de la vía A**: en el firmware de fábrica analizado (placa RCT6/LQFP64),
> `PA11`/`PA12` salen como STEP/DIR de un motor. En la placa CBT6 probada el USB funciona, pero
> **comprueba con el multímetro que `PA11`/`PA12` no van a un driver**. Si van, la vía USB deja ese
> motor sin control: usa la **B** o la **C** (USART1, `PA9`/`PA10`).

Con el firmware de fábrica, conectar el USB no hace **nada** (el USB del GD32 lo activa el
software): el primer flasheo siempre va por la SD.

## 2. Firmware: compilar

No hace falta toolchain local: el workflow [`build-k10-klipper.yml`](../../.github/workflows/build-k10-klipper.yml)
compila Klipper en GitHub Actions y deja **`p10_printer.bin`** como artefacto descargable
(pestaña *Actions → Build K10 Klipper firmware → Run workflow*). Config:
[firmware/k10/klipper.config](../firmware/k10/klipper.config).

Equivalente en `make menuconfig` (si compilas tú):

| Opción | Valor |
|---|---|
| Micro-controller | STMicroelectronics STM32 |
| Processor model | STM32F103 |
| **Disable SWD at startup** | **sí** (necesario en los GD32) |
| Bootloader offset | **16KiB bootloader** |
| Clock Reference | 8 MHz crystal |
| Communication interface | **USB (on PA11/PA12)** para la vía A · **Serial (USART1 PA10/PA9)** para B/C |

Tras `make`: `cp out/klipper.bin p10_printer.bin` (este bootloader no cifra).

## 3. Flashear

1. Copia `p10_printer.bin` a la raíz de la microSD de la K10.
2. Enciende. **Éxito = el archivo pasa a `p10_printer.CUR`**. Si sigue `.bin`, fue rechazado.
3. Vía A: `lsusb` muestra `1d50:614e` y aparece `/dev/serial/by-id/usb-Klipper_…`.

Alternativa por SWD (ST-Link + OpenOCD, `SWDIO=PA13`, `SWCLK=PA14`): [docs/05](../docs/05-flasheo-klipper.md).

## 4. Comprobar la comunicación

```bash
~/klippy-env/bin/python ~/klipper/klippy/console.py /dev/serial/by-id/usb-Klipper_stm32f103xe_*
# > get_uptime   (debe responder)
```

## 5. Configurar (pendiente de reversing)

[`printer.cfg`](printer.cfg) tiene lo conocido (termistor `PA2`, final de carrera `PA3`, candidatos
de STEP/DIR) y marca con `P??` lo que falta. Para completarlo, con la placa **sin motores ni
calentador conectados**, sigue [docs/10](../docs/10-reversing-pinout-k10.md): multímetro desde los
pines de cada driver y del MOSFET del calentador (HY1403) hasta el MCU.

## Referencias

- Reversing de la placa por palmarci: <https://palmarci.me/blog/2025-05-02-easythreed-k10-3d-printer-main-board-reversing/index.html>
- GD32F303 en Klipper ("Disable SWD at startup"): configs `printer-creality-*` de Klipper
  (p. ej. `printer-creality-ender3-v2-neo-2022.cfg`, placa 4.2.2 con GD32F303).
