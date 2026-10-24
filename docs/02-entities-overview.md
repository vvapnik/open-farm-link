# 2. Entities Overview

OFL organizes a garden installation into four entities.

| Entity | Bound to | Covers |
|---|---|---|
| [Garden](12-garden.md) | One Head Device | The whole garden block: its Plants + its System |
| [Plant](03-plant.md) | A single `plant_id` | A plant instance and every module grouped under it |
| [System](10-system.md) | Nothing plant-specific | Shared resources used by all Plants (e.g. water tank, mains pump) |
| [Scenarios](11-scenarios.md) | Neither — references Plants/System | Declarative automation rules that read and act on Plant/System state |

Garden, Plant, System, and Scenarios are all specified now (§3–15).

## 2.1 Garden

The top-level container: one Garden per Head Device. A Control Panel talks
to many Gardens, but this protocol mostly describes interacting with one
at a time — see [Garden §12.2](12-garden.md#122-one-garden-per-head-device-many-gardens-per-control-panel).
Garden is composed of Plants, a System, and
[Preferences](13-preferences.md) — garden-wide user constraints (quiet
hours, brightness cap, vacation mode) that bound what Scenarios are
allowed to do. Preferences' concrete fields aren't defined yet.

## 2.2 Plant

A Plant is the entity physically tied to a plant instance, identified by
a `plant_id`. It does not control a single module — it controls the group
of every module assigned to that `plant_id` (one anchor module plus
whatever non-anchor modules are grouped with it). See
[Plant](03-plant.md) for its full composition.

## 2.3 System

Modules that are not assigned to any single Plant — e.g. a shared water
tank or mains pump — are grouped here instead. System modules serve every
Plant in the Garden at once rather than one. It repeats Plant's
composition minus Grow Profile — see [System](10-system.md).

## 2.4 Scenarios

Declarative rules triggered by conditions (e.g. a time window plus a
Plant's Vitals reading) that perform actions crossing entity boundaries
(e.g. open a Plant's local valve, then run a System pump). Three kinds:
`generic` (one shared template per category, evaluated against every
matching target), `specialized` (bound to one target, e.g. a
weight-aware watering override), and `script` (reserved for a future,
more capable Head Device — not implemented by the open-source firmware).
See [Scenarios](11-scenarios.md).

## 2.5 Capabilities and Culture Pack

[Capabilities](14-capabilities.md) is a binary-flag analog of
[Modules](09-modules.md) — what a Plant/Garden can do, used to validate
Scenarios/Grow Profile/Culture Pack before accepting them.
[Culture Pack](15-culture-pack.md) bundles a Grow Profile with
specialized Scenarios for a single library import.

## 2.6 Actuators

[Actuators](17-actuators.md) is the write-side counterpart to
[Vitals](04-vitals.md): normalized current state of every controllable
capability on a Plant or System, addressed by the same Capability
vocabulary Scenarios and Capabilities already use — an alternative to
raw [Modules](09-modules.md) access for ordinary Control Panel UI, and
the interface Scenario actions execute through.

## 2.7 Standard dictionaries

Four reference catalogs tie the hardware and normalized layers together:
[Module Classes](18-module-classes.md) and
[Endpoint Types](19-endpoint-types.md) describe the raw hardware layer
(Modules); the [Capability Catalog](20-capability-catalog.md) is the
single shared vocabulary Vitals, Actuators, and Capabilities all draw
from; [System Fields](21-system-fields.md) standardizes the `status`/
`protocol_version` fields on Garden and Plant.
