# 21. Garden and Plant System Fields

Standard service fields on [Garden](12-garden.md) and [Plant](03-plant.md)
themselves, beyond the identity fields already listed in each entity's own
doc.

## 21.1 Garden

| Field | Type | Description |
|---|---|---|
| `protocol_version` | `string` | The OFL protocol version this Head Device implements (e.g. `"0.1"`). Read-only. |
| `status` | `enum: ok, problem` | `problem` if System or any Plant currently has a module in a faulted state. Aggregate signal only — see each entity's own `capabilities`/`modules` for detail. |
| `type` | `string` | Identifies the Head Device's hardware/bus backend (e.g. `"ogr"`). Read-only, informational — see §21.4. |

## 21.4 `type` does not create a backend dependency

`type` is a self-description a Control Panel MAY use to render
backend-aware UI (e.g. a physical rack schematic that knows how to lay
out an OGR-style rack) — it does not change what OFL itself requires or
guarantees. A Control Panel that doesn't recognize a given `type` value
still has full read/write access to every Garden through
[Vitals](04-vitals.md)/[Actuators](17-actuators.md)/
[Capabilities](14-capabilities.md)/[Grow Profile](08-grow-profile.md) —
`type` is an enhancement hint, never a requirement, consistent with
[Philosophy §1.7](01-philosophy.md#17-independence-from-any-specific-hardwarebus-backend).

The catalog is open, same convention as
[Module Classes §18.1](18-module-classes.md#181-vendor-extension) —
`"ogr"` is the first known value (a Head Device built on
[OpenGardenRack](https://github.com/vvapnik/open-garden-rack)'s I2C bus
internally); anything else passes through unvalidated.

Note this does **not** give a Control Panel positional/physical-layout
data through the standard protocol — [Philosophy §1.7](01-philosophy.md#17-independence-from-any-specific-hardwarebus-backend)
still holds: `plants` stays unordered, and a whole-rack physical schematic
either falls back to API list order or requires the Head Device to expose
that separately (not yet specified).

## 21.2 Plant

| Field | Type | Description |
|---|---|---|
| `status` | `enum: ok, problem` | `problem` if any module in this Plant's group is currently faulted or unreachable. |

## 21.3 Why `status` is aggregate-only

Both `status` fields are a cheap, single-field health signal for a
Control Panel's summary UI (e.g. a dashboard showing which Gardens/Plants
need attention) — not a diagnostic. Finding out *what* is wrong means
reading [Modules](09-modules.md) for the affected Plant/System.
