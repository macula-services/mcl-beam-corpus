---
title: "Event Sourcing: Stream Fork and Join"
layer: guide
audience: [agent, human]
stage: stable
---

# Event Sourcing: Stream Fork and Join

*Fork splits one stream into many; join merges many into one. They are inverse operations — and the join is where the hard questions live.*

---

## Stream fork (split)

Take one stream and produce many, by a criterion:

```
from_stream("sales").fork("bylocation-" + event.location)
```

Every sale event is reindexed into a `bylocation-{location}` stream. This
"reindexing" is a fundamental stream operation: keep one canonical stream
(per sales) and derive grouping streams (per location, per region) for
reporting needs.

### Link events, not copies

A fork should emit a **link event** — a pointer to the original — rather
than a copy:

- Links are smaller than the events they reference.
- The underlying stream can be deleted or scavenged; links remain and
  simply stop resolving — you can *see* that the underlying data is gone.
- Reads of the derived stream behave like reads of a normal stream.

Implement both link and copy: copies exist for when the store is sharded
and resolution would cross shards.

---

## Stream join

The inverse: take multiple streams, produce one. Common in systems — and
the source of subtler problems:

- **Fork knows its input is ordered.** Join does not: its sources can live
  in multiple locations.
- With one source, ordering assurances are easy. With several, they are
  not always obvious — and may hold 99.9% of the time.
- If the system is **partitioned across sources, a precise join may be
  impossible**. Two events from different streams have no single total
  order to respect.

---

## Rules of thumb

- Fork freely: it is cheap, reversible, and the standard move for
  reindexing/reporting.
- Before joining, ask which ordering the consumer actually needs. Many
  read models do not need a total order — they need "eventually correct".
- If the join must be precise, keep its sources co-located or sequence the
  join itself through a single log rather than reading two.
