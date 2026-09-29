---
title: "Event Sourcing: Ids and Correlation"
layer: guide
audience: [agent, human]
stage: stable
---

# Event Sourcing: Ids and Correlation

*Four ids on every message answer the two hardest questions in a messaging system: "what is this?" and "how did I get here?"*

---

## The four ids

| Id | Identifies | Copied or new? | Answers |
|----|-----------|----------------|---------|
| **Message id** | One message, uniquely | New per message | "Has this been processed?" (deduplication) |
| **Causation id** | The message that *caused* this one | Set to the parent's message id | "What directly caused this?" |
| **Correlation id** | One workflow (one logical operation) | Copied from the message being responded to | "Which conversation is this part of?" |
| **Conversation id** | One conceptual thing, across many workflows | Usually a domain id (an order id) | "What happened with this thing, overall?" |

---

## Message id — the deduplication anchor

Every message carries a unique id, normally a UUID. It is what makes
at-least-once delivery safe: without it, a receiver cannot know whether it
has already processed a message.

Ids can be built *from message content* — three servers producing the same
internal message then generate the same id, and receivers that see the
message three times process it once. Cheap availability gain; the cost is
having to prove the system is actually idempotent.

---

## Causation id — the causal chain

`FooOccurred` has message id `43e9…`. A service reacts, producing
`BarOccurred`: it copies the correlation id and sets causation id to
`43e9…` — the message id of what caused it.

Chained across services, causation ids let you answer **"how did I get
here?"** by reconstructing the causal graph of a conversation.

---

## Correlation id — the conversation

Every message carries a correlation id; anything responding to a message
copies it onto its own messages. All messages of one workflow share one
id:

```
OrderAccepted { correlationId: "7d03…" }
OrderPaid     { correlationId: "7d03…" }
OrderPicked   { correlationId: "7d03…" }
OrderShipped  { correlationId: "7d03…" }
```

Subscribing to a correlation id is subscribing to everything about this
one operation — a topic created implicitly by the id.

---

## Conversation id — when one correlation id is not enough

Some conceptual operations produce many workflows: a warehouse order
splits into three processes — direct fulfilment, a firearm approval, a
substitution. Each gets its own correlation id; all share the order's
**conversation id**.

Rules of thumb:

- The conversation id is almost always a **domain identifier** (an order
  id), because the question it answers is asked by people: "what happened
  with *this thing*?"
- Subscribe monitors and external viewers to the **conversation id**, not
  the correlation id — then sub-processes created later are still seen.
- Customer service is the canonical consumer: one screen, the whole
  lifecycle of the thing, across every split and follow-up order.

(Microsoft Service Broker calls the same idea a "conversation group".)

---

## The visualisations — build them

Two graphs are cheap to generate and change how teams talk about the
system:

1. **Conversation graph** (filter by correlation id, connect by causation
   id): the message flow of one conversation, "how did I get here?"
   answered visually.
2. **Flow overview** (10,000 correlation ids, line thickness = frequency):
   the high-level map of what happens in production, one screen.

Both are just graphs over two ids. The second one especially replaces
hours of whiteboard archaeology with a glance.
