# 11. Scenarios

A Scenario is a declarative automation rule: a condition, evaluated
against current state, gates one or more actions. The Head Device
evaluates every applicable Scenario on its own control loop, without any
Control Panel involvement — this is the concrete mechanism behind
[Philosophy §1.3](01-philosophy.md#13-head-device-autonomy).

A Scenario never touches [Modules](09-modules.md) — it reads
[Vitals](04-vitals.md)/[Grow Profile](08-grow-profile.md)/
[Settings](07-settings.md)/[Preferences](13-preferences.md) and writes
through [Actuators](17-actuators.md), both already normalized to real
physical units via [Hardware Calibration](06-hardware-calibration.md).
This is what makes a [Grow Profile](08-grow-profile.md) or
[Culture Pack](15-culture-pack.md) portable across different hardware
unchanged: a Scenario written once runs identically regardless of which
specific module implements a given capability underneath.

## 11.1 Three kinds

| `kind` | Stored per-target? | Condition/action shape | Who can produce it |
|---|---|---|---|
| `generic` | No — one shared template per category, evaluated against every matching target each tick | `condition` + `actions` | Built into firmware (the standard automation catalog) |
| `specialized` | Yes — bound to one explicit `target` | `condition` + `actions` | User, or bundled with a [Culture Pack](15-culture-pack.md) |
| `script` | Yes — bound to one explicit `target` | `language` + `source` (no declarative condition/actions) | User or third party, gated by an `engine.*` capability |

## 11.2 Generic scenarios

A generic scenario is not materialized once per Plant/System — that would
multiply storage by target count for no benefit, since every target
already carries its own [Grow Profile](08-grow-profile.md)/
[Preferences](13-preferences.md) values to parameterize the same shared
logic. Instead there is exactly one stored record per category, and the
engine evaluates it against every target `applies_to` names, each tick,
substituting that target's own field values.

```json
{
  "id": "watering",
  "kind": "generic",
  "category": "watering",
  "applies_to": "plant",
  "condition": {
    "combinator": "all",
    "clauses": [
      { "field": "grow_profile.watering_enabled", "op": "eq", "value": true },
      { "field": "vitals.moisture", "op": "lt", "value": { "field": "grow_profile.targets.moisture_percent.min" } },
      { "field": "time", "op": "not_in", "value": { "field": "garden.preferences.quiet_hours" } }
    ]
  },
  "actions": [
    { "capability": "valve", "value": 1, "duration_s": 6 },
    { "capability": "system.pump", "value": 1, "duration_s": 5 }
  ],
  "required_capabilities": ["valve", "moisture"]
}
```

The enable/disable switch for a generic scenario is not a separate
mechanism — it's just `grow_profile.watering_enabled` (or the equivalent
field for another category) evaluated as an ordinary clause. See §11.5 for
why there is deliberately no other precedence mechanism.

## 11.3 Specialized scenarios

Bound to one explicit `target` (Plant, System, or Garden — not
Plant-only). Same `condition`/`actions` shape as generic, plus a
persisted `status`, since — unlike a generic template — a specific stored
instance can be marked `noop` by capability revalidation (§12) and needs
somewhere to remember that between ticks.

```json
{
  "id": "scn-8891",
  "kind": "specialized",
  "category": "watering",
  "target": { "type": "plant", "id": "plant-1234" },
  "status": "active",
  "condition": {
    "combinator": "all",
    "clauses": [
      { "field": "vitals.moisture", "op": "lt", "value": { "field": "grow_profile.targets.moisture_percent.min" } },
      { "field": "vitals.pot_weight", "op": "lt", "value": 800 }
    ]
  },
  "actions": [
    { "capability": "valve", "value": 1, "duration_s": 6 },
    { "capability": "system.pump", "value": 1, "duration_s": 5 }
  ],
  "required_capabilities": ["valve", "moisture", "weight"]
}
```

A complementary specialized scenario (different trigger surface, e.g. a
weekly deep-flush) can coexist with its category's generic template left
enabled. A *replacing* one (same trigger surface) requires whoever
configures it to turn the generic template's own enable flag off — the
engine does not detect or prevent running both. See §11.5.

## 11.4 `condition` format

```json
{
  "combinator": "all",
  "clauses": [
    { "field": "<dotted path>", "op": "<operator>", "value": <literal> | { "field": "<dotted path>" } }
  ]
}
```

- **`combinator`** — `all` or `any`, applies to the entire clause list.
  Deliberately **flat, no nested combinators** (no `(A or B) and C`) — a
  scenario needing that is a candidate for `kind: "script"` (§11.6), not
  for stretching this format. This keeps evaluation a single bounded loop
  with no recursion.
- **Bound:** a scenario has at most 8 clauses and at most 4 actions. A
  Garden has at most 32 Scenarios total (generic + specialized + script
  combined). Both ceilings are arbitrary, not derived from a hardware
  limit — just concrete enough that implementations can use fixed arrays
  instead of dynamic allocation.
- **`field`** — a dotted path into the evaluating target's own data:
  `vitals.*`, `grow_profile.*`, `settings.*`; `garden.preferences.*` and
  `system.vitals.*` reach outside the target itself. `time` is a reserved
  pseudo-field (current time-of-day), not part of any entity — needed for
  quiet-hours/time-window checks.
- **`op`** — one of `eq`, `ne`, `lt`, `lte`, `gt`, `gte`, `in`, `not_in`.
- **`value`** — a literal, or `{ "field": "<dotted path>" }` to compare
  against another field instead of a constant — needed whenever a
  condition compares two live values (e.g. current moisture against a
  per-species target read from Grow Profile) rather than a value against
  a fixed constant.

Each entry in `actions` has the same shape [Actuators](17-actuators.md)
writes: `capability` + `value`, where `value` is a real physical-unit
value, not a module's raw command range — executing an action is exactly
a write through Actuators, not a direct Modules write. This is what keeps
a Scenario hardware-independent: it names *what* it wants (a capability
and a physical value), never a specific module or its raw range.

## 11.5 No automatic precedence between generic and specialized

Deliberate design choice, not an oversight: the engine does not compare
`category` between a generic template and a specialized scenario to
decide which one "wins" for a given target. The only thing that stops a
generic template from running is its own enable-flag clause evaluating
false. This is deliberately not conflict-safe — see
[Philosophy](01-philosophy.md) and §11.3 above. `category` itself carries
no functional/exclusion role; it exists for UI grouping only ("show every
watering-related scenario for this Plant").

## 11.6 `kind: "script"`

Reserved for a future, more capable Head Device — **not implemented by
the open-source ESP32 firmware**, but part of the wire format now so
adding it later doesn't require a protocol revision. Complex logic run
instead on the Control Panel (which can be arbitrarily powerful — a local
hub, a cloud service) needs no protocol support at all: it already has
full REST access and can simply issue the same actions a Scenario would.
`kind: "script"` exists specifically for the case that Panel-side logic
can't cover — autonomous execution with no Control Panel connected at
all, per [Philosophy §1.3](01-philosophy.md#13-head-device-autonomy).

```json
{
  "id": "scn-custom-99",
  "kind": "script",
  "category": "watering",
  "target": { "type": "plant", "id": "plant-1234" },
  "status": "active",
  "language": "lua",
  "source": "-- arbitrary logic",
  "required_capabilities": ["valve", "moisture", "engine.script_lua"]
}
```

No `condition`/`actions` — the script is both. Gated entirely by
`engine.script_lua` (or a similarly-named capability — exact flag name
TBD with the rest of the [Capabilities](14-capabilities.md) catalog): a
Head Device that never declares it can never receive one, by the ordinary
add-time validation path in §12 — no special-casing needed for the fact
that this firmware doesn't implement scripting at all.

**Known, deliberately deferred cost:** defining the field is cheap;
safely *running* arbitrary script logic autonomously is not. Whatever
future firmware declares `engine.script_lua` is responsible for
sandboxing (execution step/time limits, a restricted binding surface —
e.g. only the same field-read/capability-action operations §11.2-§11.3
already expose, not raw memory/network access) so that a hung or
runaway script cannot violate
[Philosophy §1.3](01-philosophy.md#13-head-device-autonomy)'s "runs
reliably indefinitely" guarantee for every *other* Scenario sharing the
same device. Not specified further here — out of scope until a concrete
advanced-firmware implementation needs it.

## 11.7 Capability validation

See [Capabilities §14.4](14-capabilities.md#144-validation-lifecycle) for
the full lifecycle. Summary as it applies to Scenarios specifically:

- **`specialized`/`script`** — checked against the target's available
  Capabilities when added; rejected outright if insufficient (no partial
  acceptance — see [Capabilities §14.4](14-capabilities.md#144-validation-lifecycle)).
  Re-checked on every physical reconfiguration; a previously-valid
  instance that now fails is marked `status: "noop"` rather than deleted,
  and re-validated again at the next reconfiguration.
- **`generic`** — checked fresh against each matching target on every
  tick, since it's never "added" to a specific target in the first place.
  No persisted status: a target simply doesn't get evaluated against a
  generic template it currently lacks the capabilities for, and picks
  back up automatically the moment it does.
