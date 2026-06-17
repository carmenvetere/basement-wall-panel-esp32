# ESPHome + LVGL — Wall Panel Dashboard

An ESPHome LVGL port of the 480×480 Home Assistant wall-panel dashboard
(charcoal + steel-blue theme). 9 screens: **Home, Scenes, Music, Shades,
Security, Sprinklers, Alerts, Pool, Settings.**

> The original design is an HTML prototype (see the sibling
> `design_handoff_nspanel_dashboard/` if you have it). This package is the
> **ESPHome implementation** of that design. It talks to Home Assistant
> natively over the ESPHome API — no MQTT, no REST.

## Files
| File | What it is |
|---|---|
| `nspanel-dashboard.yaml` | **Entry point.** Substitutions (design tokens + entity map), board/display/touch, fonts, HA entity bindings, scripts, and `lvgl: !include lvgl_pages.yaml`. |
| `lvgl_pages.yaml` | The full LVGL UI — styles + all 9 pages. Included by the entry file. |
| `secrets.yaml` | Wi-Fi / API / OTA secrets (fill in). |

ESPHome bundles its own LVGL (8.x) via the `lvgl:` component — **you do not
install or pick an LVGL version.** Just use a current ESPHome (2024.6+ has the
mature `lvgl:` component; newer is better).

## Hardware
Designed for an **ESP32-S3 + 480×480 capacitive touch panel**. The display/
touch block in `nspanel-dashboard.yaml` is filled in for the popular
**Guition ESP32-4848S040** (ST7701S RGB + GT911). If your board differs, the
only thing you must swap is that one block (and the PSRAM/board lines).

> Note: the actual *Sonoff NSPanel Pro 86* is an **Android** device and does
> **not** run ESPHome/LVGL. This config is for an ESP32-S3 panel used in its
> place. (The older non-Pro *NSPanel* is ESP32 but only 480×320.)

## Setup — 4 things to edit

### 1. `secrets.yaml`
```yaml
wifi_ssid: "Your SSID"
wifi_password: "..."
api_encryption_key: "<base64 32-byte key>"   # `openssl rand -base64 32`
ota_password: "..."
```

### 2. Display + touch (`nspanel-dashboard.yaml` → `display:` / `touchscreen:`)
Replace pins / `init_sequence` / PSRAM / `board:` with your panel's known-good
values. If you already run ESPHome on this board, paste its existing
`display:`, `touchscreen:`, `i2c:`, `psram:`, `esp32:` blocks over these.

### 3. Entity map (`substitutions:`, top of `nspanel-dashboard.yaml`)
Every `entity_*`, `scene_*`, `cover_*`, `switch_*`, `ib_*`, etc. is a
placeholder — point each at your real Home Assistant entity. They're all in
one block so this is a quick find-and-replace. `alarm_user_code` is the code
checked locally before the keypad calls `alarm_arm_away` / `alarm_disarm`.

### 4. Icons (MDI codepoints)
The UI uses **Material Design Icons** loaded from the MDI web-font. Each icon
is a substitution like `i_pool: "\U000F0606"  # mdi:pool`. A few codepoints
are marked `(verify)` — **a wrong codepoint renders as a blank box (□)**.
Check any that look wrong against <https://pictogrammers.com/library/mdi/>
(click an icon → "Codepoint"). The names in the comments are authoritative;
only the hex may need fixing.

## Design tokens (already wired as substitutions)
| Token | Hex | Use |
|---|---|---|
| `color_bg` | `#202327` | screen background |
| `color_surface` | `#2B2F34` | cards / tiles / keys |
| `color_pill_off` | `#363B41` | settings row OFF |
| `color_accent` | `#7295B2` | steel blue — **active states only** |
| `color_accent_light` | `#A9C4DD` | text on accent tint |
| `color_on_accent` | `#F4F2EC` | text/icon on solid accent |
| `color_alert` | `#C8805F` | rust — critical alerts |
| `color_warning` | `#CBA14E` | amber — warnings |
| `color_text` / `color_text2` | `#E9E5DD` / `#9A988F` | primary / secondary text |
| `color_icon` / `color_dim` | `#B4AFA4` / `#8B8980` | idle icon / tertiary |

Borders are `color_border` (#EAE6DD) at 8% opacity. Accent "tints" are the
accent color at 12–20% `bg_opa` over the dark background — the LVGL equivalent
of the prototype's translucent fills.

## How the dynamic bits work
- **Clock**: `time: homeassistant` → `on_time` updates `lbl_clock` / `lbl_ampm` each second.
- **Sensors** (`sensor:` / `text_sensor:`) push HA state into labels/sliders via `on_value` + `lvgl.*.update`.
- **Banners**: `binary_sensor` for power-outage and sprinkler-running toggle `hidden:` on the Home banners. The Home grid uses `flex_grow: 1`, so it **absorbs the banner height — Home never scrolls.**
- **Controls** call `homeassistant.action` (scenes, covers, media, alarm, select, number, switch). Shared calls are wrapped in `script:` (`ha_scene`, `ha_toggle`) to keep the UI terse.
- **Keypad**: appends to the `code_buffer` global; on the 4th digit the
  `code_append` script compares to `alarm_user_code` and arms/disarms, or
  flashes the dots rust on mismatch. Logic lives in the script (not inline)
  so fast taps can't desync it.

## Known caveats / things to tune
1. **Toggle feedback loop (Settings & Pool heater).** Each row reflects an
   `input_boolean`/`switch` via a `binary_sensor` → `lvgl.widget.update
   state.checked`, and the user tap calls `homeassistant.toggle`. Updating a
   widget's checked state can re-fire `on_value`/`on_click` paths in some
   ESPHome versions. If you see a flip-flop, gate it: add a `globals` "syncing"
   bool set true around the `lvgl.widget.update`, and early-return in the
   `on_click` script while it's true. (Left simple here for readability.)
2. **Active-state highlighting** (which scene / filter speed / arm mode is
   currently active) is driven by HA state in the prototype. Wire the
   `text_sensor` `on_value` handlers (`txt_scene`, `txt_filter`,
   `txt_alarm_state`) to `lvgl.widget.update` the relevant card's
   `bg_color`/`border` — a stub is in place for labels; extend to the cards.
3. **Press feedback**: the prototype scales tiles on press. LVGL's default
   theme darkens pressed buttons; if you want more, add a `pressed:` style
   block per button (e.g. slightly lighter `bg_color`).
4. **`long_mode`/widths**: a few labels use fixed widths for ellipsis on the
   480px canvas. Adjust if your font metrics differ.
5. **Pulsing banner icons** (prototype animates opacity) are static here.
   Add an `interval:` that toggles the icon's `text_opa` if you want the pulse.

## Validate before flashing
```bash
esphome config nspanel-dashboard.yaml     # check substitutions/syntax
esphome compile nspanel-dashboard.yaml    # full build (downloads fonts/MDI)
esphome run nspanel-dashboard.yaml        # flash + logs
```
The first compile downloads Manrope (gfonts) and the MDI web-font — needs
internet. If a glyph is missing, it's almost always a codepoint to fix in
step 4.
