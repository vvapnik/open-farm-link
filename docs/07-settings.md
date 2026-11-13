# 7. Settings

Settings holds physical installation parameters for a Plant's modules —
values that depend on how *this specific installation* was actually put
together, not on the hardware model in the abstract.

## 7.1 Format — a list, keyed by `module_id`

Unlike [Hardware Calibration](06-hardware-calibration.md) (model-level,
keyed by `blueprint_id`), Settings is **instance-level**: two modules of
the identical model, in two different Plants, can be mounted at different
heights and need different values. Settings is therefore a list keyed by
`module_id` (from [Modules](09-modules.md)), not by `blueprint_id` — it
does not carry `blueprint_id` itself as a field at all.

```json
{
  "settings": [
    {
      "module_id": "mod-42",
      "settings": {
        "mount_height_mm": 300
      }
    }
  ]
}
```

`mount_height_mm` here is the distance between a light module and the pot
— needed (together with [Hardware Calibration](06-hardware-calibration.md))
to translate a [Grow Profile](08-grow-profile.md) target expressed in real
units (lux at the plant) into the correct command value for the installed
light module. Unlike Hardware Calibration's curve, this number isn't a
property of the light module model — it's a property of how far apart the
user physically mounted this particular light and this particular pot.

## 7.2 Why this is a separate structure from Hardware Calibration

Both structures get their initial values from the same library entry —
looked up by the module's `blueprint_id` ([Modules](09-modules.md)) — the
library ships sensible defaults for a standard kit assembly (e.g. the
default rack spacing). The two structures diverge after that import:

- **Hardware Calibration** stays library-owned forever: it's replaced
  wholesale on every library update, and the user never edits it directly
  (§6.4).
- **Settings** is library-owned only at import time ("Default Settings"),
  and becomes user-owned from then on, per `module_id`. A later library
  update to the same `blueprint_id` does **not** touch an already-seeded
  Settings entry, even for a field the user never touched.

This split exists specifically to avoid field-level merge conflicts. If
both concerns lived in one structure, a library update would need to know
which individual fields the user had hand-edited (e.g. after adding
spacers) versus which still held the imported default, in order to decide
what's safe to overwrite. Splitting by *entire-entry* refresh semantics
instead of *per-field* tracking removes that problem: refreshing Hardware
Calibration is always safe because nothing user-owned lives there, and a
Settings entry is never touched by a refresh at all, so there is nothing
to reconcile.

## 7.3 Standard fields

| Field | Type | Description |
|---|---|---|
| `mount_height_mm` | `uint32` | Distance between a light module and the pot. Open to extension as more installation-geometry cases come up. |
| `moisture_dry_raw` | `uint16` | On a module whose `moisture` endpoint reports a raw reading: that reading with the dry pot in air — maps to 0 %. See [Hardware Calibration §6.1](06-hardware-calibration.md#61-what-this-is-not). |
| `moisture_wet_raw` | `uint16` | The same reading with the sensor in water — maps to 100 %. Must differ from `moisture_dry_raw`. |
| `dimmer_lux_curve` | array of `{ command, lux }`, up to 8 | On a light module: illuminance measured with a lux meter at the plant for given commands (`command` in the module's 0–100 % range). Sorted by `command`; `lux` must rise strictly with it. With two or more points the Head Device reports and accepts the `dimmer` actuator in lux, piecewise-linear and clamped to the measured range ([Actuators §17.2](17-actuators.md#172-format)). |

`moisture_dry_raw`/`moisture_wet_raw` and `dimmer_lux_curve` belong here
rather than in Hardware Calibration because they depend on the installed sensor and pot wall
(or the light's mount height), and a user re-measures them in place. Their defaults (§7.2) are typical points for
the sensor model, so an uncalibrated pot still reads approximately; a
production batch can be calibrated once on a reference pot and the same
pair written to every pot of the batch.

## 7.4 Relationship to the zero-configuration principle

Per [Philosophy §1.5](01-philosophy.md#15-zero-configuration-ecosystem),
the default flow never requires a user to open Settings at all — the
imported defaults describe the standard kit assembly. Settings is only
touched when the physical installation deviates from that standard (e.g.
extra spacers were added), at which point the user corrects the relevant
module's value through the UI.
