# CLAUDE.md

ESPHome + LVGL Home Assistant touch panels ("SmartDisplay") plus 3D-printable mounts.

## Layout

- `esphome/home-like/<variant>/home-like.yaml`: the iOS Home-style tile UI. Published variants are `cyd-2432s028`, `cyd-2432s028-9342` (proposed upstream, see below) and `ili9341-external-esp32`; `jc3248w535` and `cyd-2432s028-ili9342` are personal device configs, committed on the `local-panels` branch of the `fork` remote only — never on `main` and never in an upstream PR. Assets live in `home-like/fonts/` and `home-like/images/`, but **no config reads them from disk any more** — every one fetches them over HTTP at build time (see *Remote assets*), so no variant folder holds `fonts`/`images` symlinks and each config is a single self-contained file. The folders stay because the URLs point at them in the repo. `TILE_CONFIGURATION.md` documents every `TILE*_` substitution.
- `esphome/buttons/<variant>/buttons.yaml`: the older, simpler button UI.
- `esphome/secrets.yaml.example`: copy it to `esphome/secrets.yaml` (gitignored). Every config reads `!secret` keys from it. It holds four names — `wifi_ssid`, `wifi_password`, `api_encryption_key`, `ap_fallback_password` — matching `/config/esphome/secrets.yaml` in the Home Assistant add-on, which must be kept in sync with it. `ota_password` was dropped on 2026-09-17 once every config moved to OTA encryption. The older `smartdisplay_api_key` / `wifi_ap_password` names were removed on 2026-09-17; every config in the repo was renamed to match, so both panels now share one API key.
- `3d_print/`: STL files and assembly guides. `images/`: README screenshots.
- `CHANGELOG.md`: dated entries (Added / Fixed / Changed) for config changes. Docs-only changes are not listed.

Personal device configs, committed on `local-panels` (pushed to `fork`) but **never upstream**:

- `esphome/home-like/cyd-2432s028-ili9342/home-like.yaml`: CYD, USB-C + micro-USB **ILI9342** variant (`smartdisplay`). Runs the **landscape 180° preset** (`LVGL_ROTATION: 0`, USB cord on the left), 320×240 with **8 tiles in a 4×2 grid**, stock repo backgrounds. A **portrait 2×4 preset** (240×320, USB at the bottom) is in the file and verified on hardware — switching is 40 lines in two adjacent substitution blocks and nothing else. Tiles are `H2C Plug · H2C Light · X1C Plug · X1C Light` over `Bento Box · Heater · Exhaust · Riser`; the two plug tiles use `confirm_off`. **Self-contained**: it downloads its assets (see *Remote assets*) and needs no `fonts`/`images` symlinks, so the same file builds here and in the Home Assistant add-on.
- `esphome/home-like/jc3248w535/home-like.yaml`: Guition **JC3248W535** 3.5" ESP32-S3 (`smartdisplay-jc3248`). **Mothballed since 2026-09-19** — the hardware was repurposed to test unrelated code and now advertises as `home35` (192.168.89.135), so nothing runs this config; it is kept because the panel may come back. Edit it if you like, but do not assume it is live, and re-copy it to the add-on before any rebuild. It is portrait with 8 tiles (the published configs have 6), uses 12h time, and adds `TILE*_POWER_TILE` and `TILE*_AVAILABILITY_ENTITY` (its `confirm_off` tap action has since been ported to the CYD config too). Also self-contained for the add-on, but its `ASSET_BASE` points at the **fork** (`pleasantone/…`, branch `local-panels`), because the 320×480 background exists only there.

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

### Ported upstream (2026-09-18/19)

Built 2026-09-18 from `origin/main` in throwaway worktrees and validated with `esphome config` on **both** published home-like variants before pushing to `fork`. Each ports a change developed in `cyd-2432s028-ili9342/`, since that file itself never goes upstream. None has a PR yet.

| branch | what |
|--------|------|
| `confirm-off-action` | `TILE*_TAP_ACTION: confirm_off` — dialog on switch-off only, plus a `TILE_CONFIGURATION.md` row. Deliberately **not** the local implementation: the dialog's OK button re-enters `do_tile_action` as a `toggle` instead of calling `switch.turn_off`, so lights, fans and covers work too, and the card is sized in percentages to fit portrait as well. Worth backporting that shape to the two local configs. |
| `slider-live-colour` | brightness sliders tinted from each light's `rgb_color`, with a `sync_open_rgb` helper keeping them current while an overlay is open |
| `colour-swatch-palette` | **PR #31**, opened 2026-09-19. Rebuilt as two independent commits so the maintainer can take the colour fix without the resizing: swatch colours matched to what `color_name` actually sets, then the 3×2 grid sized at runtime (~47px, was 30px) with the brightness slider moved to the bottom edge |

`ili9342-variant` (PR #26) also gained a third commit adding `color_order: RGB`, pushed as a fast-forward to the open PR, whose description was rewritten to cover it.

`confirm-off-action` and `slider-live-colour` are pushed but **deliberately have no PR**: upstream `main` has not moved since 2026-06-21 and the five PRs from 2026-09-17 have no comments, so the plan is to open at most one at a time and wait for a sign of life.

Each of the three new branches adds its own `## 2026-09-18` CHANGELOG section, so those hunks conflict with one another exactly as #26–#30's do — say so in every PR body.

The colour-order fix also went to **ESPHome itself: [esphome/esphome#19419](https://github.com/esphome/esphome/pull/19419)** (filed 2026-09-19 against `dev`), setting `color_order=MODE_RGB` on the `ESP32-2432S028-9342` board in `models/cyd.py` — on the board, not the `ILI9342` chip, because the M5Stack Core's ILI9342C is usually driven BGR. Verified both on hardware and by patching an installed ESPHome and re-resolving a config (BGR before, RGB after). If it lands, the explicit `color_order: RGB` in our configs becomes a harmless restatement. `scratchpad/esphome-ili9342-color-order-issue.md` was the issue draft it replaced.

Validating an upstream config locally needs two things the repo gitignores: a `secrets.yaml` with the **upstream** names (`smartdisplay_api_key`, `wifi_ap_password`, `ota_password` — not the local ones), and `fonts`/`images` symlinks, because the published configs still read assets from disk until PR #29 lands.

## Commands

Run these from `esphome/home-like/`, e.g. `esphome run jc3248w535/home-like.yaml`. Relative asset paths resolve against the YAML file's own directory, not the working directory. Each variant folder holds its own `.esphome/` build cache and a gitignored `secrets.yaml` symlink to `../../secrets.yaml`, since ESPHome only looks for secrets next to the main config. The ESPHome in use is 2026.9.0 (Homebrew) — as of 2026-09-18 the same version as the add-on, so the drift noted below is gone for now.

```sh
esphome config <file>.yaml                                     # validate
esphome compile <file>.yaml
esphome run <file>.yaml --device <name>.local --no-logs        # build + OTA
esphome upload <file>.yaml --device /dev/cu.usbmodemXXXX       # first flash over USB
esphome logs <file>.yaml --device <name>.local                 # config dump + runtime logs
```

`esphome logs` never exits on its own. In scripts, run it in the background and kill it after N seconds.

## ESPHome Device Builder (the Home Assistant add-on)

Every config in the repo is now self-contained (remote assets, no local `fonts`/`images`), and `ota:` everywhere uses a bare `encryption:` rather than a password, so `ota_password` is no longer referenced by anything. Both local configs live under the **ESPHome Device Builder** add-on (slug `5c53de3b_esphome`, running 2026.9.0), copied to `/config/esphome/smartdisplay.yaml` and `/config/esphome/smartdisplay-jc3248.yaml` (the add-on's convention is filename = device name). **Only `smartdisplay.yaml` is live**; the `smartdisplay-jc3248.yaml` copy is stale and left in place for whenever that panel returns, so re-copy it from the repo before rebuilding it. HA is `192.168.88.69`; the SSH add-on takes the user's key on port 22.

- **Remote assets:** the add-on cannot see this repo, so the config downloads everything at build time. ESPHome's `font:` accepts a bare `https://…` URL (it becomes `type: web`) and `image: file:` accepts a URL or `mdi:<icon>`. The `ASSET_BASE` and `MDI_FONT_URL` substitutions hold those URLs. `ASSET_BASE` floats on upstream `main` (backgrounds change rarely and a fix should propagate); `MDI_FONT_URL` is **pinned** to tag `v7.4.47`, because the `MDI_GLYPH_*` values are raw codepoints and an unpinned font bump would silently change which icon a tile shows. The repo's `fonts/materialdesignicons-webfont.ttf` is byte-identical to upstream v7.4.47.
- **Secret names:** the add-on's wizard only ever writes `wifi_ssid` and `wifi_password` into `secrets.yaml`; it puts the API key, OTA password and AP password **inline** in each device YAML. Inlining is not an option here because these files are committed to a public fork, so the credentials stay as `!secret` and the extra names (`api_encryption_key`, `ota_password`, `ap_fallback_password`) live in `/config/esphome/secrets.yaml`. Both panels use `api_encryption_key`, so nothing extra has to be added to the add-on's secrets.
- **OTA:** both configs use `ota: - platform: esphome` with a bare `encryption:` (no password), which inherits each device's api key. See the OTA note below before changing an api key.
- **Version drift:** the add-on has run ahead of the Homebrew CLI in the past (both are 2026.9.0 as of 2026-09-18), and then warns about things the older CLI accepts silently. Seen so far: the bare `image:` block is deprecated (removed in 2027.1) in favour of `image: - platform: file`, which 2026.8 also accepts — both local configs now use it. Upstream's configs still use the old form.
- **Rotating a key now costs a serial flash.** Since OTA inherits the api key, changing `api:`'s key changes the OTA key too, and the running firmware rejects a handshake with the new one — no plaintext fallback, because it requires encryption. Flash over USB (CYD `/dev/cu.usbserial-3120`, JC3248 `/dev/cu.usbmodem31101`) or don't change the key. Same reason `password:` → `encryption:` needed one serial install per panel (done 2026-09-17; verified afterwards that encrypted OTA works).
- **Home Assistant kept up by itself** when the CYD's api key changed: no re-auth prompt, `allow_service_calls` preserved. The config entry's `modified_at` matched the moment the YAML landed in `/config/esphome`, so HA seems to take the key from the add-on's copy of the config — inferred from timing, not confirmed in HA's source.

## How the YAML is structured

- **Substitutions:** everything a user edits sits in `substitutions:` at the top: the board block, `TILE1..N_*` blocks, the `MDI_GLYPH_*` list and the orientation presets. The **board block** (`DISPLAY_MODEL`, `DISPLAY_COLOR_ORDER`) is what the display section reads — `model: ${DISPLAY_MODEL}` and `color_order: ${DISPLAY_COLOR_ORDER}` — because substitutions expand before validation, so moving this config to another ESP32-2432S028 revision never means editing the driver block. Rotation and touch differ per revision too, so change those together (table below). Below the "No need to edit below this line" banner, nothing should be hard-coded per install.
- **Per-tile code is copy-pasted, not generated.** Each tile appears in:
  - its substitutions block
  - `homeassistant` sensor and text_sensor items (brightness, percentage, preset, cover, climate, color modes, state)
  - the `# ---------- TILE n ----------` widget block, inside the `tile_grid` container
  - these lines in the `ui_refresh` lambda: `tN_on`, `set_value_auto`, `sync_open_climate` and `apply_tile`

  To add a tile, clone every tile-6 site. `grep -c "tileN\|TILEN\|tN_"` is a rough check, not an equality test: tile 1 and the highest tile also appear in the header and `publish_action` comments, so their counts run 1–2 high. Comparing the *set* of sites is the real check — normalise each matching line by replacing the tile number and diff the sets. Watch the word boundary if you script it: `\btile6` does **not** match inside `ha_state_tile6` or `ha_avail_tile6`.
- **Tile placement is LVGL flex, not coordinates.** The tiles are children of a `tile_grid` container with `layout: {type: flex, flex_flow: row_wrap}`; they flow in creation order (= reading order) and wrap when the next one no longer fits, so **no tile has an `x`/`y`**. The grid shape falls out of the container width against the tile width: 301px fits exactly four 70px tiles (landscape 4×2), 228px fits exactly two 110px tiles (portrait 2×4). An orientation block therefore sets six values — `GRID_X/Y/W/H`, `GRID_GAP_X/Y` — plus the tile size, where it used to restate sixteen coordinates. The container is `clickable: false` so taps still reach the tiles.
- **Icon glyphs:** each MDI glyph must appear **exactly once** in `MDI_GLYPH_*`, or the font build fails with a duplicate-glyph error. Several tiles may share one glyph — the list is the set of distinct glyphs to build, not one entry per tile (both plug tiles point at the same `\U000F06A5`). Look codepoints up from the TTF with `freetype-py`, which is available in ESPHome's venv: `/opt/homebrew/Cellar/esphome/*/libexec/bin/python`.
- **Rotation:** LVGL owns it (`lvgl: rotation: ${LVGL_ROTATION}`). The CYD config ships the two presets that were wanted — landscape 180° and portrait 90° — each with its touch mirrors; the flips (0° and 270°) are one-line notes beside them, and the full four-preset table is below. The display runs at its native dimensions with **no** `transform:`; ESPHome 2026.6+ rejects swapped dimensions with "Invalid offsets". The touch `transform:` is a fixed correction for the native axes, and LVGL rotates touch coordinates itself.
- **Home Assistant actions:** direct HA service calls (`DIRECT_ACTIONS: "true"`) also need **"Allow the device to perform Home Assistant actions"** turned on in the HA ESPHome integration options. When it's off, logs show `action ... dropped; client has not subscribed to actions`.

## Hardware notes learned the hard way

- **CYD panel variants:** boards sold as "ESP32-2432S028" differ.
  - Micro-USB only: ILI9341 (240×320)
  - USB-C + micro-USB: ST7789V or **ILI9342, which is natively landscape 320×240**

  Driving an ILI9342 as a 240×320 ILI9341 gives a clean boot and valid logs, but a screen of diagonal stripes. The fix is `model: ESP32-2432S028-9342` with no `dimensions:` block, plus the rotation/touch values below. Prefer that board model over the generic `ILI9342` chip: it carries `data_rate`, `cs_pin` and `dc_pin` **including `ignore_strapping_warning` on GPIO15/GPIO2**, and if ESPHome ever fixes the colour-order default (below) it will land on the board model, not the chip. Verified on hardware 2026-09-17: picture upright and touch aligned.
  - `LVGL_ROTATION` = `ORIENTATION + 180` (the ILI9341 config uses `ORIENTATION + 90`); its native top edge faces the USB side.
  - Touch: `swap_xy` true everywhere, cal X 340-3860 / Y 280-3860, but the mirrors depend on the preset — landscape (0/180) both true, portrait (90/270) both false. All four verified on hardware; the upstream ILI9341 file's "same for every preset" comment does not hold here.
- **Stale state from unpowered devices:** `TILE*_AVAILABILITY_ENTITY` (both local configs) forces a tile inactive and labels it "Offline" while that entity is not `on`; point it at the tile's own entity to disable the check. The Bambu printers keep reporting their last light state after power loss, so the tiles watch `binary_sensor.<printer>_online` (~50s lag, catches a manual unplug too).
- **CYD ILI9342 colour order:** ESPHome's `ILI9342` model declares no `color_order`, so it inherits `mipi_spi`'s **BGR** default and the panel draws **red as blue**. `color_order: RGB` fixes it; verified 2026-09-18 with a pure-primary test pattern (red/green/blue all correct, white and grey neutral, no viewing-angle shift). It hides well, because BGR only swaps red and blue: greens stay green and most of this UI is white, grey and black. The background image is swapped too. Upstream PR #26 still carries the bug.
- **CYD ILI9342 contrast:** blacks look grey on this TN panel. Hand-written VCOM (`0xC7`) tweaks made no visible difference — do not retry those. Gamma **does** matter, contrary to what this note said before 2026-09-19: the `ESP32-2432S028-9342` board model and the generic `ILI9342` chip model ship different `0xE0`/`0xE1` tables, and the board model's blacks look better on this panel. The earlier comparison had been made while red and blue were swapped. What did help, all on the bright end: active tiles `bg_opa: COVER` (was 95%, the wallpaper was showing through white), off-tile value text `0xFFFFFF`, the `sublabel` font at Roboto@700, `label`/`sublabel` sizes 13/12, and a longer auto-dim timeout (the panel was dropping to 40% after 10s). This tuning predates the colour-order fix above, but it was all greys and whites, which BGR leaves alone, so the conclusions still hold.
- **JC3248W535 pins:**
  - Display: `mipi_spi` model `JC3248W535` (AXS15231B) on a quad SPI bus. Clock GPIO47, data 21/48/40/39, CS 45.
  - Touch: `axs15231` on I2C, SDA 4 / SCL 8, interrupt GPIO3, address 0x3B. No calibration needed. GPIO3 is an ESP32-S3 strapping pin (JTAG source select) and ESPHome warns about it; the INT line is wired there on the board and cannot be moved, so the pin carries `ignore_strapping_warning: true`. The panel has always booted reliably with it.
  - Backlight: GPIO1.
  - Board settings: ESP-IDF framework, octal PSRAM, 16MB flash.
  - Software rotation works fine. Occasional "lvgl took a long time" warnings are harmless.
