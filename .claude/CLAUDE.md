# tarmac

At the start of a new session, read `CHANGELOG.md` in this repo's root first
— it records what's actually been built and why, and is the fastest way to
get oriented without re-deriving context from scratch.

## What this project IS

`tarmac` is a flight-connection risk scoring service. Given an origin
airport, a connecting airport, and a layover duration, it scores how risky
that connection is likely to be, by comparing live flight conditions against
a real historical baseline. It's a deployed, independent service with its
own API — it owns all of the domain logic (what "risk" means, how live and
historical data combine into a score) end to end.

## Dependencies this service is built on

- **`pipeline_runtime`** — an installed feature-store framework dependency.
  `tarmac` writes its own adapter against `pipeline_runtime`'s SDK (see
  below) and configures `pipeline_runtime`'s engine as its feature server —
  the process that ingests live flight data, computes windowed aggregates,
  and serves them over HTTP. `tarmac` owns the adapter and the deployment
  config; `pipeline_runtime` owns the generic engine that runs them.
- **OpenSky Network** — the live, per-flight data source. Free, community
  ADS-B aviation data. `tarmac`'s adapter polls its arrivals-by-airport
  endpoint, which returns an array of many flights per call, not one.
- **US DOT Bureau of Transportation Statistics (BTS) On-Time Performance
  dataset** — the historical baseline. Free, official monthly government
  data with scheduled vs. actual times, delay minutes, cancellations, and
  diversions per flight. `tarmac` bulk-loads this into its own baseline
  store to compare live conditions against real historical norms per
  route/airport/time-of-day.

## The OpenSky adapter (owned entirely by this repo)

```python
# tarmac_adapters/opensky_flights.py
from pipeline_runtime.sdk import SourceAdapter, AdapterConfig, Event
from datetime import datetime
from typing import Iterable
import json

class OpenSkyConfig(AdapterConfig):
    """Empty for now — this adapter needs no settings of its own, since the
    connector's request list already determines what gets polled. Kept as a
    named class rather than deleted so the pattern is consistent for
    adapters that DO need config."""
    ...

class OpenSkyFlightsAdapter(SourceAdapter):
    config_schema = OpenSkyConfig

    def parse(self, raw: bytes) -> Iterable[Event]:
        records = json.loads(raw)   # OpenSky returns a JSON array per call
        for record in records:
            yield Event(
                entity_key=f"{record['icao24']}_{record['callsign'].strip()}",
                timestamp=datetime.utcfromtimestamp(record["lastSeen"]),
                fields={
                    "airport": record["estArrivalAirport"],
                    "departure_delay_minutes": record.get("departure_delay_minutes"),
                },
            )
```

This adapter is real, versioned `tarmac` code — it just happens to implement
an interface (`SourceAdapter`) defined by an external dependency.

## Deployment shape

`tarmac`'s feature-server deployment installs both `pipeline_runtime` and
this repo's own `tarmac_adapters` package into the same container, and runs
`pipeline_runtime`'s CLI (pointed at a config file that references
`tarmac_adapters.opensky_flights` by dotted path) as its feature-server
process. That process is entirely `pipeline_runtime`'s code, running
entirely `tarmac`'s adapter.

`tarmac`'s own scoring service then:
- Calls that feature server's `/features/{entity_key}` endpoint over HTTP —
  a real network hop even though it's `tarmac`'s own deployment, to keep the
  boundary between the generic engine and the domain logic honest.
- Compares those live values against `tarmac`'s own BTS-derived historical
  baseline store.
- Computes and serves `GET /score/{origin}/{destination}`.

## Why the BTS baseline has its own store, separate from the feature server

The feature server (`pipeline_runtime`, configured by this repo) is
deliberately kept to live, current-window aggregates only — it has no
bulk-load path. Bulk-loading BTS's historical records (all past timestamps)
into that same store would risk the feature server returning a value
influenced by an event timestamped after the query time — a point-in-time
correctness bug. `tarmac` avoids this by keeping its BTS baseline in its own
separate store, loaded by its own `bts_loader.py`, and combining live +
historical values only at scoring time, in `tarmac`'s own scoring logic —
never inside the feature server itself.

## Suggested structure

```
tarmac_adapters/
  opensky_flights.py         # the actual adapter
tarmac/
  baseline/
    bts_loader.py
    store.py
  scoring.py
  api.py
configs/
  feature_server.yaml         # references tarmac_adapters via dotted path
Dockerfile.feature-server       # installs pipeline_runtime + tarmac_adapters,
                                  #   runs the feature-server process
Dockerfile.scoring-api
tests/
```

## Working agreement

Explain a change and why it's shaped that way before writing code; wait for
confirmation before implementing — small, reviewed steps rather than large
generated diffs absorbed after the fact. Append new entries to
`CHANGELOG.md` at the top, short — what was built/decided and the one-line
reason, not a full diff.

**Keep this `CLAUDE.md` current.** It's a living document, not a one-time
setup artifact. Whenever a decision here changes, a new component gets
added, or an architecture choice shifts, update this file as part of the
work — not as a separate cleanup pass — so the next session starts from
accurate context instead of a stale snapshot.
