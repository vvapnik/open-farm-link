# 20. Capability Catalog

The single standard vocabulary shared by [Vitals](04-vitals.md) `type`,
[Actuators](17-actuators.md) `capability`, and the module-derived
(standard-range) flags in [Capabilities](14-capabilities.md) — one key
means the same thing in all three. This is deliberate: a hardware
capability flag like "has a moisture sensor" must be the identical key
Vitals reports readings under and Scenarios reference in
`required_capabilities`, not a parallel identifier that happens to look
similar.

This is a distinct, higher layer than [Endpoint Types](19-endpoint-types.md):
Endpoint Types describe what a module raw-reports; entries here are
already normalized to real physical units via
[Hardware Calibration](06-hardware-calibration.md). Most keys correspond
1:1 to an Endpoint Type (see each type's "Normalizes to" column), but
that correspondence isn't required — a Vitals reading may aggregate
multiple endpoints or be computed with no direct endpoint at all (see
[Vitals §4.1](04-vitals.md#41-vitals-has-its-own-type-registry)).

## 20.1 Sensor keys (`kind: sensor`)

| Key | Unit | Used by |
|---|---|---|
| `moisture` | percent | Vitals, Capabilities |
| `temp` | celsius | Vitals, Capabilities |
| `humidity` | percent | Vitals, Capabilities |
| `light` | lux | Vitals, Capabilities |
| `weight` | gram | Vitals, Capabilities |
| `distance` | millimeter | Vitals, Capabilities |
| `water_level` | percent | Vitals, Capabilities |

## 20.2 Actuator keys (`kind: actuator`)

| Key | Unit | Used by |
|---|---|---|
| `relay` | binary | Actuators, Capabilities |
| `dimmer` | lux (or the target quantity the light module controls) | Actuators, Capabilities |
| `valve` | binary | Actuators, Capabilities |
| `light_switch` | binary | Actuators, Capabilities |
| `pump` | percent | Actuators, Capabilities |
| `vibro` | percent | Actuators, Capabilities |

## 20.3 Scope prefix, not a separate key

A key referenced from a [Scenario](11-scenarios.md) may be prefixed with
`system.` (e.g. `system.pump`) to mean "this key, on System" rather than
on the Scenario's own evaluating target. The prefix is a scope modifier
applied where a key is *used*, not a distinct catalog entry — `pump` and
`system.pump` are the same capability at two different scopes.

## 20.4 `engine.*` is a separate namespace

Firmware/engine feature flags (`engine.impulse_watering`,
`engine.custom_endpoints`, ...) are not part of this catalog — they
describe what the Head Device's firmware can do, not a sensor or actuator
capability. See [Capabilities §14.1](14-capabilities.md#141-two-sources-two-scopes).

## 20.5 Vendor extension

Open beyond this list, same convention as
[Module Classes §18.1](18-module-classes.md#181-vendor-extension).
