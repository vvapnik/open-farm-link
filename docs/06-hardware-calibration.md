# 6. Hardware Calibration

Hardware Calibration normalizes *between different hardware models* that
expose the same standardized command/reading range, so the rest of OFL
(in particular [Grow Profile](08-grow-profile.md)) can target real-world
physical units without knowing which specific module is installed.

## 6.1 What this is not

Raw sensor/actuator physical calibration (e.g. a moisture sensor's
dry/wet reference points) is **not** part of this structure — that
calibration happens inside the Head Device's own module-level
implementation, using points kept in that module's
[Settings](07-settings.md#73-standard-fields) (`moisture_dry_raw` /
`moisture_wet_raw`). A module's endpoint values, as OFL sees them, are already
in a standardized range (e.g. 0–100%). OFL never sees, and does not need,
a module's raw sensor values.

## 6.2 What this is

Two modules of the same endpoint type (e.g. two different light module
models, both exposing a 0–100% dimmer endpoint) can produce different
real-world output at the same commanded value — one module's "100%" may be
brighter than another's. Hardware Calibration is the table that lets OFL
convert between a module's standardized command range and a real physical
quantity, **per hardware model**.

## 6.3 Format — a list, keyed by `blueprint_id` + `endpoint_type`

A Plant or System's module group can contain several distinct hardware
models at once (a moisture sensor, a light, a valve — each its own
`blueprint_id`, from [Modules](09-modules.md)). Hardware Calibration is
therefore a **list**, one entry per distinct `(blueprint_id,
endpoint_type)` pair actually present — not a single object:

```json
{
  "calibrations": [
    {
      "blueprint_id": "acme-light-v2",
      "endpoint_type": "dimmer",
      "calibration": {
        "curve": [
          { "command": 0, "physical": 0, "unit": "lux_at_reference_distance" },
          { "command": 50, "physical": 1800, "unit": "lux_at_reference_distance" },
          { "command": 100, "physical": 4200, "unit": "lux_at_reference_distance" }
        ]
      }
    }
  ]
}
```

`blueprint_id` here is not this structure's own identifier — it's a
reference to the same value already present on the relevant module(s) in
[Modules](09-modules.md#92-format). Resolving a calibration for a
specific endpoint means: look up that endpoint's module in Modules, read
its `blueprint_id`, then find the entry here matching that `blueprint_id`
+ the endpoint's `type`. The exact curve/table shape is illustrative, not
final — the point fixed at this stage is this lookup structure and the
lifecycle rule in §6.4, not the calibration data format itself.

## 6.4 Source and refresh: library-owned, not user-owned

Each entry is imported from the hardware-model library entry identified
by `blueprint_id` — a catalog of known hardware models, each describing
what it is and how its endpoints normalize (library import itself: not
yet implemented, deferred). An entry is **not** user-editable, and is
**replaced wholesale** whenever the library entry for that `blueprint_id`
is updated (e.g. the manufacturer publishes a corrected calibration
curve).

This is the deliberate counterpart to [Settings](07-settings.md), which is
seeded from the same library import but never auto-refreshed afterward.
Splitting the two avoids needing to track which individual fields a user
has hand-edited before deciding what a library update is allowed to
touch — see [Settings §7.2](07-settings.md#72-why-this-is-a-separate-structure-from-hardware-calibration)
for the full rationale.
