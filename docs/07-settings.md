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
| `mount_height_mm` | `uint32` | Distance between a light module and the pot. The only standard field so far — open to extension as more installation-geometry cases come up. |

## 7.4 Relationship to the zero-configuration principle

Per [Philosophy §1.5](01-philosophy.md#15-zero-configuration-ecosystem),
the default flow never requires a user to open Settings at all — the
imported defaults describe the standard kit assembly. Settings is only
touched when the physical installation deviates from that standard (e.g.
extra spacers were added), at which point the user corrects the relevant
module's value through the UI.
