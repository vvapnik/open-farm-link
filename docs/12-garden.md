# 12. Garden

Garden is the top-level entity: the entire garden block tied to **one**
Head Device. It is the unit this protocol is scoped to — OFL mostly
describes talking to a single Head Device's Garden, not to a fleet of
them.

## 12.1 Composition

Like [Plant](03-plant.md#32-composition), Garden is thin — it references
the structures below rather than inlining them:

```json
{
  "garden_id": "mfrggzdfmztwq2lk",
  "name": "Kitchen Rack",
  "description": "Basement grow rack, 3 tiers",
  "protocol_version": "0.1",
  "status": "ok",
  "type": "ogr",
  "plants": [
    { "$ref": "03-plant.md" }
  ],
  "system": { "$ref": "10-system.md" },
  "preferences": { "$ref": "13-preferences.md" },
  "scenarios": [
    { "$ref": "11-scenarios.md" }
  ],
  "engine_capabilities": { "$ref": "14-capabilities.md" }
}
```

| Field | Type | Description |
|---|---|---|
| `garden_id` | `string` | Uniquely identifies a Garden/Head Device across the network: 16 lowercase base32 characters derived from the device's public key — see [Security §22.2](22-security.md#222-device-identity-and-transport). Read-only. Changes on factory reset ([§22.8](22-security.md#228-factory-reset)), which makes the device a new Garden. |
| `name` | `string` | User-assigned label. |
| `description` | `string` | User-assigned free text. Empty string if unset. |
| `protocol_version` / `status` / `type` | see [System Fields §21.1](21-system-fields.md#211-garden) | |
| `plants` | array of [Plant](03-plant.md) | Every Plant in this Garden. **Unordered** — per [Philosophy §1.7](01-philosophy.md#17-independence-from-any-specific-hardwarebus-backend), this list carries no positional meaning; any physical layout, where it matters, is only exposed through each Plant's own [Modules](09-modules.md). |
| `system` | [System](10-system.md) | The one shared-resources entity for this Garden. |
| `preferences` | [Preferences](13-preferences.md) | Garden-wide user constraints/flags (quiet hours, brightness cap, vacation mode, ...) that bound what Scenarios are allowed to do. |
| `scenarios` | array of [Scenario](11-scenarios.md) | Every generic and specialized Scenario known to this Garden — generic templates included, since there's exactly one shared record per category rather than one per Plant. |
| `engine_capabilities` | [Capabilities](14-capabilities.md) (`engine` list only) | Firmware-declared feature flags for this Head Device (e.g. `engine.impulse_watering`). Unioned into every Plant's and into System's available capabilities at validation time — see [Capabilities §14.1](14-capabilities.md#141-two-sources-two-scopes). |

Garden's own effective Capabilities, for a Garden-scoped
`specialized`/`script` Scenario, is `system.capabilities ∪
engine_capabilities` — there is no separate stored "Garden capabilities"
beyond that union.

## 12.2 One Garden per Head Device; many Gardens per Control Panel

A Control Panel is expected to manage an effectively unbounded number of
Gardens — one per Head Device it talks to. This protocol, however, mostly
describes the interaction between a Control Panel and a *single* Garden's
Head Device: **bulk operations across multiple Gardens (e.g. "water every
basil plant across every garden") are a Control Panel-side concern**, built
on top of repeated single-Garden operations, not a feature OFL itself
provides.
