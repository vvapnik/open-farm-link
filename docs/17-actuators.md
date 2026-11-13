# 17. Actuators

Actuators is the write-side counterpart to [Vitals](04-vitals.md): a
normalized, current-state view of every controllable capability available
for a Plant or System, keyed by the **[Capability Catalog](20-capability-catalog.md)**
(actuator keys, §20.2) — the same vocabulary
[Capabilities](14-capabilities.md) and
[Scenario actions](11-scenarios.md#114-condition-format) already use, so
a Control Panel can read and set an actuator without knowing which
physical module or endpoint backs it.

## 17.1 Why this exists alongside Modules

[Modules](09-modules.md) already allows direct read/write of any
endpoint, but only by raw `module_id` + endpoint `index` — the caller
has to already know which physical module implements a given capability,
and the value exchanged is the module's raw, uncalibrated range. Actuators
is the normalized equivalent of that same underlying state, addressed by
capability and expressed in real physical units. The two are two views of
the same state, not independent copies: writing through either one
changes what the other reports.

This is not just a UI convenience — it's the interface
[Scenario actions](11-scenarios.md) execute through. A Scenario never
writes to Modules directly (see [Scenarios](11-scenarios.md)); "executing
an action" *is* an Actuators write. This is what makes a Scenario, and by
extension a [Grow Profile](08-grow-profile.md) or
[Culture Pack](15-culture-pack.md) built on one, portable across
different hardware unchanged.

## 17.2 Format

```json
{
  "actuators": [
    { "capability": "valve", "value": 1, "unit": "binary" },
    { "capability": "dimmer", "value": 1800, "unit": "lux" }
  ]
}
```

- `capability` — a key from the [Capability Catalog](20-capability-catalog.md).
- `value` — current commanded state, in real physical units (see §17.3),
  never the module's raw standardized command range.
- `unit` — explicit, same convention as [Vitals](04-vitals.md#42-format).
- `min`, `max` — optional: the range a write can actually reach, in `unit`
  (e.g. the measured range of a light's `dimmer_lux_curve`,
  [Settings §7.3](07-settings.md#73-standard-fields)). A client bounds its
  controls by them; a write outside is clamped.

A Head Device that cannot yet convert a capability to its physical unit
(e.g. a light with no measured curve) reports the module's own normalized
range honestly — `"unit": "percent"` with `min: 0, max: 100` — rather than
labelling it with the catalog's physical unit.

An entry is only present if that capability is currently available for
the Plant/System (same presence rule as Vitals — see
[Vitals §4.2](04-vitals.md#42-format)).

## 17.3 Read/write semantics

`value` is a **high-level, physical-unit value** (e.g. lux, not a 0–100%
command) — never the module's raw standardized command range. Converting
between the two is [Hardware Calibration](06-hardware-calibration.md)'s
job, applied by the Head Device on every read and write:

- `GET` reads the module's raw endpoint value and converts it through
  Hardware Calibration before returning it, the same way
  [Vitals](04-vitals.md) already does for sensors.
- `PUT` converts the supplied physical value through Hardware Calibration
  into the module's raw command range, then writes it to the underlying
  module/endpoint the same way a Scenario action would.

A capability with no Hardware Calibration entry for its module (e.g. a
plain binary relay, where "physical value" and "command" already
coincide) passes the value through unchanged.

