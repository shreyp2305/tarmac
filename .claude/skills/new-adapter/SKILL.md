---
name: new-adapter
description: Scaffold a new SourceAdapter implementation in this repo, following the shape of the existing adapter(s)
---

# new-adapter

Use this when adding a new adapter to this repo — the code that turns a raw
payload from some data source into `Event`s, implementing the `SourceAdapter`
interface from the `pipeline_runtime` package this repo depends on.

## The established shape

```python
from pipeline_runtime.sdk import SourceAdapter, AdapterConfig, Event
from datetime import datetime
from typing import Iterable
import json

class SomeSourceConfig(AdapterConfig):
    """Even if this ends up empty, keep it as a named class — consistent
    with every other adapter in this repo, and ready for the day this
    source does need its own settings (an API key, a lookback window)."""
    ...

class SomeSourceAdapter(SourceAdapter):
    config_schema = SomeSourceConfig

    def parse(self, raw: bytes) -> Iterable[Event]:
        records = json.loads(raw)   # shape this to the actual payload format
        for record in records:
            yield Event(
                entity_key=...,      # a stable, unique key per real-world entity
                timestamp=...,
                fields={...},
            )
```

## Steps

1. **Always define a `config_schema`, even if empty.** Every adapter in this
   repo declares one — it's what lets a misconfigured deployment fail fast
   with a specific validation error instead of a mysterious `KeyError` deep
   inside a request.
2. **`parse()` yields, it doesn't return one value.** A single raw payload
   commonly contains many records (e.g. a paginated API response) — never
   assume 1:1.
3. **Think hard about `entity_key`.** It needs to be a stable, unique
   identifier for whatever real-world thing the event is about. Get this
   wrong and downstream aggregates silently mix data from different
   entities.
4. **Use `.get()` with sensible defaults for optional upstream fields**,
   rather than assuming a field is always present in the raw payload — a
   real API will eventually send you a record missing something.
5. **One file per source**, named for what it adapts (e.g.
   `tarmac_adapters/opensky_flights.py`).
6. **Test `parse()` directly**: feed it a raw sample payload (captured from
   the real source, not invented), and assert on the `Event`s it yields.

## A note on this pattern

This reflects the current best-known adapter pattern for this repo, modeled
on the first adapter written here. Expect to revisit it once a second
adapter exists and it becomes clearer what's genuinely shared structure
versus what was specific to the first source.
