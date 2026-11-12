# AGENTS.md

Guidance for AI agents working in this repo.

Project map: `ARCHITECTURE.takesix.json` — read it first. It shows the
spec's layers (Control Panel, local HTTPS binding, entity model,
dictionaries) and which `docs/*.md` file holds each part. When a doc is
added, renamed or split, update the map so its paths still match `docs/`.

This project is developed with the help of Claude Code (Anthropic).

## What this is

Open Farm Link (OFL) — a network protocol connecting a farm's Head Device
to a Control Panel (web/mobile). Start at [README.md](README.md) for the
doc index.

**OFL is backend-agnostic and intentionally independent of any specific
underlying hardware/bus protocol.** Earlier drafts of this spec were
written with a particular sibling project's I2C bus protocol in mind and
leaked its concepts (identifiers, register-level terms, its own
extensibility conventions) directly into OFL's entities. That coupling
was deliberately removed — OFL must describe a Head Device's network-facing
behavior in its own terms, working for any farm hardware, not just one
family of it. **Do not reintroduce dependencies on another bus protocol's
specific concepts** (register maps, bus addressing, its identifier
formats, etc.) into these docs. Where a Head Device's internal hardware
detail needs to surface at all, it belongs only in
[Modules](docs/09-modules.md) — see
[Philosophy §1.7](docs/01-philosophy.md#17-independence-from-any-specific-hardwarebus-backend).

## Status

Pre-1.0 draft, still being designed, no implementation yet. All entities,
the concrete REST API, the standard dictionaries and security (docs 1–22) are
specified — see `docs/`. Each doc
flags its own remaining open questions inline (e.g.
`docs/15-culture-pack.md` §15.4); nothing is currently tracked here that
isn't already in a doc.

## Conventions

- Docs live in `docs/`, numbered, one focused topic each.
- Entities reference each other via `{ "$ref": "NN-name.md" }` rather than
  inlining one structure inside another's example — keep the composition
  visible.
- JSON is used for all schema examples.
- No ADRs, no `CHANGELOG.md` while pre-1.0 — flag open design questions
  inline in the relevant doc's prose instead of a separate decision
  record.
- Spec docs (`docs/*.md`, `README.md`) describe **only the protocol
  itself** — no design-process narrative (who proposed what, what an
  earlier draft assumed, session dates, open questions phrased as "not
  yet confirmed by the user"). State the current design directly, the way
  a finished spec would. Process history and rationale that's useful for
  future design work but doesn't belong in the spec text goes in this
  file instead — see "Naming notes" below for an example.
- This is currently a design-only repo, read only by AI agents for
  prototyping — formatting/polish is explicitly deferred until the design
  is validated against real hardware. Don't spend effort reformatting
  during this phase unless asked.

## TODO

- **Firmware updates.** EN 18031 (RED cybersecurity) expects a Head
  Device to receive security updates, and only authentic ones: signed
  firmware images, Secure Boot, OTA that verifies the signature before
  applying. Deliberately not part of the OFL wire protocol for now — decide
  whether OFL gets an update endpoint (authenticated like everything else,
  per `docs/22-security.md`) or updates stay an implementation concern
  documented outside the spec.

- **Cloud binding compliance.** Once a cloud binding exists (see
  "Product context" below): how a Head Device authenticates to the cloud,
  how it gets bound to a user account during pairing, and what EN 18031 /
  Cyber Resilience Act require for the cloud path specifically. Nothing
  about this is specified yet.

## Product context

Background the spec doesn't state but design decisions should be checked
against. OFL is meant to serve four ways of operating a Head Device:

1. **Mobile app, local** — the official app talks OFL directly to the
   Head Device; works only at home (same LAN).
2. **Proprietary cloud** — the main consumer path. Remote access through
   a cloud service.
3. **Home Assistant** — for enthusiasts; an OFL integration acting as the
   Control Panel. Also used now as the test bench for real hardware.
4. **Local proprietary hub** — for users who want to stay off the cloud
   without running HA. Its main jobs: reverse proxy in front of the Head
   Devices (one place clients connect to), recording telemetry including
   e.g. camera timelapses (the camera stream itself is out-of-band, see
   `docs/19-endpoint-types.md` §19.3), and storing the profile libraries
   (Grow Profiles, Culture Packs, Hardware Calibration). A web UI for
   management on top is welcome but secondary.

Implications for the spec:

- **Next protocol version must add a cloud connection.** v0.1 defines only
  the local HTTPS binding (`docs/01-philosophy.md` §1.2, now worded to
  allow other bindings). A cloud binding will almost certainly be
  device-initiated (Head Device dials out, e.g. MQTT/WebSocket) and should
  reuse the entity model as-is — keep entities transport-agnostic, and
  keep `docs/16-rest-api.md` readable as "the local binding", not as the
  definition of the entities themselves.

### Open question — concurrent access modes (user to decide)

Can several of the four modes be active on one Head Device at the same
time (e.g. cloud + HA, or the local app + HA polling), or is a device in
one mode at a time? To be decided after going through the entities and
use cases again. Inputs to weigh:

- The local binding serves **one connection at a time** (§1.2, §16.3,
  §22.7); multiple local clients contend for it, and periodic polling
  (HA) occupies it often. A bulk read (`/garden/snapshot`-style) endpoint
  would shorten that occupancy considerably.
- A cloud connection plus a local TLS session at once is two TLS
  sessions — tight on an ESP32-C3 without PSRAM (§1.6).
- The hub as reverse proxy would naturally be the single local client
  of each Head Device and fan out to everyone else — which solves
  contention for mode 4, but not for modes 1/3 used side by side.

## Security design notes

Rationale behind `docs/22-security.md`, kept here so the spec stays
declarative:

- Two credential types (pairing token + login/password over HTTP Basic)
  exist because the protocol is meant to be equally usable by an official
  app and by enthusiasts from curl/scripts/Home Assistant. Both are
  first-class; neither requires the other.
- No unauthenticated mode at all, not even read-only, and no switch to
  disable auth: under EN 18031's harmonisation restrictions a product
  where the user can opt out of authentication loses presumption of
  conformity (self-declaration no longer enough). Keep it that way.
- No roles/permissions in v0.1 — every credential is full access. Revisit
  only with a concrete use case (e.g. read-only dashboard credential).
- Pinned self-signed key instead of a CA: no internet dependency, fits
  zero-config. `garden_id` derives from the key so identity and trust
  anchor can't drift apart; full `fingerprint` uses curl's
  `--pinnedpubkey` format on purpose.
- Two pairing paths. The button is mandatory on every device because a
  printed code gets lost and must never be the only way back in (home
  use). The setup code is optional and exists for fleets: an installer
  scans a waybill and pairs dozens of devices from one terminal instead
  of running to each one within its button window. A setup code only
  works on an unclaimed device, so a leaked code is not a permanent
  backdoor.
- The challenge step (`/garden/pair/challenge`) is there so an official
  app never hands the setup code to an impersonator, and doubles as the
  code-to-device matcher for fleets. 80-bit setup code is required
  because the HMAC proof would otherwise allow offline brute force (a
  short numeric PIN would need a PAKE such as SPAKE2+ instead).
- Button pairing is trust-on-first-use by design. If that ever needs
  hardening without a sticker, the option is a code shown on a device
  display (needs display hardware), not a printed one.
- Factory reset regenerates the key, so `garden_id` changes — a reset
  device is intentionally a new Garden for every Control Panel.

## Naming notes

Background for naming choices that isn't needed to *use* the spec, but is
useful if a name ever needs revisiting:

- **Culture Pack** (not "Advanced Grow Profile"): a name implying "a
  fancier Grow Profile" would undersell that it also carries Scenarios —
  it's a combination of two different entities, not a variant of one.
- **`blueprint_id`**: a hardware-model library identifier, entirely OFL's
  own concept — not borrowed from, or required to match, any identifier
  scheme a Head Device's internal bus protocol might separately use for
  the same physical module. It lives **only** on
  [Modules](docs/09-modules.md) (a module's own field) — an earlier draft
  mistakenly also put a `blueprint_id` field directly on
  [Hardware Calibration](docs/06-hardware-calibration.md) and
  [Settings](docs/07-settings.md) themselves, as if each carried its own
  identity, instead of referencing a module's `blueprint_id` through
  Modules. Fixed 2026-09-07: both are now lists (Hardware Calibration
  keyed by `blueprint_id`+`endpoint_type`, model-level; Settings keyed by
  `module_id`, instance-level, since two modules of the same model can be
  mounted differently). Don't reintroduce a top-level `blueprint_id` field
  on either.

## Decisions settled but not yet written into docs

Record things here as soon as they're settled in conversation, even if the
exact field names/palette aren't ready yet — so a later session detailing
the actual schema doesn't have to reconstruct the reasoning from scratch.
Move an item into the relevant `docs/*.md` (and delete it from here) once
it's been written up there.

Next open item to pick up:
[Culture Pack §15.4](docs/15-culture-pack.md#154-open-question) — refresh
semantics for an already-imported pack's bundled Scenario(s) when its
library entry updates.
