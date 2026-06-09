# MQTT payload contract

This is the **single source of truth** for the messages the agent publishes and
every consumer (server, homekit, anything you build) reads. Producers must emit
exactly this shape; consumers must decode it defensively. If the contract changes,
it changes here first.

## Topics

The agent publishes under a per-inverter **base topic** (configurable; default base
`ghrian`, e.g. `ghrian/inverter/01`):

| Topic | Retained | QoS | Payload |
| --- | --- | --- | --- |
| `<base_topic>` | no (default) | 0 (default) | JSON data message (below) |
| `<base_topic>/availability` | **yes** | 1 | `online` or `offline` |

A consumer subscribes to one wildcard (`<base_topic>/#`) and matches each message to
an inverter by its topic, so new inverters need no consumer restart.

## Data message

```json
{
  "timestamp": "2026-06-04T12:34:56.789Z",
  "device_model": "solis_s5-eh1p5k-l",
  "values": {
    "active_power":            {"value": -1234, "unit": "W",   "label": "Active Power"},
    "battery_soc":             {"value": 87,    "unit": "%",   "label": "Battery Capacity SOC"},
    "inverter_temperature":    {"value": 41.2,  "unit": "°C",  "label": "Inverter Temperature"},
    "today_energy_generation": {"value": 5.6,   "unit": "kWh", "label": "Today Energy Generation"},
    "operating_status_active": {"value": ["Normal operation"], "unit": "", "label": "Operating Status (active bits)"}
  }
}
```

- `timestamp` — ISO 8601 (UTC), when the poll was taken.
- `device_model` — matches the agent's device definition name.
- `values` — a map of metric key → **self-describing object**:
  - `value` — a number, a string, or an array of strings.
  - `unit` — display unit (may be empty).
  - `label` — human-readable name.

### Bitfields

A bitfield metric `<name>` additionally emits `<name>_active`: an array of the
currently-active bit labels (see `operating_status_active` above). The base
`<name>` carries the raw/enum value; `<name>_active` carries the decoded list.

### Cumulative counters

`today_*` metrics (e.g. `today_energy_generation`) are **cumulative for the local
day** and reset at midnight. Consumers that want daily totals take the running max
per day rather than summing deltas.

## Availability

The retained `<base_topic>/availability` reflects inverter reachability:

- set `online` when the agent connects and is polling successfully;
- flipped to `offline` after a few consecutive poll failures (inverter unreachable);
- set `offline` on graceful shutdown (a clean MQTT disconnect suppresses the Last
  Will, so the agent publishes it explicitly);
- delivered to the agent's MQTT Last Will as the crash/network-drop fallback.

## Decoding rules for consumers

1. **Be defensive.** `value` may be number, string, or `[]string`. Missing keys are
   tolerated — render what's present, skip what isn't.
2. **Don't hard-code metric keys.** New inverter models add new keys; surface them
   generically from `values` rather than assuming a fixed set.
3. **Honor the retained availability** message for online/offline state, separate
   from the data stream.
4. **Sign conventions are encoded in the values**, not assumed — e.g. grid port
   power is positive on export / negative on import. Read the unit and label.
