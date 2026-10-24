# 18. Standard Module Classes

The standard catalog for a [Modules](09-modules.md) entry's `class`
field.

| Class | Description | Typical role |
|---|---|---|
| `plant_container` | Tied directly to one plant slot/pot/hydroponic container. | Usually the anchor module for its Plant. |
| `light` | Illumination. | Plant-local or System. |
| `water_supply` | Irrigation, dosing, water management (pumps, valves, level sensors, EC/pH). | Usually System; can be Plant-local (e.g. a plant's own drip valve). |
| `climate` | Environmental monitoring/control (temp/humidity/CO2, fans, heaters). | System or Plant. |
| `custom` | Vendor-specific or experimental, not fitting a standard class. | Either. |

## 18.1 Vendor extension

Open beyond this list, same convention as
[Capabilities §14.2](14-capabilities.md#142-standard-range--open-vendor-range):
a vendor can introduce its own `class` value for a module not covered
above. It passes through unvalidated; a Control Panel needs that vendor's
library entry (`blueprint_id`) to render it meaningfully.
