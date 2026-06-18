# Entity mapping — wired to your real Home Assistant

The ESPHome panel's `substitutions:` (in `common/dashboard.yaml`) are now pointed
at the real entities from your `carmenvetere/mobile` HA config — specifically the
original Lovelace prototype in `dashboards/nspanel/`, which this panel was ported
from. Where your real setup uses a different entity *type* than the original
handoff assumed, the binding/handler/service was updated too (not just the ID),
so the control actually works.

## Map

| Panel feature | Substitution | Real entity |
|---|---|---|
| Outdoor temp | `entity_outdoor_temp` | `sensor.bayberry_tempest_temperature` |
| Weather | `entity_weather` | `weather.bayberry` |
| Scene · Morning | `scene_morning` | `scene.good_morning_2` |
| Scene · Movie | `scene_movie` | `scene.movie` |
| Scene · *Evening* (was "Away") | `scene_away` | `scene.evening` |
| Scene · Goodnight | `scene_goodnight` | `scene.good_night` |
| Scene · Dinner | `scene_dinner` | `scene.entertaining` |
| Scene · *Welcome* (was "Focus") | `scene_focus` | `scene.welcome` |
| All Off | `scene_all_off` | `script.nspanel_all_off` |
| Media player | `entity_media_player` | `media_player.living_room` |
| Shade · *First Floor* | `cover_living` | `cover.first_floor_all` |
| Shade · *Bedroom Side* | `cover_bedroom` | `cover.bedroom_side_shades` |
| Shade · *Bedroom Back* | `cover_kitchen` | `cover.bedroom_back` |
| Alarm | `entity_alarm` | `alarm_control_panel.alarmo` |
| Alert count | `entity_alert_count` | `sensor.notification_alert_counter` |
| Power/outage | `entity_power_outage` | `binary_sensor.bayberry_grid_status` *(inverted)* |
| Pool water temp | `entity_pool_water` | `sensor.omnilogic_pool_watersensor` |
| Pool filter speed (display) | `select_pool_filter` | `number.omnilogic_pool_filter_pump_speed` |
| Pool pump (Off) | `switch_pool_filter_pump` | `switch.omnilogic_pool_filter_pump` |
| Pool filter Low/Med/High | `button_pool_*` | `button.omnilogic_pool_filter_pump_{low,medium,high}_speed` |
| Pool heater | `switch_pool_heater` | `water_heater.omnilogic_pool_heater` |
| Pool setpoint | `number_pool_setpoint` | `input_number.pool_heater_setpoint` |
| Settings · Dinner Party | `ib_dinner` | `input_boolean.dinner_party` |
| Settings · Overnight Guests | `ib_guests` | `input_boolean.guest_mode` |
| Settings · Cleaners | `ib_cleaners` | `input_boolean.cleaners_mode` |
| Settings · Away | `ib_away` | `input_boolean.away_mode` |
| Settings · Auto-Arm Away | `ib_autoarm_away` | `automation.alarm_arm_away_when_everyone_leaves` |
| Settings · Auto-Arm Home | `ib_autoarm_home` | `automation.alarm_arm_home_mode_if_someone_is_home_on_good_night_or_11pm` |
| Settings · Auto-Disarm | `ib_autodisarm` | `automation.alarm_disarm_upon_arriving_home` |
| Settings · Nightly Vacuum | `ib_vacuum` | `automation.12_00am_roborock_s8_vacuum_basement` |
| Settings · Living Room Thermostat | `ib_thermo_living` | `automation.living_room_hvac_schedule_v3` |
| Settings · Bedroom Thermostat | `ib_thermo_bed` | `automation.bedroom_hvac_schedule` |
| Settings · Office Thermostat | `ib_thermo_office` | `automation.office_thermostat_schedule_v2` |
| Settings · *Living Room Adaptive* | `ib_auto_bright` | `automation.living_room_adaptive_shading` |
| Settings · *Dining Room Adaptive* | `ib_screensaver` | `automation.dining_room_adaptive_shading` |
| Settings · *Primary Bedroom Adaptive* | `ib_touch_sounds` | `automation.primary_bedroom_shades_schedule_adaptive` |
| Sprinkler banner / start-stop | `entity_sprinkler_run` / `switch_sprinkler` | `switch.pool_180s` *(representative zone)* |

*Italic* names were relabeled in `lvgl_pages.yaml` so the on-screen text matches
the real entity.

## Type changes that also required handler/service edits

- **Power outage is inverted.** `binary_sensor.bayberry_grid_status` is **ON when
  the grid is healthy**. The `bs_power_outage` handler now shows the banner when
  the sensor is **off**.
- **Pool heater is a `water_heater`, not a switch.** The toggle now calls
  `water_heater.turn_on` / `water_heater.turn_off`, and the on/off state is read
  via a `text_sensor` (`txt_pool_heater`, checked = state ≠ `off`).
- **Pool setpoint is an `input_number`.** The +/- buttons now call
  `input_number.set_value`.
- **Pool filter speed is buttons, not a select.** Off turns the pump switch off;
  Low/Med/High press `button.omnilogic_pool_filter_pump_*_speed`.
- **Settings rows beyond Home Modes are `automation.*`.** `homeassistant.toggle`
  works on automations, so the row taps + state reflection are unchanged.

## Things to review / finish (your call)

1. **Alarm code** — `alarm_user_code` is still the placeholder `"1234"`. Set your
   real arm/disarm code (the panel checks it locally before calling Alarmo).
   *Heads up:* your Lovelace keypad uses `input_text.alarm_code_input` +
   `script.nspanel_keypad_digit`; the ESPHome keypad instead checks the code
   on-device — simpler, but it's a separate mechanism.
2. **Sprinklers** are a 7-zone Rachio system with **no aggregate sensor**, so the
   "watering now" banner + start/stop currently track one representative zone
   (`switch.pool_180s`). For a true "any zone on" banner, add a template binary
   sensor in HA and point `entity_sprinkler_run` at it, e.g.:
   ```yaml
   template:
     - binary_sensor:
         - name: Irrigation Active
           state: >
             {{ expand('switch.front_180s','switch.front_and_side_90s',
                       'switch.pool_180s','switch.pool_90s','switch.pool_shrubs',
                       'switch.firepit_shrubs','switch.retaining_wall_drip')
                | selectattr('state','eq','on') | list | count > 0 }}
   ```
   The ESPHome sprinklers page is single-zone-oriented; wiring all 7 zones as
   individual rows is a UI change, not done here.
3. **Music** targets one fixed player (`media_player.living_room`). The Lovelace
   version follows `input_select.sonos_speaker_select`; replicating that on the
   panel needs extra logic. Change the default to your most-used speaker.
4. **Active scene** has no single entity in HA (Lovelace computes it from each
   scene's `last_changed`), so the scene sub-label isn't meaningful.
   `entity_scene_select` points at a real scene only so it resolves.
5. **Basement thermostat** (`automation.basement_hvac_schedule`) has no row — the
   Climate section only has 3 slots (Living/Bedroom/Office). Add a 4th row if you
   want it on the panel.
