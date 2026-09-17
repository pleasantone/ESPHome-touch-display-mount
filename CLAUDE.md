# CLAUDE.md

ESPHome + LVGL Home Assistant touch panels ("SmartDisplay") plus 3D-printable mounts.

## Layout

- `esphome/home-like/<variant>/home-like.yaml`: the iOS Home-style tile UI. Published variants are `cyd-2432s028`, `cyd-2432s028-9342` (proposed upstream, see below) and `ili9341-external-esp32`; `jc3248w535` and `cyd-2432s028-ili9342` are local and untracked. Assets live in `home-like/fonts/` and `home-like/images/`, but **no config reads them from disk any more** — every one fetches them over HTTP at build time (see *Remote assets*), so no variant folder holds `fonts`/`images` symlinks and each config is a single self-contained file. The folders stay because the URLs point at them in the repo. `TILE_CONFIGURATION.md` documents every `TILE*_` substitution.
- `esphome/buttons/<variant>/buttons.yaml`: the older, simpler button UI.
- `esphome/secrets.yaml.example`: copy it to `esphome/secrets.yaml` (gitignored). Every config reads `!secret` keys from it. It holds four names — `wifi_ssid`, `wifi_password`, `api_encryption_key`, `ap_fallback_password` — matching `/config/esphome/secrets.yaml` in the Home Assistant add-on, which must be kept in sync with it. `ota_password` was dropped on 2026-09-17 once every config moved to OTA encryption. The older `smartdisplay_api_key` / `wifi_ap_password` names were removed on 2026-09-17; every config in the repo was renamed to match, so both panels now share one API key.
- `3d_print/`: STL files and assembly guides. `images/`: README screenshots.
- `CHANGELOG.md`: dated entries (Added / Fixed / Changed) for config changes. Docs-only changes are not listed.

Local, **untracked** device configs:

- `esphome/home-like/cyd-2432s028-ili9342/home-like.yaml`: CYD, USB-C + micro-USB **ILI9342** variant (`smartdisplay`). Landscape 320×240, 6 tiles, stock repo backgrounds. **Self-contained**: it downloads its assets (see *Remote assets*) and needs no `fonts`/`images` symlinks, so the same file builds here and in the Home Assistant add-on.
- `esphome/home-like/jc3248w535/home-like.yaml`: Guition **JC3248W535** 3.5" ESP32-S3 (`smartdisplay-jc3248`). It is portrait with 8 tiles (the published configs have 6), uses 12h time, and adds the `confirm_off` tap action, `TILE*_POWER_TILE` and `TILE*_AVAILABILITY_ENTITY`. Also self-contained for the add-on, but its `ASSET_BASE` points at the **fork** (`pleasantone/…`, branch `local-panels`), because the 320×480 background exists only there.

## Open upstream PRs

All against `akuehlewind/ESPHome-touch-display-mount`, each on its own branch pushed to the `fork` remote (`pleasantone/…`). Opened 2026-09-17; every branch was built in a throwaway worktree off `origin/main` and validated with `esphome config` before pushing.

| PR | branch | what |
|----|--------|------|
| #26 | `ili9342-variant` | new `cyd-2432s028-9342` variant — the upstream `cyd-2432s028` config with ONLY the ILI9342 hardware settings changed. **No personal entities, tiles or colours; keep it that way** — user-specific changes belong in `cyd-2432s028-ili9342/`. Also carries #27/#29/#30's changes so it doesn't land out of step. |
| #27 | `image-platform` | `image: - platform: file` (the bare form goes away in ESPHome 2027.1) |
| #28 | `strapping-pins` | `ignore_strapping_warning` on the CYD's GPIO15/GPIO2 |
| #29 | `remote-assets` | the `REMOTE ASSETS` block + README updates |
| #30 | `ota-encryption` | OTA encryption instead of a password — **needs ESPHome 2026.9+ and a serial flash for existing devices**, so it may well be rejected or deferred |

Each adds a `## 2026-09-17` CHANGELOG section, so those hunks conflict with one another; every PR body says so. **The secret renames were deliberately excluded** — all five PRs still use upstream's `smartdisplay_api_key` and `wifi_ap_password`. If a PR needs reworking, rebuild the branch the same way rather than editing files in the main tree.

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

## ESPHome Device Builder (the Home Assistant add-on)

Every config in the repo is now self-contained (remote assets, no local `fonts`/`images`), and `ota:` everywhere uses a bare `encryption:` rather than a password, so `ota_password` is no longer referenced by anything. The two local configs are the ones actually deployed under the **ESPHome Device Builder** add-on (slug `5c53de3b_esphome`, running 2026.9.0 — ahead of the Homebrew CLI), copied to `/config/esphome/smartdisplay.yaml` and `/config/esphome/smartdisplay-jc3248.yaml` (the add-on's convention is filename = device name). HA is `192.168.88.69`; the SSH add-on takes the user's key on port 22.

- **Remote assets:** the add-on cannot see this repo, so the config downloads everything at build time. ESPHome's `font:` accepts a bare `https://…` URL (it becomes `type: web`) and `image: file:` accepts a URL or `mdi:<icon>`. The `ASSET_BASE` and `MDI_FONT_URL` substitutions hold those URLs. `ASSET_BASE` floats on upstream `main` (backgrounds change rarely and a fix should propagate); `MDI_FONT_URL` is **pinned** to tag `v7.4.47`, because the `MDI_GLYPH_*` values are raw codepoints and an unpinned font bump would silently change which icon a tile shows. The repo's `fonts/materialdesignicons-webfont.ttf` is byte-identical to upstream v7.4.47.
- **Secret names:** the add-on's wizard only ever writes `wifi_ssid` and `wifi_password` into `secrets.yaml`; it puts the API key, OTA password and AP password **inline** in each device YAML. Inlining is not an option here because these files are committed to a public fork, so the credentials stay as `!secret` and the extra names (`api_encryption_key`, `ota_password`, `ap_fallback_password`) live in `/config/esphome/secrets.yaml`. Both panels use `api_encryption_key`, so nothing extra has to be added to the add-on's secrets.
- **OTA:** both panels use `ota: - platform: esphome` with a bare `encryption:` (no password), which inherits each device's api key. See the OTA note below before changing an api key.
- **Version drift:** the add-on runs ahead of the Homebrew CLI, so it warns about things 2026.8 accepts silently. Seen so far: the bare `image:` block is deprecated (removed in 2027.1) in favour of `image: - platform: file`, which 2026.8 also accepts — both local configs now use it. Upstream's configs still use the old form.
- **Rotating a key now costs a serial flash.** Since OTA inherits the api key, changing `api:`'s key changes the OTA key too, and the running firmware rejects a handshake with the new one — no plaintext fallback, because it requires encryption. Flash over USB (CYD `/dev/cu.usbserial-3120`, JC3248 `/dev/cu.usbmodem31101`) or don't change the key. Same reason `password:` → `encryption:` needed one serial install per panel (done 2026-09-17; verified afterwards that encrypted OTA works).
- **Home Assistant kept up by itself** when the CYD's api key changed: no re-auth prompt, `allow_service_calls` preserved. The config entry's `modified_at` matched the moment the YAML landed in `/config/esphome`, so HA seems to take the key from the add-on's copy of the config — inferred from timing, not confirmed in HA's source.

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
  - Touch: `axs15231` on I2C, SDA 4 / SCL 8, interrupt GPIO3, address 0x3B. No calibration needed. GPIO3 is an ESP32-S3 strapping pin (JTAG source select) and ESPHome warns about it; the INT line is wired there on the board and cannot be moved, so the pin carries `ignore_strapping_warning: true`. The panel has always booted reliably with it.
  - Backlight: GPIO1.
  - Board settings: ESP-IDF framework, octal PSRAM, 16MB flash.
  - Software rotation works fine. Occasional "lvgl took a long time" warnings are harmless.
