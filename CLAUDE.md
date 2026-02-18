# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

ZMK firmware configuration for **roBa**, a custom split keyboard. The right half is the central side and includes a PMW3610 trackball; the left half has a rotary encoder (EC11).

- **MCU**: Seeeduino XIAO BLE (nRF52840)
- **Firmware framework**: ZMK (v0.3-branch)
- **External driver**: `kumamuk-git/zmk-pmw3610-driver` for the trackball

## Build & Deploy

Firmware is built entirely through **GitHub Actions** — there is no local build. Pushing to `main` (or opening a PR) triggers `.github/workflows/build.yml`, which calls `zmkfirmware/zmk/.github/workflows/build-user-config.yml@main`. The build matrix in `build.yaml` produces three artifacts: `roBa_R` (central/right), `roBa_L` (peripheral/left), and `settings_reset`. The right side uses the `studio-rpc-usb-uart` snippet for ZMK Studio support.

A separate **keymap-drawer** workflow (`.github/workflows/draw.yml`, manual dispatch only) regenerates the SVG in `keymap-drawer/roBa.svg` from the keymap.

## Architecture

### Key Files

| File | Purpose |
|---|---|
| `config/roBa.keymap` | Keymap definition — layers, combos, macros, custom behaviors |
| `config/roBa.json` | Physical layout metadata for ZMK Studio / keymap-drawer |
| `config/west.yml` | West manifest — pins ZMK version and the PMW3610 driver |
| `boards/shields/roBa/roBa.dtsi` | Shared devicetree: matrix transform, physical layout, kscan, encoder, trackball listener |
| `boards/shields/roBa/roBa_R.overlay` | Right half: column GPIOs, SPI/trackball pin config, trackball listener enable |
| `boards/shields/roBa/roBa_L.overlay` | Left half: column GPIOs, encoder enable |
| `boards/shields/roBa/roBa_R.conf` | Right half Kconfig: PMW3610 settings (CPI, scroll, orientation), ZMK Studio, battery, encoder |
| `boards/shields/roBa/roBa_L.conf` | Left half Kconfig: battery proxy, encoder, pointing |
| `build.yaml` | GitHub Actions build matrix |

### Split Architecture

- **Right half (`roBa_R`)** is the central role — it runs the keymap, hosts the trackball (PMW3610 over SPI), and exposes ZMK Studio.
- **Left half (`roBa_L`)** is the peripheral — it has the rotary encoder and forwards key events over BLE to the right half.
- The matrix is 11 columns x 4 rows; right side uses `col-offset = <6>` to map into the shared transform.

### Keymap Layers (8 total)

0. **Default** — QWERTY with home-row-ish mods and combos (Tab, Shift+Tab, Esc, Space, Equal)
1. **FUNCTION** — F-keys, navigation (arrows, Home/End, PgUp/PgDn), volume, window management
2. **NUM** — Numpad on the left, symbols/brackets on the right
3. **ARROW** — Dedicated arrow navigation with tab switching
4. **MOUSE** — Auto-activated by trackball movement (`automouse-layer = <4>`); mouse buttons
5. **SCROLL** — Activated when holding K (`my_tap_hold 5 K`); trackball becomes scroll wheel (`scroll-layers = <5>`)
6. **Layer 6** — Ctrl+letter shortcuts on the left, Bluetooth profile selection + `bootloader` + `BT_CLR` on the right
7. **Layer 7** — Empty / reserved

### Custom Behaviors

- `to_layer_0` macro: sends a keypress then switches to layer 0
- `lt_to_layer_0`: hold-tap that momentarily activates a layer on hold, fires `to_layer_0` on tap
- `my_tap_hold`: balanced hold-tap with `hold-trigger-on-release` for specific key positions (used for scroll layer activation on K)

### Trackball Config (PMW3610)

Configured in `roBa_R.conf`: CPI 400, 180-degree orientation, inverted scroll X, 125 Hz polling, 700ms automouse timeout, smart algorithm enabled. SPI wired on the right half overlay.
