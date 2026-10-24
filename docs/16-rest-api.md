# 16. REST API

The concrete HTTP surface a Head Device exposes to a Control Panel, per
[Philosophy §1.2](01-philosophy.md#12-transport-direction-is-independent-of-logical-direction):
stateless REST over HTTPS, JSON bodies, one request in flight at a time.
Every request is authenticated — see [Security](22-security.md).

## 16.1 Conventions

- HTTPS only, port 443; no plain-HTTP listener ([Security §22.2](22-security.md#222-device-identity-and-transport)).
- Every request except `POST /garden/pair/challenge` and `POST /garden/pair`
  carries `Authorization: Bearer <token>` or `Authorization: Basic …`
  ([Security §22.5](22-security.md#225-authenticating-requests)).
- All request/response bodies are JSON; `Content-Type: application/json`.
- Paths are rooted at `/garden` — a Head Device serves exactly one Garden
  (§12.2).
- Every resource listing (`GET` on a collection) returns a summary shape,
  not the full nested entity — a client fetches a specific item's
  sub-resources individually. This keeps any single response small and
  bounded, per [Philosophy §1.6](01-philosophy.md#16-resource-constraints).
- A Plant is never created or deleted through this API — it comes into
  existence when the Head Device's own hardware layer detects a new
  anchor module, and disappears when that module is removed. The API can
  only rename (`PUT .../plants/{id}` with `name`) or configure an
  existing Plant, consistent with the zero-configuration principle
  ([Philosophy §1.5](01-philosophy.md#15-zero-configuration-ecosystem)).
- Direct hardware control goes exclusively through `Modules` — raw,
  unmanaged, at the caller's own risk (§9.3, §1.3). There is no separate
  managed-override mechanism in this version of the protocol.

## 16.2 Error format

```json
{ "error": "missing_capabilities", "missing": ["weight"] }
```

| Status | `error` | When |
|---|---|---|
| 400 | `bad_request` | Malformed JSON or a field fails basic type/shape validation. |
| 401 | `unauthorized` | Missing or invalid credentials. Response carries `WWW-Authenticate: Bearer realm="ofl", Basic realm="ofl"`. |
| 403 | `pairing_closed` | Pairing endpoint called while neither pairing path is open — see [Security §22.4](22-security.md#224-pairing). |
| 403 | `invalid_setup_code` | Wrong setup code in a pairing request. |
| 404 | `not_found` | Path references a `plant_id`, module `id`, endpoint index, or Scenario `id` that doesn't exist. |
| 405 | `not_writable` | `PUT`/`DELETE` attempted on a `generic` Scenario (read-only — see [Scenarios §11.1](11-scenarios.md#111-three-kinds)). |
| 409 | `username_taken` | Creating a login/password credential with a `username` already in use. |
| 409 | `no_free_slots` | Creating a credential while credential storage is full — see [Security §22.6](22-security.md#226-credential-management). |
| 422 | `missing_capabilities` | A Scenario, Grow Profile, or Culture Pack was submitted requiring a capability the target doesn't have — see [Capabilities §14.4](14-capabilities.md#144-validation-lifecycle). Body includes `missing`, the specific flags that failed. |
| 429 | `too_many_attempts` | Authentication throttled after repeated failures — see [Security §22.5](22-security.md#225-authenticating-requests). |

## 16.3 Concurrency

The Head Device accepts exactly one connection at a time
([Philosophy §1.2](01-philosophy.md#12-transport-direction-is-independent-of-logical-direction)).
A second connection attempt while one is in progress is refused at the
TCP level — not answered with an HTTP status. A client must retry
establishing the connection itself, not just retry the request over an
existing one.

A connection with no complete request within 5 seconds is closed by the
Head Device, so an idle or hostile client can't hold the single slot —
see [Security §22.7](22-security.md#227-connection-handling).

## 16.4 Discovery

A Head Device advertises itself on the local network via mDNS
(`_ofl._tcp.local.`, port 443); its TXT record carries `garden_id`, plus
`pair=button` / `pair=code` while the device accepts pairing
([Security §22.4](22-security.md#224-pairing)). A Control Panel
resolves this to an address before making any request below. TXT data is
unauthenticated: a Control Panel identifies an already-paired device by
its pinned `fingerprint`, not by `garden_id` alone.

### Hostname: `ofl-peer{N}.local`

A Head Device's mDNS hostname is `ofl-peer{N}.local`, where `N` is a
positive integer. On startup (and on any hostname conflict thereafter —
e.g. two devices racing on boot after a power restoration), a Head Device
probes `ofl-peer1.local`, `ofl-peer2.local`, ... in order and claims the
first one nothing else on the network answers for. `N` is **not** a
stable identity — a device may claim a different `N` after a reboot if a
lower one has freed up or is newly contested; `garden_id` (in the TXT
record) is what identifies the Garden itself, not the hostname. This
naming scheme is deliberately simple (sequential, not random-suffixed
like generic Bonjour conflict resolution) so a small home installation's
device list stays predictable and low-numbered.

## 16.5 Endpoints

### Garden

| Method | Path | Notes |
|---|---|---|
| `GET` | `/garden` | `{ garden_id, name, description, protocol_version, status, type, plant_ids }` |
| `PUT` | `/garden` | Body: `{ name?, description? }` — either field alone is a valid partial update |
| `GET` | `/garden/preferences` | See [Preferences](13-preferences.md) |
| `PUT` | `/garden/preferences` | |
| `GET` | `/garden/capabilities` | Engine-declared flags — see [Capabilities §14.1](14-capabilities.md#141-two-sources-two-scopes) |
| `GET` `PUT` | `/garden/time` | See [§16.6](#166-time) |

### Pairing and credentials

See [Security](22-security.md) for bodies and rules.

| Method | Path | Notes |
|---|---|---|
| `POST` | `/garden/pair/challenge` | No auth; setup-code path only. Device proves it knows the setup code. |
| `POST` | `/garden/pair` | No auth; button window or setup-code path. Creates a token or login/password credential. |
| `GET` `POST` | `/garden/clients` | List credentials / add one without the pairing window |
| `PUT` `DELETE` | `/garden/clients/{client_id}` | Rename, change password, revoke |

### Plants

| Method | Path | Notes |
|---|---|---|
| `GET` | `/garden/plants` | `[{ plant_id, name, status }]` |
| `GET` | `/garden/plants/{id}` | |
| `PUT` | `/garden/plants/{id}` | Body: `{ name?, description? }` — either field alone is a valid partial update |
| `GET` | `/garden/plants/{id}/vitals` | See [Vitals](04-vitals.md) |
| `GET` | `/garden/plants/{id}/vitals/history` | `?since=<ISO8601>&limit=<n>` — server caps `limit` regardless of what's requested |
| `GET` | `/garden/plants/{id}/hardware-calibration` | Read-only, returns the list — see [Hardware Calibration §6.4](06-hardware-calibration.md#64-source-and-refresh-library-owned-not-user-owned) |
| `GET` | `/garden/plants/{id}/settings` | Returns the list — see [Settings](07-settings.md) |
| `GET` `PUT` | `/garden/plants/{id}/settings/{module_id}` | Body: `{ settings }` — one module's Settings entry |
| `GET` `PUT` | `/garden/plants/{id}/grow-profile` | `PUT` validated against Capabilities (422) |
| `GET` | `/garden/plants/{id}/modules` | See [Modules](09-modules.md) |
| `GET` `PUT` | `/garden/plants/{id}/modules/{module_id}/endpoints/{index}` | Raw passthrough, no validation |
| `GET` | `/garden/plants/{id}/capabilities` | |
| `GET` | `/garden/plants/{id}/actuators` | See [Actuators](17-actuators.md) |
| `GET` `PUT` | `/garden/plants/{id}/actuators/{capability}` | Body: `{ value }` |
| `POST` | `/garden/plants/{id}/culture-packs` | Import — see [Culture Pack](15-culture-pack.md). Validated (422); applies Grow Profile + `disables` + Scenarios atomically |
| `GET` `POST` | `/garden/plants/{id}/scenarios` | `POST` for `specialized`/`script` only; validated (422) |
| `GET` `PUT` `DELETE` | `/garden/plants/{id}/scenarios/{scenario_id}` | `PUT`/`DELETE` return 405 for a `generic` scenario |

### System

Same sub-resource shape as Plants, minus `grow-profile` and
`culture-packs` — see [System](10-system.md).

| Method | Path |
|---|---|
| `GET` | `/garden/system` |
| `GET` | `/garden/system/vitals` |
| `GET` | `/garden/system/vitals/history` |
| `GET` | `/garden/system/hardware-calibration` |
| `GET` | `/garden/system/settings` |
| `GET` `PUT` | `/garden/system/settings/{module_id}` |
| `GET` | `/garden/system/modules` |
| `GET` `PUT` | `/garden/system/modules/{module_id}/endpoints/{index}` |
| `GET` | `/garden/system/capabilities` |
| `GET` | `/garden/system/actuators` |
| `GET` `PUT` | `/garden/system/actuators/{capability}` |
| `GET` `POST` | `/garden/system/scenarios` |
| `GET` `PUT` `DELETE` | `/garden/system/scenarios/{scenario_id}` |

### Scenarios (Garden-wide view)

| Method | Path | Notes |
|---|---|---|
| `GET` | `/garden/scenarios` | Every Scenario known to the Garden, all `kind`s, all targets — see [Scenarios](11-scenarios.md) |
| `GET` `PUT` `DELETE` | `/garden/scenarios/{id}` | Same record as under a Plant/System's own `scenarios` sub-resource — one Scenario, one canonical path regardless of where it's reached from. `PUT`/`DELETE` on `generic` → 405 |

`/garden/plants/{id}/scenarios` and `/garden/system/scenarios` are
filtered views of this same collection, scoped by `target`.

## 16.6 Time

The Head Device's own clock — what the `time` pseudo-field
([Scenarios §11.4](11-scenarios.md#114-condition-format)) reads at
evaluation time, so quiet-hours/time-window Scenarios only work once this
clock is set.

```json
{ "time": "2026-09-07T18:30:00Z", "source": "ntp" }
```

| Field | Type | Description |
|---|---|---|
| `time` | `string` | Current device time, ISO 8601, UTC. |
| `source` | `enum: ntp, manual, none` | How the Head Device currently believes it knows the time. `none` means Scenario time-window conditions cannot be evaluated correctly yet. |

`GET` reads the above; `PUT` takes `{ "time": "<ISO 8601 UTC>" }` and sets
the device clock directly — the Head Device applies it immediately and
reports `source: "manual"` from then on.

A Head Device with internet access is expected to sync via NTP on its own
(consistent with [Philosophy §1.5](01-philosophy.md#15-zero-configuration-ecosystem)
— no Control Panel involvement needed in the common case) and report
`source: "ntp"` once that succeeds. `PUT` exists for the case NTP can't
reach it at all — an isolated LAN with no internet uplink — where a
Control Panel is the only available time source. A Control Panel MAY
proactively `PUT` its own clock whenever it observes `source: "none"`, as
a zero-config convenience; this is a Control Panel-side policy choice, not
a protocol requirement. `PUT` always wins over a pending NTP sync for the
device's current boot session; a reboot with no internet uplink returns to
`source: "none"` until re-set, since this version of the protocol doesn't
require battery-backed RTC hardware.
