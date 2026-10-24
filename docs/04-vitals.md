# 4. Vitals

Vitals is the current-state snapshot of every sensor reading available for
a Plant, in a standardized but extensible format.

## 4.1 Vitals has its own type registry

Vitals `type` values are keyed by the
**[Capability Catalog](20-capability-catalog.md)** (sensor keys, §20.1) —
not by whatever internal sensor/endpoint identifiers a Head Device's own
hardware uses. The two are not required to correspond one-to-one: a
single Vitals entry may aggregate several physical sensors in the same
Plant group (e.g. two moisture sensors averaged into one reading), or be
computed from more than one underlying input (e.g. VPD computed from
temperature and humidity), or have no direct hardware source at all. How
a Head Device derives a Vitals value from its own internal readings is an
implementation detail; this document only fixes the *shape* of a Vitals
entry once produced.

## 4.2 Format

```json
{
  "vitals": [
    { "type": "moisture", "value": 62, "unit": "percent" },
    { "type": "temp", "value": 21.4, "unit": "celsius" },
    { "type": "light", "value": 3400, "unit": "lux" }
  ]
}
```

- `type` — a value from the [Capability Catalog](20-capability-catalog.md),
  extensible (vendor range) for new sensor kinds without breaking the
  format.
- `value` — numeric reading, already normalized to `unit` via
  [Hardware Calibration](06-hardware-calibration.md) where the source
  module has an entry — never a raw endpoint value.
- `unit` — explicit, not implied by `type` alone, so the format tolerates
  a future type whose natural unit isn't yet anticipated.

An entry is only present if that Vitals type is currently available for
the Plant — no entry for un-populated sensor kinds. Not a fixed-width
record; Vitals lives above the resource constraints that motivate
fixed-width formats elsewhere in this spec.
