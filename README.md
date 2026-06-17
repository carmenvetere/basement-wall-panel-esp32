# Basement Wall Panel — ESP32-S3 + ESPHome + LVGL

A 480×480 touch **wall-panel dashboard for Home Assistant**, built with
ESPHome's built-in **LVGL** engine. Charcoal + steel-blue theme, 9 screens:
**Home, Scenes, Music, Shades, Security, Sprinklers, Alerts, Pool, Settings.**

It talks to Home Assistant **natively over the ESPHome API** — no MQTT, no REST.

The whole thing can be **previewed on your computer in a desktop window
(no hardware) and even driven by your real Home Assistant** — so you can
test everything *before* buying a panel.

---

## What's in here

| Path | What it is |
|---|---|
| `nspanel-dashboard.yaml` | **Real-hardware entry point** — ESP32-S3 board, Wi-Fi/OTA, the physical ST7701S display + GT911 touch. Composes the shared core. |
| `simulator/simulator.yaml` | **Desktop preview** — runs the *same* UI on your PC via ESPHome's `host` + `SDL` platforms. Mouse = touch. No ESP32 needed. |
| `common/dashboard.yaml` | **Shared core** — design tokens + Home Assistant entity map, fonts, HA↔widget bindings, scripts. Used by both files above. |
| `lvgl_pages.yaml` | The LVGL UI — styles + all 9 pages. |
| `secrets.yaml.example` | Template for Wi-Fi / API key / OTA password. Copy to `secrets.yaml` (git-ignored). |
| `docs/HANDOFF.md` | The original design handoff notes (theme tokens, icon list, caveats). |
| `.github/workflows/` | CI that runs `esphome config` on both targets every push. |

> **ESPHome bundles its own LVGL (8.x)** via the `lvgl:` component — you do not
> install or pick an LVGL version. Just use a current ESPHome (2024.6+; this
> repo was validated on **2026.5.3**).

Architecture: both the real device and the simulator are thin shells that
`!include` the same `common/dashboard.yaml`, so the UI you test on your desk is
byte-for-byte the UI that runs on the panel — only the board/display/touch layer
differs.

```
  common/dashboard.yaml  (UI + HA logic — shared)
        ▲                       ▲
        │ package               │ package
  nspanel-dashboard.yaml   simulator/simulator.yaml
  (ESP32-S3 + ST7701S)     (your PC + SDL window)
```

---

## Step 0 — Install the tools (once)

ESPHome is a Python package. The simulator additionally needs SDL2.

```bash
# ESPHome (use a venv so it doesn't clutter your system Python)
python3 -m venv ~/.esphome-venv
source ~/.esphome-venv/bin/activate
pip install --upgrade pip wheel setuptools
pip install esphome

# SDL2 — only needed for the desktop simulator
#   macOS:           brew install sdl2
#   Debian/Ubuntu:   sudo apt-get install libsdl2-dev
#   Fedora:          sudo dnf install SDL2-devel
```

Verify: `esphome version` (expect 2026.5.x or newer).

---

## Step 1 — Get the code onto GitHub

This repo is already initialized. To get it and your secrets set up:

```bash
git clone https://github.com/carmenvetere/basement-wall-panel-esp32.git
cd basement-wall-panel-esp32

# Create your secrets file from the template (it is git-ignored)
cp secrets.yaml.example secrets.yaml
```

For the **simulator** you don't even need to fill in `secrets.yaml` — it has no
Wi-Fi/OTA. You'll fill it in later for the real hardware (Step 4).

> CI: every push runs `esphome config` on both targets
> (`.github/workflows/esphome-validate.yml`), so you get a green/red check
> that the YAML is valid before you ever flash anything.

---

## Step 2 — Preview in the simulator (no hardware)

This renders the exact LVGL UI in a window on your computer. The first run
downloads the Manrope + Material Design Icons fonts (needs internet) and
compiles a small native binary.

```bash
source ~/.esphome-venv/bin/activate
esphome run simulator/simulator.yaml
```

A 480×480 window opens with the Home screen. **Click with your mouse** to
navigate pages, work the keypad, drag the shade/volume sliders, etc.

Home-Assistant-bound labels (clock, temperatures, track name…) show their
placeholder defaults until you connect HA in Step 3 — the *layout, theme,
navigation and all touch interactions* are fully live offline.

<details>
<summary>How it works / troubleshooting</summary>

- It builds for `platform: host` and draws with `platform: sdl`; your mouse is
  registered as the touchscreen.
- "Unable to run sdl2-config" → SDL2 isn't installed (see Step 0).
- A blank box (□) where an icon should be → a wrong MDI codepoint; see
  `docs/HANDOFF.md` §Icons and fix the `i_*` substitution in
  `common/dashboard.yaml`.
- To stop: close the window or Ctrl-C the terminal.
</details>

---

## Step 3 — Connect the simulator to Home Assistant (live test, still no hardware)

This is the big one: drive your **real** Home Assistant from the simulator, so
you can confirm your entities and controls work before spending a cent.

1. In `simulator/simulator.yaml` the `api:` block is already enabled.
2. Run the simulator on a machine **on the same network as Home Assistant**
   (`esphome run simulator/simulator.yaml`).
3. In Home Assistant: **Settings → Devices & Services → ESPHome → Add Device**,
   and enter the IP of the computer running the simulator (port `6053`).
   HA connects to it exactly like a physical ESPHome node.
4. Now the bound sensors populate (clock, outdoor temp, media, shade positions,
   alarm state…) and your taps call real HA actions (scenes, covers, alarm,
   toggles). Wire up your entity IDs in Step 4 first so they point at things
   that exist.

> Prefer to keep it offline? Delete the `api:` block from
> `simulator/simulator.yaml` for a pure UI preview.

---

## Step 4 — Point it at your Home Assistant entities & icons

All of these live in **one block** at the top of `common/dashboard.yaml` under
`substitutions:` — a quick find-and-replace.

1. **Entities.** Every `entity_*`, `scene_*`, `cover_*`, `switch_*`, `ib_*`,
   etc. is a placeholder (e.g. `entity_outdoor_temp: "sensor.outdoor_temperature"`).
   Point each at a real entity in your HA. `alarm_user_code` is checked locally
   before the keypad calls `alarm_arm_away` / `alarm_disarm`.
2. **Icons (MDI codepoints).** Each icon is a substitution like
   `i_pool: "\U000F0606"  # mdi:pool`. A wrong codepoint renders as a blank box
   (□). A few are marked `(verify)` — check them at
   <https://pictogrammers.com/library/mdi/> (click an icon → "Codepoint").
3. Re-run the simulator (Step 2/3) to see your changes instantly.

See `docs/HANDOFF.md` for the full design-token table and known caveats
(toggle feedback loops, active-state highlighting, etc.).

---

## Step 5 — Buy the panel & flash it

Recommended board: **ESP32-S3 + 480×480 capacitive panel**, e.g. the popular
**Guition ESP32-4848S040** (ST7701S RGB + GT911), which the hardware config is
pre-filled for.

> The actual *Sonoff NSPanel Pro 86* is an **Android** device and does **not**
> run ESPHome/LVGL. This config targets a generic ESP32-S3 panel used in its
> place. (The older non-Pro *NSPanel* is ESP32 but only 480×320.)

Then:

1. **Fill in `secrets.yaml`** (Wi-Fi, OTA password, and an API key from
   `openssl rand -base64 32`).
2. **Match the hardware block** in `nspanel-dashboard.yaml` to your exact panel:
   the `esp32: board:`, `psram:`, `spi:`, `i2c:`, `display:` (pins +
   `init_sequence`) and `touchscreen:` blocks are filled for the Guition board
   but marked `# TODO`. If you already run ESPHome on this panel, paste its
   known-good `display:`/`touchscreen:`/`spi:`/`i2c:`/`psram:` blocks over these.
3. **Validate, then flash over USB the first time:**
   ```bash
   esphome config  nspanel-dashboard.yaml   # check syntax/substitutions
   esphome run     nspanel-dashboard.yaml   # compile + flash + logs
   ```
   After the first USB flash, subsequent updates go over Wi-Fi (OTA).
4. Add the device in **HA → ESPHome** (it'll be auto-discovered) using the API
   key from your `secrets.yaml`.

---

## Quick command reference

```bash
esphome config  simulator/simulator.yaml    # validate the simulator
esphome run     simulator/simulator.yaml    # preview UI on your computer
esphome config  nspanel-dashboard.yaml      # validate the hardware target
esphome run     nspanel-dashboard.yaml      # compile + flash the panel
```

## Security note

`secrets.yaml` is git-ignored and must never be committed. Only
`secrets.yaml.example` (placeholders) lives in the repo. If you ever commit a
real key by accident, rotate it (`openssl rand -base64 32`) and re-flash.
