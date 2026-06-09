# Architecture

```
inverter ──Modbus──▶ [agent] ──MQTT──┬──▶ [server]  ── DB · dashboard · REST API ──▶ clients
                                      └──▶ [homekit] ── HAP ──▶ Apple Home / Siri
```

ghrian is a small pipeline with one producer and many independent consumers, glued
together by MQTT.

## Design principles

- **One thing talks to the inverter.** Only the agent speaks Modbus. Inverters
  allow very few concurrent connections and dislike bursts of tiny reads, so
  centralizing access (with range-merged reads over one persistent connection)
  keeps the hardware happy. Consumers never touch the inverter.
- **Read-only, end to end.** Nothing ghrian runs ever writes to the inverter. The
  whole system is observational.
- **MQTT is the contract, not the implementation.** The agent publishes a
  [self-describing JSON payload](payload.md). Consumers depend only on that shape —
  never on the agent's internals, the register map, or each other. You can add,
  remove, or restart a consumer at any time without coordinating with the rest.
- **Self-describing data.** Each metric carries its own `value`, `unit`, and
  `label`, so consumers can render new metrics generically without a schema change.
  Adding a metric or a new inverter model is a data change, not a code change across
  the fleet.

## The modules

### agent (Go)

Polls the inverter over Modbus TCP on a fixed interval and publishes the curated
"live" metrics (PV / grid / battery / load power, SoC, temperatures, today's
energies, status, model/serial) to MQTT. Device register maps are JSON files
embedded in the binary, so supporting a new inverter model means adding a JSON file,
not rewriting code. Configured by file / env / flags. Stateless.

### server (Rails 8)

The reference consumer. Subscribes to the wildcard topic in a dedicated listener
process, persists every reading, and:

- rolls daily energy/cost totals up from the cumulative `today_*` counters, with
  month / year / lifetime aggregates;
- broadcasts live updates to a Hotwire dashboard (live power-flow diagram, battery
  and energy cards, intraday charts) over Turbo Streams;
- re-serves current state and time-series over a token-authenticated REST API for
  other clients;
- is an installable PWA.

Unlike the agent, the server has a UI, so its configuration (inverters, broker
connection, electricity tariff) lives in its database and is managed in the browser.

### homekit (Go, not yet public)

A separate consumer that bridges the same stream into Apple Home over HAP, so the
inverter shows up as accessories you can read in the Home app or ask Siri about.

## Why separate repos

Each module has its own language, dependencies, and release cadence, and they don't
share code — only the MQTT contract. Keeping them in independent repositories lets
each evolve and ship on its own, and lets a consumer be public or private
independently. This repo is the umbrella that documents the project as a whole and
points at the modules.
