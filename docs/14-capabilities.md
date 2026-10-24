# 14. Capabilities

Capabilities is a high-level analog of [Modules](09-modules.md): where
Modules is the raw per-endpoint listing, Capabilities is a set of binary
presence flags answering "what can this Plant/Garden do" — cheap to check
(a subset test, not a scan), used to validate whether a
[Scenario](11-scenarios.md), [Grow Profile](08-grow-profile.md), or
[Culture Pack](15-culture-pack.md) can run on this particular system
before accepting it.

## 14.1 Two sources, two scopes

Capabilities come from two independent sources:

- **Module-derived** — formed dynamically from the Head Device's own
  module registry. Scoped per **Plant** (that Plant's own module group)
  and per **System** (System's module group; there is exactly one System
  per Garden, so no separate storage per System sub-block is needed).
- **Engine-declared** — firmware/engine feature flags the Head Device
  itself declares at startup, *not* derived from any module (e.g.
  `engine.impulse_watering`, `engine.custom_endpoints`,
  `engine.script_lua`). This is what actually distinguishes a basic
  firmware tier from a more capable one when both happen to have
  identical hardware attached — hardware-presence flags alone can't
  express "this firmware can't execute multi-stage dosing" if the valve
  module is physically there either way.

Engine-declared flags are global to the one Head Device running
everything, so they are **unioned into both** the Plant-level and the
Garden-level available-capability set at validation time:

- **Plant's available capabilities** = that Plant's own module-derived
  flags ∪ engine-declared flags.
- **Garden's available capabilities** = System's module-derived flags ∪
  engine-declared flags. (**Not** a union across every Plant too — a
  Garden-scoped check only ever sees System + engine, never reaches into
  individual Plants' hardware.)

```json
{
  "modules": ["valve", "moisture"],
  "engine": ["engine.impulse_watering", "engine.custom_endpoints"]
}
```

## 14.2 Standard range + open vendor range

Standard-range, module-derived flags are drawn from the
**[Capability Catalog](20-capability-catalog.md)** — the same key a
module-derived flag uses here is the identical key
[Vitals](04-vitals.md) reports readings under and
[Actuators](17-actuators.md) accepts writes on. This is deliberate, not
incidental: it's what lets `required_capabilities` (a flag name) and a
Scenario's `condition`/`actions` (which reference the same names as
fields/capabilities) stay in lockstep — there's exactly one vocabulary
for "this thing exists" and "read/write this thing," not two that happen
to look similar. `engine.*` flags are a separate namespace, not part of
the Capability Catalog — see
[Capability Catalog §20.4](20-capability-catalog.md#204-engine-is-a-separate-namespace).

Beyond the standard catalog, the dictionary is extensible **without
central control**: a third-party vendor can mint a new flag for its own
custom module or firmware feature; the protocol does not validate or
interpret it — it just passes through, as long as producer
(module/firmware) and consumer (a Scenario's `required_capabilities`)
agree on the same value. That agreement happens through the same
hardware-model library that already publishes custom endpoint semantics —
see [Hardware Calibration](06-hardware-calibration.md) — a vendor's
library entry is the natural place to also declare what its custom
Capability flags mean, rather than a separate out-of-band registry.

## 14.3 Representation

An open-ended vendor range rules out a fixed-width packed bitmask (it caps
how many flags can ever exist). The wire format is an **array of flag
identifier strings**. Firmware is free to intern this into an actual
bitmask/bitset internally for cheap validation — the protocol only
constrains the wire representation, the same posture
[Vitals History](05-vitals-history.md) takes toward on-device storage.

## 14.4 Validation lifecycle

Enforced by the **Head Device**, not the Control Panel — consistent with
[Philosophy §1.3](01-philosophy.md#13-head-device-autonomy): the device
does not trust that the Panel already validated, since the Panel's view
of Capabilities could be stale.

- **Add-time** (a Scenario, Grow Profile, or Culture Pack is submitted):
  checked against the target's available capabilities (§14.1). **Rejected
  outright** if any `required_capabilities` entry is missing — no partial
  acceptance, no automatic simplification. Adapting a Culture Pack to fit
  a system lacking some capability (e.g. dropping a weight-based clause)
  is a **Control Panel-side** concern, done before submission — the Head
  Device only ever sees a request it either fully satisfies or rejects.
- **Revalidation** (triggered by a physical reconfiguration — a module
  removed or replaced): every currently-installed `specialized`/`script`
  Scenario, Grow Profile, and Culture Pack is re-checked. One that now
  fails is marked `status: "noop"` (stays installed, simply doesn't run)
  rather than deleted, and is re-checked again at the next
  reconfiguration — it resumes automatically if the missing capability
  reappears (e.g. the module is reconnected or replaced with an
  equivalent one).
- **Generic scenarios** don't go through add-time validation at all
  (nothing is "added" per target) — see
  [Scenarios §11.7](11-scenarios.md#117-capability-validation).

## 14.5 `required_capabilities` appears on three entities

[Grow Profile](08-grow-profile.md), [Scenario](11-scenarios.md), and
[Culture Pack](15-culture-pack.md) each carry their own
`required_capabilities` field independently — each of the three can be
submitted to a Head Device on its own, not only bundled together, so each
needs standalone validation rather than inheriting it from a container.
