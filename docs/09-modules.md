# 9. Modules

Modules is a raw listing of every physical device grouped into a Plant,
and their endpoints — a direct, low-level read/write surface with no
business logic layered on top.

## 9.1 Purpose

Two things, and nothing more:

1. **Render the physical system in the UI** — show the user what hardware
   actually makes up this Plant's group.
2. **Direct hardware access** — let a user read or write a specific
   endpoint value directly, bypassing Vitals/Grow Profile/Scenarios
   entirely. This is an escape hatch for inspection and manual testing,
   not a mechanism the rest of OFL is expected to build on.

## 9.2 Format

```json
{
  "modules": [
    {
      "id": "mod-42",
      "class": "plant_container",
      "is_anchor": true,
      "blueprint_id": "acme-moisture-v1",
      "endpoints": [
        {
          "index": 0,
          "type": "moisture",
          "mode": "read",
          "format": "percent",
          "value": 62
        }
      ]
    }
  ]
}
```

`id` is an opaque, Head-Device-assigned identifier for the module — not
assumed to be a bus address or to carry any positional meaning (see
[Philosophy §1.7](01-philosophy.md#17-independence-from-any-specific-hardwarebus-backend)).
`class` is a value from [Module Classes](18-module-classes.md);
`is_anchor` marks the one module in a Plant's group physically tied to
the container/pot/slot. `blueprint_id` identifies this module's hardware
model — it lives **here and only here**; [Hardware Calibration](06-hardware-calibration.md)
and [Settings](07-settings.md) reference a module's `blueprint_id`
through Modules rather than each carrying their own copy of it (see each
doc's own composition for why). `endpoints` lists each of the module's
raw capabilities: `type`/`format` are from
[Endpoint Types](19-endpoint-types.md), `mode` is `read`/`write`/`rw`, and
`value` is the module's own raw reading — not yet passed through
Hardware Calibration (compare
[Vitals](04-vitals.md)/[Actuators](17-actuators.md), which are). How the
Head Device populates this list from its own internal hardware
representation is outside OFL's scope.

## 9.3 Read/write semantics

A write to an endpoint through Modules is a direct pass-through to the
underlying module — no validation, unit conversion, or calibration is
applied beyond whatever the Head Device's own internal implementation
already guarantees. Modules is intentionally the one place in OFL where a
value is exactly what the module itself reports, with none of the
normalization that [Hardware Calibration](06-hardware-calibration.md)
applies elsewhere.
