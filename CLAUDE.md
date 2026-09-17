# CLAUDE.md

ESPHome + LVGL Home Assistant touch panels ("SmartDisplay") plus 3D-printable mounts.

## Layout

- `esphome/home-like/<variant>/home-like.yaml`: the iOS Home-style tile UI. Published variants are `cyd-2432s028`, `cyd-2432s028-9342` (proposed upstream, see below) and `ili9341-external-esp32`; `jc3248w535` and `cyd-2432s028-ili9342` are local and untracked. Assets are shared in `home-like/fonts/` and `home-like/images/`. `TILE_CONFIGURATION.md` documents every `TILE*_` substitution.
- `esphome/buttons/<variant>/buttons.yaml`: the older, simpler button UI.
- `esphome/secrets.yaml.example`: copy it to `esphome/secrets.yaml` (gitignored). Every config reads `!secret` keys from it.
- `3d_print/`: STL files and assembly guides. `images/`: README screenshots.
- `CHANGELOG.md`: dated entries (Added / Fixed / Changed) for config changes. Docs-only changes are not listed.

Local, **untracked** device configs:

- `esphome/home-like/cyd-2432s028-ili9342/home-like.yaml`: CYD, USB-C + micro-USB **ILI9342** variant (`smartdisplay`). Landscape 320×240, 6 tiles, stock repo backgrounds, upstream-style `fonts`/`images` symlinks to the shared folders.
- `esphome/home-like/jc3248w535/home-like.yaml`: Guition **JC3248W535** 3.5" ESP32-S3 (`smartdisplay-jc3248`). It is portrait with 8 tiles (the published configs have 6), uses 12h time, and adds the `confirm_off` tap action, `TILE*_POWER_TILE` and `TILE*_AVAILABILITY_ENTITY`.

Tracked and sent upstream (branch `ili9342-variant`, PR akuehlewind/ESPHome-touch-display-mount#26, pushed to the `fork` remote = `pleasantone/…`):

- `esphome/home-like/cyd-2432s028-9342/home-like.yaml`: the upstream `cyd-2432s028` config with ONLY the ILI9342 hardware settings changed — no personal entities, tiles or colours. Keep it that way; user-specific changes belong in `cyd-2432s028-ili9342/`.

## Commands

Run these from `esphome/home-like/`, e.g. `esphome run jc3248w535/home-like.yaml`. Relative asset paths resolve against the YAML file's own directory, not the working directory. Each variant folder holds its own `.esphome/` build cache and a gitignored `secrets.yaml` symlink to `../../secrets.yaml`, since ESPHome only looks for secrets next to the main config. The ESPHome in use is 2026.8.x (Homebrew).

```sh
esphome config <file>.yaml                                     # validate
esphome compile <file>.yaml
esphome run <file>.yaml --device <name>.local --no-logs        # build + OTA
esphome upload <file>.yaml --device /dev/cu.usbmodemXXXX       # first flash over USB
esphome logs <file>.yaml --device <name>.local                 # config dump + runtime logs
```

`esphome logs` never exits on its own. In scripts, run it in the background and kill it after N seconds.

## How the YAML is structured

- **Substitutions:** everything a user edits sits in `substitutions:` at the top: `TILE1..N_*` blocks, the `MDI_GLYPH_*` list and `ORIENTATION` presets. Below the "No need to edit below this line" banner, nothing should be hard-coded per install.
- **Per-tile code is copy-pasted, not generated.** Each tile appears in:
  - its substitutions block
  - `homeassistant` sensor and text_sensor items (brightness, percentage, preset, cover, climate, color modes, state)
  - the `# ---------- TILE n ----------` widget block
  - these lines in the `ui_refresh` lambda: `tN_on`, `set_value_auto`, `sync_open_climate` and `apply_tile`

  To add a tile, clone every tile-6 site and check that `grep -c "tileN\|TILEN\|tN_"` gives the same count for every N.
- **Icon glyphs:** each MDI glyph must appear **exactly once** in `MDI_GLYPH_*`, or the font build fails with a duplicate-glyph error. Look codepoints up from the TTF with `freetype-py`, which is available in ESPHome's venv: `/opt/homebrew/Cellar/esphome/*/libexec/bin/python`.
- **Rotation:** LVGL owns it (`lvgl: rotation: ${LVGL_ROTATION}`). The display runs at its native dimensions with **no** `transform:`; ESPHome 2026.6+ rejects swapped dimensions with "Invalid offsets". The touch `transform:` is a fixed correction for the native axes, and LVGL rotates touch coordinates itself.
- **Home Assistant actions:** direct HA service calls (`DIRECT_ACTIONS: "true"`) also need **"Allow the device to perform Home Assistant actions"** turned on in the HA ESPHome integration options. When it's off, logs show `action ... dropped; client has not subscribed to actions`.

## Hardware notes learned the hard way

- **CYD panel variants:** boards sold as "ESP32-2432S028" differ.
  - Micro-USB only: ILI9341 (240×320)
  - USB-C + micro-USB: ST7789V or **ILI9342, which is natively landscape 320×240**

  Driving an ILI9342 as a 240×320 ILI9341 gives a clean boot and valid logs, but a screen of diagonal stripes. The fix is `model: ILI9342` (or `ESP32-2432S028-9342`) with no `dimensions:` block, plus the rotation/touch values below. Verified on hardware 2026-09-17: picture upright and touch aligned.
  - `LVGL_ROTATION` = `ORIENTATION + 180` (the ILI9341 config uses `ORIENTATION + 90`); its native top edge faces the USB side.
  - Touch: `swap_xy` true everywhere, cal X 340-3860 / Y 280-3860, but the mirrors depend on the preset — landscape (0/180) both true, portrait (90/270) both false. All four verified on hardware; the upstream ILI9341 file's "same for every preset" comment does not hold here.
- **Stale state from unpowered devices:** `TILE*_AVAILABILITY_ENTITY` (both local configs) forces a tile inactive and labels it "Offline" while that entity is not `on`; point it at the tile's own entity to disable the check. The Bambu printers keep reporting their last light state after power loss, so the tiles watch `binary_sensor.<printer>_online` (~50s lag, catches a manual unplug too).
- **CYD ILI9342 contrast:** blacks look grey on this TN panel. Gamma tables and VCOM (`0xC7`) tweaks made no visible difference — do not retry them. What did help, all on the bright end: active tiles `bg_opa: COVER` (was 95%, the wallpaper was showing through white), off-tile value text `0xFFFFFF`, the `sublabel` font at Roboto@700, `label`/`sublabel` sizes 13/12, and a longer auto-dim timeout (the panel was dropping to 40% after 10s).
- **JC3248W535 pins:**
  - Display: `mipi_spi` model `JC3248W535` (AXS15231B) on a quad SPI bus. Clock GPIO47, data 21/48/40/39, CS 45.
  - Touch: `axs15231` on I2C, SDA 4 / SCL 8, interrupt GPIO3, address 0x3B. No calibration needed.
  - Backlight: GPIO1.
  - Board settings: ESP-IDF framework, octal PSRAM, 16MB flash.
  - Software rotation works fine. Occasional "lvgl took a long time" warnings are harmless.
