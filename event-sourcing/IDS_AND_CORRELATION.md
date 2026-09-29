---
title: "Event Sourcing: Ids and Correlation"
layer: guide
audience: [agent, human]
stage: stable
---

# Event Sourcing: Ids and Correlation

*A few ids in every message's metadata let you deduplicate deliveries, trace what caused what, and gather everything that belongs to one piece of work.*

---

## The ids

| Id | Identifies | How it is set | Question it answers |
|----|-----------|---------------|---------------------|
| **Message id** | This message | Fresh for every message | Have I already handled this? |
| **Causation id** | The message that directly led to this one | The triggering message's *message id* | What caused this? |
| **Correlation id** | One workflow | Copied unchanged from the triggering message | Which piece of work is this part of? |
| **Conversation id** (optional) | One business thing across several workflows | Usually a domain id, such as an order id | Everything that ever happened about this thing? |

The message/correlation/causation trio is a widespread convention in
event-sourced systems, often credited to Greg Young; "correlation
identifier" itself is an older messaging pattern (Hohpe and Woolf).

## Message id and deduplication

Most transports deliver at least once, so a handler will occasionally
see the same message twice. A unique message id (a UUID, or a stream
name plus version) lets it recognise the repeat.

A useful variation is to derive the id deterministically from the
message's content and origin. If two redundant producers emit the same
logical message, they produce the same id and consumers handle it once.
That only helps if consumers really deduplicate, so treat it as a
property to test, not assume.

## Causation and correlation, by example

```
PlaceOrder        msg=a1  corr=a1  cause=none  (first message: corr = its own id)
OrderPlaced       msg=b2  corr=a1  cause=a1
ReserveStock      msg=c3  corr=a1  cause=b2
StockReserved     msg=d4  corr=a1  cause=c3
ChargeCard        msg=e5  corr=a1  cause=b2
```

The rule for any handler is mechanical: new message id, copy the
correlation id, set causation id to the id of the message being handled.

- **Filter by correlation id** to get every message of one workflow.
- **Follow causation ids** to rebuild the tree of what triggered what
  (here, `OrderPlaced` fanned out into two branches).

## When one workflow is not enough

Some business things spawn several independent workflows: an order
might need a stock reservation, a separate age check for one item, and a
later replacement shipment. Each has its own correlation id. A
conversation id, normally just the order id, ties them together so that
a support screen or a monitor sees the full life of the order, including
workflows started days later.

## Worth building

Two cheap visualisations from the same metadata:

1. **Per-workflow graph**: messages with one correlation id as nodes,
   causation ids as edges. The answer to "how did we get here?".
2. **Aggregate flow map**: sample many correlation ids, merge their
   graphs by message type, weight edges by frequency. A picture of what
   production actually does.

## On the BEAM

Put the ids in the event envelope's metadata, not in the payload, and
set them in one place (the command dispatcher and each handler's
emit helper) so nobody has to remember. Put the correlation id into
Logger metadata (`Logger.metadata(correlation_id: id)`) at the start of
each handler so logs can be joined to the event graph.

## Sources

- Greg Young, *Patterns of Event Sourced Systems*, Leanpub (in progress, last updated 2025). https://leanpub.com/patternsofeventsourcedsystems
- Robert Pankowecki, "Correlation id and causation id in evented systems", Arkency blog (free; quotes Greg Young's description of the convention). https://blog.arkency.com/correlation-id-and-causation-id-in-evented-systems/
- Gregor Hohpe and Bobby Woolf, "Correlation Identifier", Enterprise Integration Patterns (free pattern page). https://www.enterpriseintegrationpatterns.com/patterns/messaging/CorrelationIdentifier.html
