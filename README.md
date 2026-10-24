# Open Farm Link (OFL) Protocol Specification

**Version:** 0.1 (WIP)
**Status:** Draft — actively evolving
**Architecture:** Control Panel (client) ↔ Head Device (server), HTTPS REST, authenticated, local network
**Target use case:** connecting a farm's head device to a control center (web/mobile app)
**License:** Open Source

Open Farm Link is a network protocol connecting a farm's **Head Device**
to a **Control Panel** (a web or mobile app used to monitor and configure
it). It is backend-agnostic: a Head Device may be built on any underlying
hardware/bus technology internally — OFL only specifies the network-facing
protocol between it and the Control Panel, not how the Head Device talks
to its own sensors and actuators.

## Reading the spec

The specification is split into focused documents so each part can evolve
and be reviewed independently. Entities are documented separately from the
things that reference them — e.g. [Plant](docs/03-plant.md) only links out
to the structures it's composed of, it doesn't inline them.

| # | Document | Covers |
|---|----------|--------|
| 1 | [Philosophy](docs/01-philosophy.md) | Logical vs. transport direction, device autonomy, zero-config, resource constraints |
| 2 | [Entities Overview](docs/02-entities-overview.md) | Top-level entity map: Garden, Plant, System, Scenarios |
| 3 | [Plant](docs/03-plant.md) | The per-plant entity: identity, composition |
| 4 | [Vitals](docs/04-vitals.md) | Current sensor readings for a Plant |
| 5 | [Vitals History](docs/05-vitals-history.md) | Time-series format for Vitals |
| 6 | [Hardware Calibration](docs/06-hardware-calibration.md) | Per-hardware-model normalization, library-sourced |
| 7 | [Settings](docs/07-settings.md) | Per-installation physical parameters, user-owned |
| 8 | [Grow Profile](docs/08-grow-profile.md) | Per-plant-species targets, library-sourced |
| 9 | [Modules](docs/09-modules.md) | Raw module/endpoint listing for UI + direct access |
| 10 | [System](docs/10-system.md) | Shared, non-plant-bound modules (e.g. water tank) |
| 11 | [Scenarios](docs/11-scenarios.md) | Declarative automation rules over Plant/System |
| 12 | [Garden](docs/12-garden.md) | Top-level entity: one garden block per Head Device |
| 13 | [Preferences](docs/13-preferences.md) | Garden-wide constraints/flags (quiet hours, brightness cap, vacation mode) |
| 14 | [Capabilities](docs/14-capabilities.md) | Binary-flag analog of Modules; gates Scenarios/Grow Profile/Culture Pack |
| 15 | [Culture Pack](docs/15-culture-pack.md) | Bundles Grow Profile + specialized Scenarios for library import |
| 16 | [REST API](docs/16-rest-api.md) | Concrete HTTP endpoints, error format, discovery |
| 17 | [Actuators](docs/17-actuators.md) | Normalized, capability-keyed current state of every controllable endpoint |
| 18 | [Module Classes](docs/18-module-classes.md) | Standard `class` catalog for Modules |
| 19 | [Endpoint Types](docs/19-endpoint-types.md) | Standard raw endpoint `type`/`format` catalog for Modules |
| 20 | [Capability Catalog](docs/20-capability-catalog.md) | Shared vocabulary for Vitals/Actuators/Capabilities keys |
| 21 | [System Fields](docs/21-system-fields.md) | Standard `status`/`protocol_version` fields on Garden and Plant |
| 22 | [Security](docs/22-security.md) | Device identity, HTTPS, pairing, token and login/password auth, factory reset |

## Contributing

The protocol is pre-1.0 and WIP — no ADRs or changelog discipline yet.
Open design questions are flagged inline in the relevant doc rather than
tracked separately. Issues and PRs should target the specific `docs/*.md`
file(s) they affect.
