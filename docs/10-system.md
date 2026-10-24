# 10. System

System groups modules that are shared across every Plant in a Garden —
not tied to any single plant instance. Example: a water tank level
sensor, a mains irrigation pump.

## 10.1 Identity

Modules belonging to System are not assigned to any `plant_id` — there is
no plant instance to anchor to. Unlike Plant, System has no per-instance
identity, and none is needed: **a Garden has exactly one System.** No id
field is required; an optional `name` exists purely for UI labeling.

## 10.2 Composition

System repeats [Plant](03-plant.md)'s composition, minus
[Grow Profile](08-grow-profile.md) — see §10.3 for why. It reuses the
same structure documents; nothing about
[Vitals](04-vitals.md)/[Vitals History](05-vitals-history.md)/
[Hardware Calibration](06-hardware-calibration.md)/
[Settings](07-settings.md)/[Modules](09-modules.md) is Plant-specific.

```json
{
  "name": "Main System",
  "vitals": { "$ref": "04-vitals.md" },
  "vitals_history": { "$ref": "05-vitals-history.md" },
  "hardware_calibration": { "$ref": "06-hardware-calibration.md" },
  "settings": { "$ref": "07-settings.md" },
  "modules": { "$ref": "09-modules.md" },
  "capabilities": { "$ref": "14-capabilities.md" },
  "actuators": { "$ref": "17-actuators.md" }
}
```

System's `capabilities` is System's own module-derived flags only — it is
what [Garden-level Capabilities](14-capabilities.md#141-two-sources-two-scopes)
means, since a Garden has exactly one System. It does not include the
Head Device's own `engine.*` flags directly; those are unioned in at
validation time from [Garden](12-garden.md), not stored twice.

## 10.3 Why no Grow Profile

[Grow Profile](08-grow-profile.md) expresses target values for the plant
species occupying a Plant's Anchor module. System modules aren't
associated with any plant or species, so there is nothing for a Grow
Profile to target. System modules are expected to be driven by
[Scenarios](11-scenarios.md) reading across multiple Plants (e.g. "if any
Plant currently wants water, run the shared pump"), not by targets of
their own.
