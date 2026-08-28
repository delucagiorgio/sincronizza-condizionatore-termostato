# Sync an air conditioner with a room thermostat

A Home Assistant blueprint that makes a split air conditioner follow the room
thermostat, so the thermostat stays the single place you touch.

Useful when a room has both: a wall thermostat you actually like using, and an
AC unit whose app or remote you don't. Set the thermostat, the AC follows.

---

## What it does

Four rules, one automation per room.

| When | What happens |
|---|---|
| Thermostat set to `off` / `heat` / `cool` | The AC mirrors the mode |
| Thermostat target temperature changes | The AC target temperature follows |
| Window opens | The AC is switched off |
| Window closes | The AC is switched back on — unless the thermostat was turned off meanwhile |
| House is empty and the thermostat is switched on | The thermostat is moved to its `away` preset |
| House is occupied and the thermostat is switched on while still in `away` | The thermostat is moved back to your normal preset |

Two details that are easy to get wrong and are handled here:

- **The target temperature is written only when it actually differs** from the
  value already set on the AC. Without that guard, a thermostat that reports
  small fluctuations produces a stream of redundant commands to the AC.
- **Everything is referenced by `entity_id`, never `device_id`.** Device IDs
  change when you remove and re-pair a device, and every automation that used
  them breaks silently. Entity IDs survive, and Home Assistant tells you when
  one changes.

---

## Requirements

You need four things. The blueprint's selectors will only offer you valid
candidates, but it's worth checking before you start.

**1. A thermostat exposed as a `climate` entity** that supports:
- the `off`, `heat` and `cool` HVAC modes
- `preset_mode`, including an `away` preset and a normal one

**2. An air conditioner exposed as a `climate` entity** that supports
`climate.set_hvac_mode` and `climate.set_temperature`.

**3. A window sensor** — a `binary_sensor` with `device_class: window`. Many
smart thermostats expose one themselves (an open-window detection based on a
temperature drop); a separate contact sensor works just as well. If your sensor
shows up without the right device class, set it in the entity settings under
*Show as*, otherwise the blueprint's picker won't offer it.

**4. An `input_select` helper for presence**, with at least two options — one
of which means "nobody home". Whatever drives that helper (person entities,
device trackers, a manual switch) is outside the scope of this blueprint.

---

## Installation

**Settings → Automations & scenes → Blueprints → Import blueprint**, then paste:

```
https://github.com/delucagiorgio/sincronizza-condizionatore-termostato/blob/main/thermostat_ac_sync_blueprint/sincronizza_condizionatore_con_termostato.yaml
```

Requires Home Assistant **2024.10.0** or newer — the blueprint uses the plural
`triggers:` / `actions:` keys and the `action:` key inside steps.

---

## Inputs

### Room entities

| Input | Required | Notes |
|---|---|---|
| **Thermostat** | yes | The source of truth |
| **Air conditioner** | yes | The follower |
| **Window sensor** | yes | `binary_sensor` with `device_class: window` |

### Presence *(collapsed by default)*

| Input | Required | Default | Notes |
|---|---|---|---|
| **Presence helper** | yes | — | Your `input_select` |
| **Value that means "nobody home"** | no | `Away` | Must match one of the helper's options exactly, case included |
| **Preset to restore when the house is occupied** | no | `comfort` | Whatever your thermostat calls its normal preset |

That last one varies a lot by brand — `comfort`, `home`, `custom`, `manual`.
Open your thermostat entity, look at its `preset_modes` attribute, and use the
value you see there. Getting it wrong doesn't break anything: the service call
simply fails and is logged.

---

## One automation per room

Create one instance per room. The presence inputs are the same in all of them;
only the three room entities change.

Nothing stops you from pointing two instances at the same thermostat, but
there's no reason to — one instance already handles every rule for that room.

---

## Tested with

This is the setup the blueprint was written against and has been running on.
It is not a compatibility list: anything that satisfies the requirements above
should work, and anything that doesn't is worth reporting.

| | |
|---|---|
| **Home Assistant** | Core 2026.8.3, Home Assistant OS 18.2 (Python 3.14.6) |
| **Thermostat** | Meross MTS200B — hardware 7.0.0, firmware 7.6.14 |
| **Thermostat integration** | [Meross LAN](https://github.com/krahabb/meross_lan) v5.8.0 (HACS, local polling) |
| **Air conditioner** | Gree split units, Gree LAN protocol |
| **AC integration** | [Gree A/C](https://github.com/RobHofmann/HomeAssistant-GreeClimateComponent) v4.0.1 (HACS) |
| **Window sensor** | `binary_sensor.*_windowopened`, exposed by the MTS200B itself |
| **Presence** | A two-option `input_select` |
| **Scale** | 4 rooms, 4 instances, in production since 2026-08-28 |

Relevant capabilities of that pair, because they explain the caveats below:

| | Thermostat (MTS200B) | AC (Gree) |
|---|---|---|
| HVAC modes | `off`, `heat`, `cool` | `auto`, `cool`, `dry`, `fan_only`, `heat`, `off` |
| Target range | 5–35 °C | 16–30 °C |
| Target step | **0.5 °C** | **1 °C** |
| Presets | `custom`, `comfort`, `sleep`, `away`, `auto` | none |
| State updates | immediate | **polled, ~60 s** |

---

## Things to know before you rely on it

**Synchronisation is one-way.** The thermostat drives the AC, never the
reverse. Change the AC from its remote or from a dashboard card and the
thermostat won't notice. Making it bidirectional is not a matter of adding
triggers — see the section below.

**Mismatched temperature steps get rounded.** If your thermostat steps in
0.5 °C and your AC in 1 °C, a target of 21.5 °C reaches the AC as 21 or 22.
The two then disagree, harmlessly but visibly. Same for range: a thermostat
set to 5 °C for frost protection will clamp to the AC's minimum.

**A polled AC reacts with a delay.** With an integration that polls rather than
pushes, the AC's state in Home Assistant can lag its real state by up to the
poll interval. The blueprint's guards read that state, so a command issued
while the cached value is stale may be skipped or repeated once.

**Some thermostats echo a stale state for a moment.** Right after a mode
command, the tested unit reported its *previous* mode once before settling on
the new one. Because the blueprint triggers on mode changes, that echo starts
an extra run: switching the thermostat on produced a brief `off` → `on` pair on
the AC, about half a second apart, before everything settled correctly.
`mode: queued` makes the runs serialise, so the end state is always right — but
the AC receives two commands it didn't need. If yours dislikes that, add
`for: "00:00:02"` to the three mode triggers to debounce the echo, at the cost
of a two-second delay on every mode change.

**The window rules assume the window sensor belongs to the room.** If your
thermostat's open-window detection is temperature-based, it may fire late, or
not at all in mild weather.

---

## Why "leave the away preset" is its own rule

The obvious way to handle presence is a single automation that sets `away` on
every thermostat when the house empties, and clears it when someone comes back.
That has a hole in it, and it is not a rare one.

The clearing rule almost always carries a guard like *"only if the thermostat
is not off"* — sensible, since you don't want to wake a thermostat that is
deliberately off. But if the thermostat happens to be **off at the moment you
come home** — because a schedule switched it off while you were out, which for
most people is a daily occurrence — it is skipped. Nothing clears `away`
afterwards, because the arrival transition has already passed. The preset stays
on the device.

The next time you switch that thermostat on, it starts at the away temperature
instead of yours, and the AC dutifully follows it there.

This blueprint closes the hole from the other side: it clears `away` **at the
moment the thermostat is switched on**, not at the moment you arrive. It only
acts when the house is occupied, only when the preset is still `away`, and
never touches a thermostat that is off — so a `comfort` or `sleep` you chose
yourself survives untouched.

One rough edge worth knowing: when the preset changes, the thermostat restores
its own target temperature and reports it a moment later. The first run may
therefore still push the away temperature to the AC, and the resulting
attribute change immediately triggers a second run that corrects it. It settles
on its own, at the cost of one extra command.

---

## Why bidirectional sync isn't offered

It looks like a small addition. It isn't, and the reasons are worth stating so
you can decide for your own hardware.

**Half the AC's modes don't exist on a thermostat.** `dry`, `fan_only` and
`auto` have no counterpart in `off` / `heat` / `cool`. And the tempting
mapping — *"anything that isn't heat or cool means off"* — is the one that
breaks things: it would switch the thermostat off, which by the forward rule
switches the AC off a second after you asked it for dry mode.

**Writing back the target temperature fights the rounding.** With mismatched
steps, your 21.5 °C becomes 21 °C on the AC, and a return path would then write
21 °C back onto the thermostat — silently discarding what you set. A tolerance
is required, not optional.

**A return path fights the presence rules.** Writing the target temperature
back to the thermostat overwrites whatever preset the presence logic just
applied.

**Unavailable is a value too.** Integrations drop out. A return path has to
ignore `unavailable` and `unknown` explicitly, or it will try to write them.

None of this is unsolvable — but each point is a decision about how you want
your house to behave, not a line of YAML. The alternative worth considering
first is keeping a single source of truth: drive everything from the
thermostat, and hide or make read-only the AC's own controls in your dashboard.

---

## Updating

The blueprint's `source_url` points at this repository, so **Re-import
blueprint** in the Home Assistant UI always fetches the current version of
`main`. Existing automations are reloaded automatically and keep their inputs.

New inputs are always added with a default, so a re-import never requires
editing existing automations.
