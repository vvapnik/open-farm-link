# 5. Vitals History

Vitals History is a time-series of past [Vitals](04-vitals.md) readings
for a Plant **or** a [System](10-system.md) — the format is shared by
both, since neither the record shape nor its lifecycle depends on which
kind of owner it belongs to. This specification defines only the record
**format** — how a Head Device chooses to store, retain, or aggregate
history internally is an implementation detail, deliberately left
unconstrained.

## 5.1 Format

```json
{
  "owner_id": "plant-1234",
  "timestamp": "2026-09-06T14:00:00Z",
  "interval": 3600,
  "vitals": [
    { "type": "moisture", "value": 61, "unit": "percent" },
    { "type": "temp", "value": 20.9, "unit": "celsius" }
  ]
}
```

| Field | Type | Description |
|---|---|---|
| `owner_id` | `string` or `null` | The parent [Plant](03-plant.md)'s `plant_id` when this history belongs to a Plant. `null`/omitted for [System](10-system.md), which is a single, unaddressed entity per Garden. |
| `timestamp` | `string` (ISO 8601, UTC) | Start of the period this record covers. |
| `interval` | `uint32` (seconds) | Length of the period this record aggregates. `0` means a raw, unaggregated sample rather than an aggregate over time. |
| `vitals` | array | Same shape as a live [Vitals](04-vitals.md) reading — the value(s) for this period (an implementation may store e.g. an average, min/max, or last-sample here; the format doesn't distinguish which). |

## 5.2 Why `interval` is per-record, not per-history

A single Plant's history is expected to hold records of different
granularity as they age — recent data at fine granularity, older data
rolled up coarser. Making `interval` a field on each record (rather than a
fixed property of "the history") means a single history array can freely
mix granularities without any extra structure: a consumer reading history
doesn't need to know an implementation's retention policy, just this
per-record field.

This specification does not require any particular retention or
aggregation scheme — an implementation may return only raw samples, only
one granularity, or a mix, and remain compliant.
