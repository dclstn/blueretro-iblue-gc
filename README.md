# BlueRetro for the iBlueControlMod GameCube internal board

[BlueRetro](https://github.com/darthcloud/BlueRetro) v25.10-beta, which supports the Switch 2 (NSO) GameCube controller, patched for the AliExpress **iBlueControlMod NGC** internal adapter (HISPEEDIDO, a clone of LaserBear's BlueRetro GameCube board).

Stock BlueRetro builds pair controllers on this board but leave the console with no video. This build fixes that.

## Install

1. Download `BlueRetro_iblue_gamecube.bin` from [Releases](../../releases).
2. With the console on, flash it at https://blueretro.io/ota.html. The image header has no `hw2`, so the page accepts it over the stock firmware.
3. Switch the console off and on.

To go back, flash the stock firmware the same way.

## Findings

- **GPIO13 must be held high.** When it's low or floating, the board pulls the console's reset line (GPIO14) low. The console keeps running but outputs no video, on HDMI or composite. The stock firmware drives GPIO13 high; upstream BlueRetro idles it low, hence the black screen.
- The board takes 12 V from the fan header, so it can't switch console power. The ESP32 restarts with the console and the video blanks for a moment until GPIO13 is set.
- The port shields are grounded, so HW2 wired-controller detection can't work. All four ports go to Bluetooth.

Pins, read from the stock firmware:

| Function | LaserBear HW2 | iBlueControlMod |
|---|---|---|
| Reset button in | 0 | 16 |
| Console reset out (open drain) | 14 | 14 |
| GPIO13 ("power on") | 13 | 13, held high |
| Power off | 16 | 17 (unused) |
| Power sense | 39 | 39 |
| Port LEDs 1–4 | 2, 4, 12, 15 | 15, 4, 2, 12 |
| Status LED | 17 | 22, active low (RGB on 18/22/23) |
| Controller data 1–4 | 19, 5, 26, 27 | same |

## Changes

All of these sit behind `CONFIG_BLUERETRO_BOARD_IBLUE_GC` (`configs/iblue/gamecube`):

- the pins above, with GPIO13 held high from the start of boot and through deep sleep
- always an internal adapter, port sense stubbed out, and each port claimed only once a controller pairs
- the power on/off commands do nothing

The reset button works as in BlueRetro (tap to reset, hold 3–6 s to pair). Hold it while switching the console on to park BlueRetro for that session.

## Build

```bash
docker run --rm -v "$PWD:/w" -w /w ghcr.io/darthcloud/idf-blueretro:v5.5.0_2024-12-02 bash -c '
  git config --global --add safe.directory /w && . $IDF_PATH/export.sh &&
  cp configs/iblue/gamecube sdkconfig &&
  echo "$(git describe --tags) iblue gc" | cut -c -31 > version.txt &&
  BR_HW=_iblue BR_SYS=_gamecube idf.py build'
```

The output is `build/BlueRetro_iblue_gamecube.bin`.

## Credits

[darthcloud/BlueRetro](https://github.com/darthcloud/BlueRetro) (Apache-2.0), and [discussion #1183](https://github.com/darthcloud/BlueRetro/discussions/1183) (zeus86, VDfreesince1983, darthcloud). Tested on one board.
