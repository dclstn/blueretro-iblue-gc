# BlueRetro for the iBlueControlMod GameCube internal board

This is [BlueRetro](https://github.com/darthcloud/BlueRetro) v25.10-beta patched for the iBlueControlMod NGC, an internal GameCube adapter sold on AliExpress by HISPEEDIDO. The board is a clone of LaserBear's BlueRetro GameCube adapter. v25.10-beta is the BlueRetro release that supports the Switch 2 GameCube controller.

Stock BlueRetro builds pair controllers on this board, but the console loses its video. This build keeps the video and pairs the Switch 2 controller.

## Install

1. Download `BlueRetro_iblue_gamecube.bin` from [Releases](../../releases).
2. Turn the console on and flash the file at https://blueretro.io/ota.html. The page accepts it over the stock firmware because the image header doesn't contain `hw2`.
3. Turn the console off and back on.

To go back, flash the stock firmware the same way.

## Findings

**GPIO13 has to stay high.** If it goes low or floats, the board pulls the console's reset line on GPIO14 low. The console keeps running, but there's no picture on HDMI or composite. The stock firmware drives GPIO13 high. Upstream BlueRetro leaves it low, which is why stock builds give a black screen.

I found this by knocking pieces out of the stock firmware one at a time until the picture went, then reading the pin levels back on the port LEDs.

The board gets 12 V from the console's fan header, so it can't switch the console's power. The ESP32 boots with the console, and the picture drops for a moment until GPIO13 is set.

The port shields are grounded, so HW2 wired controller detection can't work. All four ports are given to Bluetooth.

Pins, as the stock firmware sets them:

| Function | LaserBear HW2 | iBlueControlMod |
|---|---|---|
| Reset button input | 0 | 16 |
| Console reset output, open drain | 14 | 14 |
| GPIO13, "power on" | 13 | 13, held high |
| Power off | 16 | 17, unused |
| Power sense | 39 | 39 |
| Port LEDs 1 to 4 | 2, 4, 12, 15 | 15, 4, 2, 12 |
| Status LED | 17 | 22, active low, part of an RGB LED on 18/22/23 |
| Controller data 1 to 4 | 19, 5, 26, 27 | 19, 5, 26, 27 |

## Changes

Everything is behind `CONFIG_BLUERETRO_BOARD_IBLUE_GC`, set in `configs/iblue/gamecube`:

- the pins above, with GPIO13 held high from the start of boot and through deep sleep
- the board is always treated as internal, port sense always reads empty, and a port is only claimed once a controller pairs
- the power on and off commands do nothing

The reset button behaves as in BlueRetro. Tap it to reset the console, or hold it 3 to 6 seconds to start pairing. Holding it while you turn the console on parks BlueRetro until the next power cycle.

## Build

```bash
docker run --rm -v "$PWD:/w" -w /w ghcr.io/darthcloud/idf-blueretro:v5.5.0_2024-12-02 bash -c '
  git config --global --add safe.directory /w && . $IDF_PATH/export.sh &&
  cp configs/iblue/gamecube sdkconfig &&
  echo "$(git describe --tags) iblue gc" | cut -c -31 > version.txt &&
  BR_HW=_iblue BR_SYS=_gamecube idf.py build'
```

The firmware lands in `build/BlueRetro_iblue_gamecube.bin`.

## Credits

BlueRetro is by [darthcloud](https://github.com/darthcloud/BlueRetro) under Apache-2.0. The board was first pulled apart in [discussion #1183](https://github.com/darthcloud/BlueRetro/discussions/1183) by zeus86, VDfreesince1983 and darthcloud.

Tested on one board so far.
