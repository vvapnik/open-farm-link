# 19. Standard Endpoint Types

The standard catalog for a [Modules](09-modules.md) endpoint's `type`
field — the module's own raw value, before any
[Hardware Calibration](06-hardware-calibration.md) translation. Each
entry notes which [Capability Catalog](20-capability-catalog.md) key it
normalizes to under the standard mapping.

## 19.1 Sensors

| Type | Raw `format` | Normalizes to |
|---|---|---|
| `moisture` | module-defined range | `moisture` |
| `temp` | module-defined range | `temp` |
| `humidity` | module-defined range | `humidity` |
| `light` | module-defined range | `light` |
| `weight` | module-defined range | `weight` |
| `distance` | module-defined range | `distance` |
| `water_level` | module-defined range | `water_level` |

## 19.2 Actuators

| Type | Raw `format` | Normalizes to |
|---|---|---|
| `relay` | binary | `relay` |
| `dimmer` | 0–100 | `dimmer` |
| `valve` | binary | `valve` |
| `light_switch` | binary | `light_switch` |
| `pump` | 0–100 | `pump` |
| `vibro` | 0–100 | `vibro` |

## 19.3 Widgets

| Type | Raw `format` | Notes |
|---|---|---|
| `camera` | binary | Flags an external streaming module; the stream itself is out-of-band, outside this protocol's scope. |

## 19.4 Vendor extension

Open beyond this list, same convention as
[Module Classes §18.1](18-module-classes.md#181-vendor-extension) — a
vendor's own endpoint type passes through unvalidated, resolved via its
`blueprint_id` library entry.
