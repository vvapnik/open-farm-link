# 22. Security

How a Head Device proves its identity, how a Control Panel gets and
presents credentials, and how a device returns to a blank state. Every
rule here applies to every endpoint in [REST API](16-rest-api.md).

## 22.1 Principles

- **No unauthenticated access.** Every request except the two pairing
  endpoints (§22.4) MUST carry valid credentials (§22.5); anything else is
  answered `401 unauthorized`. A Head Device MUST NOT offer any setting,
  build option, or mode that disables authentication, including for
  read-only requests.
- **No roles in this version.** Every valid credential has full access to
  the whole Garden, including managing other credentials (§22.6).
- **Stateless, per [Philosophy §1.2](01-philosophy.md#12-transport-direction-is-independent-of-logical-direction).**
  Credentials travel in every request in the standard HTTP `Authorization`
  header. There is no login request and no server-side session.
- **No shared defaults.** A Head Device ships with no credentials at
  all; nothing secret is identical across devices.
- **Physical presence is always enough.** Every Head Device can be paired
  with its button alone (§22.4), so losing a printed code never locks a
  user out. A setup code (§22.3) is an optional addition for pairing many
  devices remotely.

## 22.2 Device identity and transport

On first boot, and again after every factory reset (§22.8), a Head Device
generates an **ECDSA P-256 key pair** and a **self-signed certificate**
for it. The private key never leaves the device.

All OFL traffic is **HTTPS (TLS 1.2 or newer) on port 443**. A Head Device
MUST NOT serve the OFL API over plain HTTP.

There is no certificate authority. A client authenticates the device by
**pinning** its public key instead:

| Value | Definition | Example use |
|---|---|---|
| `fingerprint` | `sha256//` + base64 of SHA-256 over the certificate's DER-encoded SubjectPublicKeyInfo. | Stored by a Control Panel at pairing; checked on every later connection. Same format as curl's `--pinnedpubkey`. |
| `garden_id` | Lowercase base32 (RFC 4648 alphabet, no padding) of the first 10 bytes of that same SHA-256 — 16 characters. | Human-readable identifier of the Garden ([Garden §12.1](12-garden.md#121-composition)); published in mDNS TXT. |

`garden_id` is for finding and naming a device; `fingerprint` is the
security anchor. A Control Panel MUST compare the full `fingerprint`, not
just `garden_id`, before sending credentials to a device it has paired
with. A mismatch means either a different device or a factory-reset one
(§22.8) — the Control Panel treats it as unpaired and does not send stored
credentials.

## 22.3 Setup code (optional)

A Head Device MAY have a **setup code**: 80 random bits, unique per
device, written as 16 characters of Crockford base32 in four groups of
four (`7K2M-Q9XD-4HTR-8WCN`). It is provisioned at manufacture and
delivered on the device, its packaging, or a delivery document (e.g. a
waybill listing the codes of every device in a shipment, as text or QR).
A DIY build MAY generate one on first boot and show it through a
local-only channel such as a serial console or a display.

When comparing, a Head Device normalizes input: drops dashes and spaces,
upper-cases, maps `O`→`0` and `I`/`L`→`1`.

The setup code is never changed by a factory reset and is never returned
by any endpoint. A device without a setup code is paired by button only.

## 22.4 Pairing

Pairing turns proof of either physical presence or knowledge of the
setup code into a stored credential. A device is **unclaimed** while it
holds no credentials (first boot, after a factory reset, or after the
last credential was deleted).

| Path | Allowed when | Proof | Device identity check |
|---|---|---|---|
| **Button** | Pairing window open (below) — claimed or unclaimed | Someone pressed the button | Trust on first use |
| **Setup code** | Device is unclaimed and has a setup code | Correct `setup_code` in the request | Verified via challenge (step 1) |

### Pairing button and window

A Head Device MUST have a physical pairing control (on a dev-board-based
build, the board's BOOT button qualifies). A **short press** opens the
**pairing window** for **2 minutes**; it closes on the first successful
pairing or when the time runs out. A Head Device with an indicator light
SHOULD show while the window is open.

A claimed device accepts new pairings **only** through the button window
(or through an already-authenticated client, §22.6) — a setup code alone
can never add a credential to a claimed device.

Outside both paths, the pairing endpoints answer `403 pairing_closed`.
The mDNS TXT record ([REST API §16.4](16-rest-api.md#164-discovery))
carries `pair=button` while the window is open, or `pair=code` while the
device is unclaimed and has a setup code, so a Control Panel can list
devices ready to pair.

### Step 1 (setup code path): verify the device — `POST /garden/pair/challenge`

Lets a client confirm, before revealing the setup code, that the TLS peer
it is talking to really knows that setup code — i.e. that no one on the
network is impersonating the device. Available whenever the setup-code
path is (`403 pairing_closed` otherwise).

```json
{ "nonce": "q2C9tY0bVt8r1w1d7Kq3xA==" }
```

`nonce` is at least 16 random bytes, base64. The response:

```json
{ "proof": "2b7r…base64…", "fingerprint": "sha256//Yx8m…=", "garden_id": "mfrggzdfmztwq2lk" }
```

`proof` = base64 HMAC-SHA256, keyed with the normalized setup code (ASCII),
over `nonce bytes ‖ SHA-256(SPKI)` of the device's own certificate.

The client recomputes `proof` using the setup code it has and the
fingerprint of the certificate **it actually saw in the TLS handshake**;
on mismatch it aborts without sending step 2. An official Control Panel
MUST perform this step on the setup-code path. A manual client MAY skip
it.

Because the check runs offline on the client, it also tells a client
**which** code belongs to **which** device: a fleet tool holding a whole
waybill of codes sends one challenge to each `pair=code` device it
discovers and tests the returned `proof` against every code on the list —
no serial numbers or physical matching needed.

### Step 2: create a credential — `POST /garden/pair`

Token, for an app:

```json
{ "method": "token", "name": "OFL app (Pixel 8)", "setup_code": "7K2M-Q9XD-4HTR-8WCN" }
```

```json
{ "client_id": "c1", "token": "ofl_3hT9…", "garden_id": "mfrggzdfmztwq2lk", "fingerprint": "sha256//Yx8m…=" }
```

Login/password, for scripts, Home Assistant, curl:

```json
{ "method": "password", "name": "Laptop", "username": "vlad", "password": "correct horse battery" }
```

```json
{ "client_id": "c2", "garden_id": "mfrggzdfmztwq2lk", "fingerprint": "sha256//Yx8m…=" }
```

`setup_code` is required on the setup-code path and not needed while the
button window is open. Both answer `201`. A client stores `fingerprint`
and pins it from now on.

| Error | When |
|---|---|
| `403 pairing_closed` | Neither path currently allowed. |
| `403 invalid_setup_code` | Setup-code path, wrong code. Counts toward the brute-force limit (§22.5). |
| `409 username_taken` | `username` already used by another credential. |
| `409 no_free_slots` | Credential storage full (§22.6). |
| `400 bad_request` | Missing field, unknown `method`, or `password` violating §22.5. |

### Security considerations

On the button path nothing proves the device's identity to the client,
and anyone on the local network can pair during the open window. The
exposure is limited to an attacker who is already on the LAN and active
during a 2-minute window that someone physically opened, and it is
visible afterwards: every credential appears in `GET /garden/clients` and
can be revoked. A Control Panel SHOULD show the credential list
prominently after pairing. Installations that need stronger guarantees
use the setup-code path.

## 22.5 Authenticating requests

Two standard HTTP schemes, equally valid everywhere:

| Scheme | Header | Credential |
|---|---|---|
| Bearer ([RFC 6750](https://www.rfc-editor.org/rfc/rfc6750)) | `Authorization: Bearer ofl_3hT9…` | Token from pairing. 32 random bytes, base64url, prefixed `ofl_`. |
| Basic ([RFC 7617](https://www.rfc-editor.org/rfc/rfc7617)) | `Authorization: Basic base64(username:password)` | Login/password from pairing. |

Basic sends the password in every request; this is acceptable only
because OFL runs exclusively over TLS (§22.2).

Requirements:

- `username`: 1–32 characters, `[a-z0-9._-]`.
- `password`: at least 10 characters, at most 64; MUST NOT equal the
  username or the device's setup code.

Missing or wrong credentials → `401 unauthorized` with
`WWW-Authenticate: Bearer realm="ofl", Basic realm="ofl"`.

**Brute-force limit:** after 5 consecutive failed authentications (any
scheme, any client, including wrong setup codes), the Head Device accepts at most one authentication
attempt per 10 seconds and answers the rest `429 too_many_attempts`, until
one attempt succeeds.

Example, manual use:

```sh
curl --pinnedpubkey 'sha256//Yx8m…=' -k -u vlad:'correct horse battery' \
     https://ofl-peer1.local/garden/plants
```

(`-k` skips CA validation, which doesn't apply to a self-signed device
certificate; `--pinnedpubkey` provides the actual check.)

## 22.6 Credential management

A Head Device holds a fixed number of credentials, **at least 4**,
implementation-defined beyond that. Tokens and passwords share the slots.

| Method | Path | Notes |
|---|---|---|
| `GET` | `/garden/clients` | `[{ client_id, name, method, username? }]` — never returns tokens or passwords. |
| `POST` | `/garden/clients` | Same body as `POST /garden/pair` without `setup_code`, same response. Lets an already-authenticated client add a credential (e.g. the app creating a login for a script) without the pairing window. |
| `PUT` | `/garden/clients/{client_id}` | `{ name?, password? }` — `password` only for `method: password`. A token is never changed, only deleted and re-issued. |
| `DELETE` | `/garden/clients/{client_id}` | Revokes immediately, including the caller's own credential. Deleting the last credential makes the device unclaimed (§22.4). |

## 22.7 Connection handling

The single-connection model ([REST API §16.3](16-rest-api.md#163-concurrency))
would let any host on the network block the Control Panel just by opening
a connection and staying silent. A Head Device therefore MUST close a
connection on which no complete request has arrived within **5 seconds**
— counted from accept, and again from the end of each response on a
kept-alive connection.

## 22.8 Factory reset

A **long press (≥ 10 s)** of the pairing button performs a factory reset.
It erases:

- every credential (§22.6);
- every user-supplied data item: Garden/Plant/System names and
  descriptions, Preferences, Settings, Grow Profiles, imported Culture
  Packs, `specialized`/`script` Scenarios;
- the device key pair — a new one is generated, so the device comes back
  with a **new `fingerprint` and `garden_id`** and is treated as a new
  Garden by every Control Panel.

It keeps the setup code (if any), firmware, and library-sourced data
([Hardware Calibration](06-hardware-calibration.md), `generic`
Scenarios). After a reset the device is unclaimed (§22.4): pair it by button,
or by its setup code if it has one.

## 22.9 Storage of secrets

- Tokens are stored only as SHA-256 hashes.
- Passwords are stored only as salted hashes produced by a slow key
  derivation function (PBKDF2-HMAC-SHA256 or stronger). To keep
  per-request cost low on constrained hardware, a Head Device MAY cache in
  RAM the hash of an `Authorization` header it has already verified,
  dropping the cache whenever any credential changes and on reboot.
- The private key, setup code (if any) and credential hashes SHOULD be kept in
  encrypted flash storage where the hardware supports it.
