# ghrian

Self-hosted solar inverter monitoring. ghrian reads live data from a solar
inverter over Modbus, publishes it as a clean MQTT stream, and lets any number of
independent consumers do something useful with it — persist it, chart it, or bridge
it into your smart home.

```
inverter ──Modbus──▶ [agent] ──MQTT──┬──▶ [server]  ── DB · dashboard · REST API ──▶ clients
                                      └──▶ [homekit] ── HAP ──▶ Apple Home / Siri
```

The design is deliberately decoupled: the **agent** is the single thing that talks
to the inverter, and everything downstream is just a subscriber to its MQTT topics.
Adding a new consumer never means touching the inverter or the agent — you only
subscribe to the stream. The whole pipeline is **read-only**: nothing ghrian runs
ever writes to your inverter.

## The name

**ghrian** comes from the Irish word *grian*, "sun" — as in *an ghrian*, "the sun",
the lenited spelling the project borrows. A fitting patron for solar monitoring: the
sun is the logo, and `ghrian` is the default MQTT topic the whole stack speaks over.

## Modules

Each module is its own repository with its own README and release lifecycle.

| Module | What it does | Stack | Repo |
| --- | --- | --- | --- |
| **agent** | Polls the inverter over Modbus TCP and publishes a rich, self-describing JSON stream to MQTT. Gentle on the inverter (range-merged reads, one persistent connection). | Go | [ghrian-agent](https://github.com/zavan/ghrian-agent) |
| **server** | Subscribes to the stream, persists it, and serves a Hotwire dashboard (live power flow, energy & cost history) plus a token-authenticated REST API. Installable PWA. | Rails 8 | [ghrian-server](https://github.com/zavan/ghrian-server) |

> A HomeKit bridge (`homekit`) also consumes the same stream to expose the inverter
> in Apple Home; it isn't public yet.

## How it fits together

1. The **agent** runs next to the inverter (or anywhere that can reach it on the
   network), polls a curated set of registers, and publishes one MQTT message per
   inverter per cycle to a per-inverter topic (e.g. `ghrian/inverter/01`), plus a
   retained `<topic>/availability` flag.
2. Any number of **consumers** subscribe to that topic. They never need to know
   about Modbus, the inverter's quirks, or each other — they just read the
   [shared payload contract](docs/payload.md).
3. The **server** is the reference consumer: it stores every reading, rolls up
   daily/monthly/yearly energy and cost totals, broadcasts live updates to the
   dashboard, and re-serves everything over REST for other clients.

See [docs/architecture.md](docs/architecture.md) for the design principles and
[docs/payload.md](docs/payload.md) for the MQTT message format every module agrees
on.

## Quick start

You need an MQTT broker (e.g. Mosquitto) the agent and consumers can both reach.

1. **Run the agent** against your inverter — see
   [ghrian-agent](https://github.com/zavan/ghrian-agent). Point it at your
   inverter's IP and your broker; it starts publishing under `ghrian/#`.
2. **Run the server** — see
   [ghrian-server](https://github.com/zavan/ghrian-server). Configure the same
   broker and add an inverter whose topic matches what the agent publishes. The
   dashboard fills in live.

Each module's README has the full, copy-pasteable setup for that piece.

## Run the whole stack (Docker Compose)

This repo ships a [`compose.yml`](compose.yml) that runs the broker, the server
(web + MQTT listener), and optionally the agent from the published images — no
Rails `master.key` required, since the server reads its secrets from the
environment.

```bash
cp .env.example .env     # then fill in the generated secrets (instructions inside)
docker compose up -d     # starts broker + server web + listener
```

The broker connection is preconfigured (the server is seeded with host `broker` /
base topic `ghrian` on first boot), so just open <http://localhost:3000>, create the
first account (it becomes the admin), and add an inverter with topic
`ghrian/inverter/01`.

To also run the agent against a real inverter (set `MODBUS_ADDR` in `.env` first):

```bash
docker compose --profile agent up -d
```

Images default to `latest`; pin reproducible deploys with `SERVER_TAG` / `AGENT_TAG`
in `.env`. Both are multi-arch (amd64 + arm64):
[`felipezavan/ghrian-agent`](https://hub.docker.com/r/felipezavan/ghrian-agent) ·
[`felipezavan/ghrian-server`](https://hub.docker.com/r/felipezavan/ghrian-server).

## License

MIT — see [LICENSE](LICENSE). Each module is MIT-licensed as well.
