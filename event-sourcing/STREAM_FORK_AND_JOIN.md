---
title: "Event Sourcing: Stream Fork and Join"
layer: guide
audience: [agent, human]
stage: stable
---

# Event Sourcing: Stream Fork and Join

*Forking derives many streams from one by some key; joining merges many streams into one. Forking is cheap and safe. Joining forces you to decide what order means.*

---

## Fork: one stream in, many out

A fork reads one stream and writes each event into a derived stream
chosen by a key in the event. The canonical data stays where it is; the
derived streams are indexes for other access patterns.

```elixir
# every ticket sale also appears in a per-venue stream
def handle(%TicketSold{venue_id: venue} = e, meta) do
  link_to("sales-by-venue-" <> venue, meta.stream, meta.version)
end
```

Typical uses: per-region or per-customer views for reporting, or
partitioning work so that separate consumers each follow one derived
stream.

### Link, don't copy (usually)

The derived stream can hold either copies of the events or **links**:
small records pointing at the original stream and version.

- Links are small, so forks are cheap to add.
- There is one copy of the data. If the original is deleted or expires,
  the link visibly fails to resolve instead of leaving an orphaned copy
  behind, which matters for retention and erasure.
- Readers see the derived stream as an ordinary stream; resolution
  happens on read.

Copies are the right choice when resolving a link would be expensive or
impossible, for example when the original lives on another shard or
another store. KurrentDB's system projections (`$by_category`,
`$by_event_type`) are well-known examples of forks built with link
events.

## Join: many streams in, one out

A join reads several streams and produces a single combined stream, for
example all events for an order *and* its shipments, in one sequence.

The hard part is order:

- A fork's input is one ordered stream, so its outputs inherit a
  well-defined order.
- A join's inputs each have their own order, but there may be no single
  true order *between* them. If they live on different nodes or
  partitions, two events from different inputs may have no meaningful
  "which came first".
- Timestamps do not solve this: clocks on different nodes drift.
- The resulting order may look right almost always and still be wrong
  under load or during a partition, which is the worst kind of bug.

## Rules of thumb

- Fork freely. It is additive, cheap, and can always be rebuilt from the
  source stream.
- Before building a join, ask what the consumer actually needs. Many
  read models are fine with "each input in order, inputs interleaved
  arbitrarily".
- If you truly need one total order, get it at write time: write the
  events through one log (one partition or a global position in a single
  store) rather than trying to reconstruct it afterwards.

See [EVENT_LOGS](EVENT_LOGS.md) for streams and positions and
[PROJECTIONS](PROJECTIONS.md) for consumers of derived streams.

## Sources

- Greg Young, *Patterns of Event Sourced Systems*, Leanpub (in progress, last updated 2025). https://leanpub.com/patternsofeventsourcedsystems
- Kurrent, "System projections" (link events, `$by_category`), KurrentDB documentation (free). https://docs.kurrent.io/server/v25.0/features/projections/system.html
