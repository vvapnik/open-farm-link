# 1. Philosophy

Open Farm Link (OFL) is the network protocol connecting a garden **Head
Device** to a **Control Panel**. This section fixes the principles that
every later entity/format decision should be checked against.

## 1.1 Logical direction: pure PUSH from Control Panel to Head Device

The Control Panel is the sole originator of intent. It pushes
configuration (Scenarios, Grow Profile, Settings) and, optionally,
time-bounded manual commands down to the Head Device. The Head Device
never decides *what* should happen on the Control Panel's behalf — it only
executes what it's told, plus whatever autonomous behavior is described in
§1.3.

This is a statement about **logical** direction only. It does not by
itself say which side opens the network connection — see §1.2.

## 1.2 Transport direction is independent of logical direction

This version of the specification defines one transport binding: the
Head Device runs an HTTPS server on the local network; the Control Panel is
the HTTP client. Within this binding the Head Device does not initiate
connections to a Control Panel — logical PUSH and transport-level PUSH
coincide, and the Control Panel both decides *and* dials.

This binding reaches a Head Device only from its own local network: it
provides no NAT traversal, remote access, or relay. Access from outside
the LAN requires a different transport binding, typically one where the
Head Device itself connects out to a remote service. Such bindings are
not defined in this version; the entity model (documents 2–15, 17–21) is
independent of the binding and is meant to carry over to them unchanged.

OFL is a stateless REST protocol: every request is self-contained, and
the Head Device carries no cross-request session state. There is no
live/streaming channel of any kind (no SSE, no WebSocket) — a Control
Panel that wants fresh data polls for it. Because there is no session to
preserve, the Head Device is free to serialize request handling
completely: it processes **one request at a time**, on a single
connection, and simply doesn't accept a second one until the first
completes. This removes an entire class of concern (concurrent-access
races, partial multi-request transactions) from the resource-constrained
target hardware for free — there is nothing stateful that a rejected or
queued request could interrupt.

## 1.3 Head Device autonomy

All baseline control scenarios — watering, lighting, climate control —
MUST run entirely on the Head Device, without any connection to the
Control Panel. The Control Panel's role is:

- **Configuration** — writing Scenarios, Grow Profile, and Settings that
  the Head Device then executes on its own.
- **Monitoring** — reading Vitals and Vitals History.
- **Direct intervention** — raw reads/writes through
  [Modules](09-modules.md), entirely at the user's own risk (§9.3) — there
  is no managed, time-bounded override mechanism in this version of the
  protocol. A module set via Modules stays at that value until written
  again; it is not tracked or reverted by any Scenario. A future version
  of the protocol may add a managed override with a bounded lifetime; not
  present in v0.1.

The Control Panel is never a required participant in a real-time control
loop. A Head Device with no Control Panel reachable at all must keep
running its configured Scenarios correctly and indefinitely.

## 1.4 Commands express desired state, not one-off actions

Where a Control Panel does need to affect the Head Device directly (as
opposed to just configuring it), commands should express **desired
state**, not an imperative action — so that repeated or out-of-order
delivery is safe. This governs configuration writes (Scenarios, Grow
Profile, Settings); it does not extend to Modules, which is deliberately
raw and outside this guarantee — see [Modules §9.3](09-modules.md#93-readwrite-semantics).

## 1.5 Zero-configuration ecosystem

The ecosystem is designed to require minimal user intervention by default:
buy a kit, physically stack the rack, plant from the kit, scan its
`AgroBlueprint`, and the system is ready to run with sensible defaults
imported from a library. Manual configuration through the UI is the
exception path for a non-standard build (e.g. added spacers changing a
physical dimension), not part of the default flow. This principle governs
the import/override split seen in [Hardware Calibration](06-hardware-calibration.md)
and [Settings](07-settings.md).

## 1.6 Resource constraints

The open-source reference Head Device firmware must run fully on a plain
ESP32, and where possible on ESP32-C3 (single-core, typically no PSRAM).
This pushes protocol-level formats toward fixed/paged shapes rather than
open-ended schemas, and implementations toward a small number of
concurrent connections and fixed-size arrays rather than unbounded
dynamic allocation.

## 1.7 Independence from any specific hardware/bus backend

OFL is backend-agnostic by design: it says nothing about how a Head
Device talks to its own sensors and actuators internally, and its
entities (Garden, Plant, System) are defined independently of any
particular physical bus or wiring topology. A Head Device may implement
OFL on top of any internal hardware architecture it likes — OFL only
specifies what that device exposes over the network, not how it gets
there internally.

The one place implementation-specific detail is deliberately allowed to
surface is [Modules](09-modules.md) — a raw listing of whatever physical
devices and endpoints the Head Device's own internal registry currently
holds, used for rendering the physical system in the UI and for direct
low-level access. Every other entity (Plant, System, Scenarios,
Capabilities) is defined purely in OFL's own terms, with no assumption
about positional ordering, addressing scheme, or bus technology
underneath.

## 1.8 No unauthenticated access

Being on the same local network grants nothing. Every request carries
credentials, every connection is TLS, and there is no mode that turns
either off — see [Security](22-security.md). Two credential types exist
side by side so the protocol stays equally usable by an official app
(pairing tokens) and by hand from scripts or curl (login/password over
standard HTTP Basic auth); neither is second-class.
