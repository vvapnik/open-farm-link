# 13. Preferences

Preferences holds garden-wide constraints and flags set by the user,
scoped to exactly **one** Garden — not to an individual Plant or module.

## 13.1 Relationship to Scenarios

Preferences is a **constraint layer**, not an automation rule itself: it
doesn't trigger actions on its own the way a [Scenario](11-scenarios.md)
does. Instead, it bounds what Scenarios are allowed to do — e.g. a
watering Scenario should not run its pump during quiet hours, a lighting
Scenario should not exceed the maximum brightness cap. It does **not**
bound direct access through [Modules](09-modules.md), which is raw and
unconstrained by design (§9.3).

## 13.2 Standard fields

```json
{
  "quiet_hours": { "from": "22:00", "to": "07:00" },
  "max_brightness": 80,
  "vacation_mode": false
}
```

| Field | Type | Description |
|---|---|---|
| `quiet_hours` | `{ from, to }` (`"HH:MM"`, 24h, Head Device local time) or `null` | Window during which noisy/disruptive actuators (pumps, fans) should not run. Read by the generic `watering` Scenario template as `garden.preferences.quiet_hours` (see [Scenarios §11.2](11-scenarios.md#112-generic-scenarios)). `null`/absent means no quiet hours set. Doesn't (yet) handle a window crossing midnight the other way — assume `from` < `to` within one day, or `from` > `to` meaning the window wraps past midnight (e.g. `22:00`–`07:00` above); exact comparison semantics for Scenario clause evaluation aren't spelled out further here. |
| `max_brightness` | `number` (percent, 0–100) or `null` | Ceiling applied across every light module in the Garden, regardless of what an individual Plant's Grow Profile asks for. `null`/absent means no cap. Not yet wired into the generic `lighting` Scenario template's condition/action — flagged here, not resolved. |
| `vacation_mode` | `boolean` | Default `false`. Alters normal automation behavior garden-wide while the user is away — exact effect on Scenario evaluation not yet defined. |

Extensible: more fields can be added later without breaking this shape.
