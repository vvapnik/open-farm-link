# 15. Culture Pack

A Culture Pack bundles a [Grow Profile](08-grow-profile.md) together with
one or more [specialized Scenarios](11-scenarios.md#113-specialized-scenarios)
for a single library import. It is a distinct entity from Grow Profile,
not a fancier variant of it — see §15.2.

## 15.1 Format

```json
{
  "id": "tomato-advanced",
  "grow_profile": { "$ref": "08-grow-profile.md" },
  "disables": ["watering_enabled"],
  "scenarios": [
    { "$ref": "11-scenarios.md" }
  ],
  "required_capabilities": ["valve", "moisture", "weight"]
}
```

| Field | Description |
|---|---|
| `grow_profile` | The bundled [Grow Profile](08-grow-profile.md) targets. |
| `disables` | Which generic-scenario category enable flags this pack turns off in the imported Grow Profile (e.g. a pack replacing standard watering with a weight-aware specialized scenario disables `watering_enabled`). See §15.3. |
| `scenarios` | The bundled [specialized Scenario](11-scenarios.md) record(s). |
| `required_capabilities` | Checked as a whole at import time, independently of the fact that the bundled Scenario(s) also carry their own — see [Capabilities §14.5](14-capabilities.md#145-required_capabilities-appears-on-three-entities). |

## 15.2 Why this is a separate entity from Grow Profile

Grow Profile is a plain declaration (targets only) that can be imported,
edited, or written from scratch with no relationship to Scenarios at all.
Culture Pack is specifically the *combination* of a Grow Profile with
behavior that goes beyond what the generic scenario catalog can express
(e.g. weight-aware watering). A user who just wants standard automation
with species-appropriate targets only ever touches Grow Profile — Culture
Pack is strictly for the case that needs custom Scenario logic alongside
it.

## 15.3 Import touches two structures at once

Importing a Culture Pack is one operation with two effects: it writes the
bundled Grow Profile's targets, **and** sets each category flag named in
`disables` to `false` in that same Grow Profile, **and** attaches the
bundled Scenario(s). This is a property of the import mechanism, not a
Scenario-engine feature — nothing about Scenario evaluation itself knows
a Culture Pack exists.

## 15.4 Open question

What happens to an already-imported pack's bundled Scenario(s) when the
library entry for that Culture Pack is later updated — wholesale refresh
(like [Hardware Calibration](06-hardware-calibration.md#64-source-and-refresh-library-owned-not-user-owned)),
or seed-once-then-user-owned (like
[Settings](07-settings.md#72-why-this-is-a-separate-structure-from-hardware-calibration))?
Not yet decided.
