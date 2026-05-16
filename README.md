# Smart AC Switch Controller — Home Assistant Blueprint

A single-automation blueprint that turns a dumb on/off switch into a thermostat-like controller for an AC unit whose heat/cool mode is selected physically on the unit (and mirrored in Home Assistant via an `input_select` helper).

## Features

- **Heat or cool**, selected via an `input_select` helper that you set whenever you flip the AC's physical mode switch.
- **Up to 5 temperature sensors** with different update rates. Stale and unavailable sensors are dropped automatically. The remaining sensors are combined into a single **recency-weighted fused temperature** — the freshest reading dominates, but every healthy sensor contributes.
- **Hysteresis (deadband)** and **minimum on/off times** to protect the compressor from short-cycling.
- **Diagnostic notifications** to your phone:
  - **Not-making-progress** — switch has been on for the trend window but the room is still well past setpoint in the wrong direction. Likely a hardware fault.
  - **Long runtime** — unit has been running continuously past your threshold, never reaching setpoint.
  - **Sensor health** — fewer than 2 sensors are reporting fresh data.
  - **Mode mismatch** — when you flip the mode helper, if the current temperature is far past setpoint *in the wrong direction*, the helper is probably set wrong for the season.
- **Edge-triggered alerts** — each alert fires once when its condition becomes true. No cooldown helpers needed.
- **Optional window/door cutoff** — forces the switch off when a binary sensor is open.

All temperatures are in **°C**.

## Prerequisites — one helper to create first

The blueprint needs exactly one helper. Create it via **Settings → Devices & Services → Helpers**:

| Helper | Type | Purpose | Suggested config |
| --- | --- | --- | --- |
| AC mode | `input_select` | You set this when you change the physical mode of the AC | Options: `Cooling`, `Heating`, `Off` |

The notification target uses the **Home Assistant Companion app** — install it on the phone you want to be notified on and confirm it appears as a device under **Settings → Devices**.

## Install

1. Copy `smart_ac_switch.yaml` into your Home Assistant config at:

   ```
   config/blueprints/automation/switch_sensors_ac/smart_ac_switch.yaml
   ```

2. **Developer Tools → YAML → Reload** (or restart Home Assistant).
3. **Settings → Automations & Scenes → Blueprints** — confirm "Smart AC Switch Controller" appears.
4. Click **Create Automation** from the blueprint and fill in:
   - The AC switch entity
   - The mode `input_select`
   - Summer & winter setpoints, deadband
   - 1–5 temperature sensors
   - The mobile device to notify
5. Save. The first evaluation runs on the next minute boundary.

## How the sensor fusion works

Every minute (or on switch/mode change) the automation:

1. Walks the 1–5 configured sensors.
2. Discards any sensor whose state is `unavailable` / `unknown` / non-numeric, or whose `last_updated` is older than the staleness threshold.
3. For each surviving sensor, computes `weight = 1 / max(age_in_seconds, 1)`.
4. Returns the weighted average. A sensor that updated 5 seconds ago dominates one that updated 5 minutes ago, but the older sensor still pulls the average a bit.
5. If 0 sensors survive, control is suspended and a sensor-health alert fires. If only 1 survives, control continues but the sensor-health alert still fires (so you know you're flying single-engine).

## How alert throttling works (no helpers needed)

Each alert is edge-triggered — it fires once when its condition becomes true and won't fire again until the condition has cleared and re-armed:

- **Runtime alert**: fires when the switch state transitions to "on" and stays on for `max_runtime_minutes`. Resets when the switch turns off.
- **Trend ("no progress") alert**: fires when the switch has been on for `trend_window_minutes` *and* the room is still beyond setpoint by (deadband + trend_delta). Resets when the switch turns off.
- **Mode-mismatch alert**: fires only when you *change* the mode helper. If you set it to "Cooling" while the room is already cold, you get one alert.
- **Sensor-health alert**: fires on the minute tick only when a sensor has just changed state (within the last 90 s) and fewer than 2 fresh sensors remain. Catches the *transition* into a degraded state.

Trade-off: on Home Assistant restart, in-memory edge state clears. If a condition is currently true (e.g. the switch has been on for an hour at the moment of restart), the corresponding alert may fire once on first evaluation after restart. This is generally fine.

## Tuning notes

- **Deadband too small** → switch chatters. **Too large** → room overshoots. `0.5 °C` is a reasonable starting point for residential rooms.
- **Min on/off time** protects the compressor. Most window/portable AC manuals require ~3 minutes between starts; `5 min` is safe.
- **Max runtime** depends on your unit's capacity vs. the load. If your AC normally runs 90 minutes on a hot afternoon, set this to `120` so you only get alerted when something is actually wrong.
- **Trend window** must be long enough that the AC has had a chance to start working. `15 min` is typical; shorter values produce false alarms.
- **Trend tolerance (wrong-direction tolerance)** is added on top of the deadband. With deadband `0.5` and tolerance `0.5`, the trend alert fires when the room is still ≥ 1 °C past setpoint after the full window — i.e. the AC ran the whole window without converging.

## Troubleshooting

- **Switch never turns on:** check that the mode helper's current option contains "Cooling" or "Heating" (case-insensitive), or change the *Cooling/Heating option label* inputs in the blueprint configuration to match your localized labels.
- **Trend alert never fires:** confirm the switch stays continuously on long enough — any flicker resets the `for:` timer. Open the automation trace after a long ON cycle.
- **Notification never arrives:** test the Companion app from **Developer Tools → Actions → `notify.send_message`** with `target: device_id: <your device>`.
- **Duplicate alert right after HA restart:** expected, see "How alert throttling works" above.

## Requirements

- Home Assistant **2024.8.0** or newer (needed for `notify.send_message` with a `device_id` target).
- The Home Assistant Companion mobile app, configured and logged in on the target device.
