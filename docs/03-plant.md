# 3. Plant

A Plant is the entity physically tied to one plant instance, identified by
a `plant_id`. It is the unit of control in OFL: the standard protocol
does not address individual modules independently (for that, see the
deliberately raw escape hatch in [Modules](09-modules.md)) — it addresses
the group of modules assigned to a given `plant_id` as a single Plant.

## 3.1 Identity

| Field | Type | Description |
|---|---|---|
| `plant_id` | `string` | Unique identifier for this Plant within its Garden, assigned by the Head Device. |
| `name` | `string` | User-assigned label (e.g. "Basil #2"). Not derived from hardware. |
| `description` | `string` | User-assigned free text (e.g. "Windowsill, south side"). Empty string if unset. |
| `status` | `enum` | See [System Fields §21.2](21-system-fields.md#212-plant). |

## 3.2 Composition

A Plant is deliberately **thin** — it is a composition of the structures
below, each specified in its own document, referenced here rather than
inlined:

```json
{
  "plant_id": "plant-1234",
  "name": "Basil #2",
  "description": "Windowsill, south side",
  "status": "ok",
  "vitals": { "$ref": "04-vitals.md" },
  "vitals_history": { "$ref": "05-vitals-history.md" },
  "hardware_calibration": { "$ref": "06-hardware-calibration.md" },
  "settings": { "$ref": "07-settings.md" },
  "grow_profile": { "$ref": "08-grow-profile.md" },
  "modules": { "$ref": "09-modules.md" },
  "capabilities": { "$ref": "14-capabilities.md" },
  "actuators": { "$ref": "17-actuators.md" }
}
```

| Structure | Lifecycle | Purpose |
|---|---|---|
| [Vitals](04-vitals.md) | Live, overwritten on read | Current sensor readings |
| [Vitals History](05-vitals-history.md) | Append-only over time | Time-series of past Vitals |
| [Hardware Calibration](06-hardware-calibration.md) | Refreshed wholesale from library | Per-hardware-model normalization (e.g. light flux) |
| [Settings](07-settings.md) | Seeded once from library, then user-owned | Per-installation physical parameters (e.g. mount height) |
| [Grow Profile](08-grow-profile.md) | Independent import, user-editable | Target values for this plant's species |
| [Modules](09-modules.md) | Live, direct passthrough | Raw list of physical modules/endpoints in this group |
| [Capabilities](14-capabilities.md) | Derived, recomputed on module change | What this Plant can do — gates Scenarios/Grow Profile/Culture Pack |
| [Actuators](17-actuators.md) | Live, read/write | Normalized current state of every controllable capability |

Why the Hardware Calibration/Settings split exists, and why it's
independent from Grow Profile, is explained in each of those documents —
this document only fixes that Plant references them rather than defining
them itself.

## 3.3 Grouping mechanism

Membership in a Plant's module group is determined by the Head Device's
own internal module registry, not chosen by OFL: every module the Head
Device has assigned to a given `plant_id` belongs to that Plant. Exactly
one of them is the **anchor**, physically tied to the container/pot/slot;
the rest are non-anchor modules (light, local valve, etc.) grouped with
it. How the Head Device itself determines and persists this assignment
internally is outside OFL's scope — see
[Philosophy §1.7](01-philosophy.md#17-independence-from-any-specific-hardwarebus-backend).
