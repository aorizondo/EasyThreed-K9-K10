# EasyThreed K9 con Klipper

> ⚠️ **Estado: sin probar en una K9 completa.** El firmware y los pines están verificados en la
> misma placa (ET4000+ con STM32F103RC) montada en otra máquina; la geometría, los pasos y las
> corrientes salen de la configuración de fábrica. Revisa todo lo marcado `VERIFICAR` en
> [`printer.cfg`](printer.cfg) antes de imprimir.

## 0. ¿Mi placa sirve?

La K9 (y la K7/X1/X3, misma familia) lleva la placa **ET4000+**, un clon de la **MKS Robin Lite**.
Hay **dos revisiones**:

| MCU (mira el chip grande) | Klipper | Marlin de este repo |
|---|---|---|
| **STM32F103RCT6** (ST) | **Sí** (esta guía) | Sí (rama `marlin`) |
| **AT32F403ARCT7** (Artery) | **No soportado** | No: solo el Marlin del fabricante ([Marlin #25952](https://github.com/MarlinFirmware/Marlin/issues/25952)) |

Si es la AT32, para aquí: deja el firmware de fábrica.

## 1. Necesitas

- Un ordenador Linux que haga de host (Raspberry Pi o PC) con Klipper + Moonraker + Mainsail o
  Fluidd (lo más fácil: [KIAUH](https://github.com/dw-0/kiauh)).
- La microSD de la impresora y un lector.
- **Copia de la SD original** antes de tocar nada: guarda el `mksLite.CUR` de fábrica (o el
  `mksLite.bin` del Marlin de la rama `marlin`). Es tu billete de vuelta.

## 2. Compilar el firmware

El cargador de arranque (bootloader) de la placa lee `mksLite.bin` de la SD, lo graba en
`0x08005000` (**20 KiB**) y lo renombra a `mksLite.CUR`. El archivo va cifrado con el mismo XOR que
usan las MKS Robin.

> **Dos trampas descubiertas al hacerlo funcionar:**
> 1. La config de MKS para la Robin Lite dice "28KiB bootloader", pero el bootloader real arranca
>    en `0x08005000` → **20 KiB**.
> 2. **Klipper ≥ v0.11 no arranca tal cual**: el bootloader rechaza (no renombra) imágenes cuyos
>    vectores (`ResetHandler`, IRQs) apuntan por encima de `0x08008000`. El parche
>    [`robin-lite-vectors-first.patch`](robin-lite-vectors-first.patch) (4 líneas en el linker)
>    coloca los manejadores justo detrás de la tabla de vectores. Verificado con Klipper v0.13.

En el host, con Klipper instalado en `~/klipper`:

```bash
cd ~/klipper && git worktree add --detach /tmp/kk9 HEAD && cd /tmp/kk9   # no ensucia ~/klipper
git apply /ruta/a/este/repo/klipper/k9/robin-lite-vectors-first.patch
cp /ruta/a/este/repo/klipper/k9/klipper.config .config && make olddefconfig && make
./scripts/update_mks_robin.py out/klipper.bin out/mksLite.bin
arm-none-eabi-nm -n out/klipper.elf | grep -E " (ResetHandler|SysTick_Handler)$"   # ambos < 0x08008000
```

`klipper.config` equivale a este `make menuconfig`:

| Opción | Valor |
|---|---|
| Micro-controller | STMicroelectronics STM32 |
| Processor model | STM32F103 |
| Bootloader offset | **20KiB bootloader** |
| Clock Reference | 8 MHz crystal |
| Communication interface | **Serial (on USART1 PA10/PA9)** — el USB de la placa es un CH340 |
| Baud rate | 250000 |

## 3. Flashear

1. Copia `out/mksLite.bin` a la raíz de la microSD (FAT32).
2. Mete la SD en la impresora y enciéndela. Espera ~30 s.
3. **Éxito = el archivo pasa a llamarse `mksLite.CUR`**. Si sigue como `.bin`, el bootloader lo
   rechazó: revisa el parche y el offset.
4. Borra el worktree: `cd ~/klipper && git worktree remove --force /tmp/kk9`.

Volver a Marlin o a fábrica = mismo proceso con el `mksLite.bin` correspondiente.

## 4. Configurar y primer arranque

1. Copia [`printer.cfg`](printer.cfg) a `~/printer_data/config/`.
2. `ls /dev/serial/by-id/` y pon la ruta en `[mcu] serial:`.
3. Mainsail debe mostrar **Ready**. Con la mano en el interruptor, y sin hacer homing todavía:
   - `QUERY_ENDSTOPS` sin pulsar = `open`, pulsando cada final de carrera = `TRIGGERED`. Si sale
     al revés, quita el `!` del `endstop_pin`.
   - Direcciones: `SET_KINEMATIC_POSITION X=50 Y=50 Z=50`, luego `G91`, `G1 X5 F600`: debe
     alejarse del final de carrera de X. Igual con Y; con `G1 Z5` la cama/boquilla deben
     separarse. Si alguno va al revés, `!` en su `dir_pin`.
4. `G28`, `PID_CALIBRATE HEATER=extruder TARGET=200` + `SAVE_CONFIG`, y calibración del extrusor.

**Corriente de motores**: `M907 X800 Z750 E800` (mA, como en Marlin). X e Y comparten canal.

### K9 sin finales de carrera

Algunas K9 llevan los finales de carrera quitados (homing "a tope", como hacía el Marlin
modificado de este repo). En Klipper eso no es seguro con los A4988 (no detectan el choque). Opciones:

- **Montar los finales de carrera** (recomendado).
- **Cero manual**: lleva el cabezal a mano al origen y usa `SET_KINEMATIC_POSITION X=0 Y=0 Z=0`
  en lugar de `G28` (ya habilitado por `[force_move]`).
- **Sonda Z en `Z_MAX` (`PB9`)**: el Marlin de este repo la usa como sonda; en Klipper sería
  `[probe] pin: ^PB9` + `endstop_pin: probe:z_virtual_endstop` en `[stepper_z]`.

## 5. Referencias

- Config de fábrica de la K9 (`lite_cfg.txt`): [gist de JohnTheCoolingFan](https://gist.github.com/JohnTheCoolingFan/39227f4ddd12a7c614d4e360d0473e6a).
- Klipper en una placa ET4000+ (EasyThreeD X3, CoreXY): [gist de t-nissie](https://gist.github.com/t-nissie/81bc471cd543e92d5cd1c3d36474c8b1).
- Pines de la Robin Lite: `Marlin/src/pins/stm32f1/pins_MKS_ROBIN_LITE.h` (rama `marlin`).
