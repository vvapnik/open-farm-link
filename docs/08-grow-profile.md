# 8. Grow Profile

Grow Profile holds the target values for the plant species/instance
occupying a Plant — what the Scenarios acting on this Plant should aim
for, independent of which hardware happens to be installed.

## 8.1 Example

```json
{
  "species": "basil",
  "watering_enabled": true,
  "lighting_enabled": true,
  "climate_enabled": true,
  "targets": {
    "moisture_percent": { "min": 55, "max": 70 },
    "light_lux": 4000,
    "light_hours_per_day": 14
  },
  "sensor_calibration": {
    "moisture": { "offset": -8 }
  },
  "required_capabilities": ["moisture"]
}
```

Target values here are expressed in real physical units (percent, lux,
hours) — never in a specific module's raw command range. Translating a
target into an actual command for the installed hardware is
[Hardware Calibration](06-hardware-calibration.md)'s job, not Grow
Profile's.

## 8.2 Standard fields

| Field | Type | Description |
|---|---|---|
| `species` | `string` | Free-form species identifier, also the library import key (§8.4). |
| `<category>_enabled` | `bool` | One flag per generic Scenario category (`watering_enabled`, `lighting_enabled`, `climate_enabled`, ...) — see §8.3. |
| `targets.moisture_percent` | `{ min, max }` | Target soil moisture range, in the [Capability Catalog](20-capability-catalog.md)'s `moisture` unit (percent). |
| `targets.light_lux` | `number` (lux) | Target illuminance, [Capability Catalog](20-capability-catalog.md) `dimmer`/`light` unit. |
| `targets.light_hours_per_day` | `number` | Target daily light duration. |
| `sensor_calibration.<key>` | `{ offset }` (or similar, TBD) | Per-substrate correction applied on top of a [Capability Catalog](20-capability-catalog.md) reading — see §8.5. Open for more keys/shape as needed. |
| `required_capabilities` | array of [Capability Catalog](20-capability-catalog.md) keys | Checked at add-time — see [Capabilities §14.4](14-capabilities.md#144-validation-lifecycle). |

## 8.3 `*_enabled` flags

One per generic Scenario category, not a single master switch — turning
off `watering_enabled` must not also silence `lighting_enabled`. Each is
read as an ordinary condition clause by that category's generic Scenario
template ([Scenarios §11.2](11-scenarios.md#112-generic-scenarios)); there
is no separate "is this automation on" mechanism beyond this flag. Turning
one off is how a user hands full manual control of that category to their
own [specialized Scenario](11-scenarios.md#113-specialized-scenarios)
instead — see
[Scenarios §11.5](11-scenarios.md#115-no-automatic-precedence-between-generic-and-specialized)
for why the engine doesn't do this automatically.

## 8.4 Optional library import

Grow Profile MAY be imported from a plant library entry (e.g. a
pre-defined "basil" profile), the same way Hardware Calibration and
Settings entries are seeded from a hardware library entry looked up by a
module's `blueprint_id` — but Grow Profile comes from a *plant* library,
keyed by `species`, not a hardware library keyed by `blueprint_id`. Import
is optional: a user can set every target value directly without ever
touching a library.

## 8.5 `sensor_calibration` — distinct from Hardware Calibration

`sensor_calibration` corrects for the **growing substrate**, not the
hardware model — a moisture sensor reads differently depending on what
it's actually inserted into (soil vs. coco coir vs. rockwool have
different dielectric properties), independent of which sensor module is
used. This is a different axis from
[Hardware Calibration](06-hardware-calibration.md), which corrects for
**hardware model** differences and knows nothing about what's planted.
Both corrections apply to the same underlying reading, in sequence,
before it reaches [Vitals](04-vitals.md) — exact composition order and
the correction's shape beyond a simple `offset` are not yet finalized.

## 8.6 Independent from Hardware Calibration / Settings

Despite the similar "optional library import, then user-editable" shape,
Grow Profile is a genuinely separate structure from
[Hardware Calibration](06-hardware-calibration.md) and
[Settings](07-settings.md), not a variant of the same mechanism:

- Hardware Calibration / Settings describe the **hardware** in this
  Plant's group, referencing a module's `blueprint_id` from
  [Modules](09-modules.md) rather than carrying identity of their own.
- Grow Profile is keyed by plant species — it describes the **plant**
  occupying the anchor module, entirely independent of which hardware
  happens to be attached to it.

A Plant's Grow Profile can change (e.g. the user replants with a
different species) without touching Hardware Calibration or Settings at
all, and vice versa (the user swaps a light module for a different model
without changing what's planted).

## 8.7 Relationship to Culture Pack

A Grow Profile can also arrive bundled with one or more specialized
Scenarios as a single import — see [Culture Pack](15-culture-pack.md).
That bundling is a separate entity, not a variant of Grow Profile itself:
a plain Grow Profile import never touches Scenarios, and Grow Profile has
no awareness that it might be part of a Culture Pack.
