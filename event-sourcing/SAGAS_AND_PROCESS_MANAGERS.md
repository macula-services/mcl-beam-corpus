---
title: "Event Sourcing: Sagas and Process Managers"
layer: guide
audience: [agent, human]
stage: stable
---

# Event Sourcing: Sagas and Process Managers

*Two ways to run a business process that spans several aggregates or services. A saga moves forward step by step and undoes completed steps when it cannot finish; a process manager is a stateful coordinator that decides the next step from what it has seen.*

---

## The terms, and why they get confused

"Saga" has two meanings in circulation:

1. **The original** (Garcia-Molina and Salem, 1987): a long-running
   transaction split into a sequence of local transactions, each paired
   with a *compensating* transaction. If step 4 fails, the compensations
   for steps 3, 2 and 1 run. No distributed lock is held throughout.
2. **The CQRS-community usage**, where "saga" often names any piece of
   code that coordinates messages, which is really a process manager.

Microsoft's CQRS Journey adopted "process manager" for the second
meaning to avoid the clash; this corpus does the same.

| | Saga | Process manager |
|---|---|---|
| Shape | A planned sequence of steps; the plan can travel with the message (a *routing slip*) | A state machine instance per process, correlated by id |
| Where state lives | In the message / slip, or in each step's own records | In the process manager (as events, if event-sourced) |
| Good at | Mostly linear flows with a clear undo for each step | Branches, waiting, timeouts, retries, loops |
| Failure handling | Run compensations for completed steps | Decide: retry, compensate, escalate to a human |

Start with the simpler shape. A forward-only flow with compensations is
easier to reason about than a state machine. Move to a process manager
when the flow needs to wait, branch or revisit earlier steps; a small
change in business rules is often enough to tip the balance.

## A process manager, concretely

A seat purchase touching reservation, payment and ticketing:

```
SeatReserved      ─▶ PM: state :reserved   ─▶ command ChargeCard
PaymentCaptured   ─▶ PM: state :paid       ─▶ command IssueTicket
PaymentDeclined   ─▶ PM: state :declined   ─▶ command ReleaseSeat
(15 min, no payment) timeout ─▶ PM         ─▶ command ReleaseSeat
TicketIssued      ─▶ PM: state :done (process ends)
```

The process manager holds no business rules of its own; those stay in
the aggregates. It routes, waits, and makes sure the process ends in a
known state.

## Event-sourcing the process manager

An event-sourced process manager stores the events it has received (or
its own decisions) and derives its state by folding them, exactly like
an aggregate. The benefit shows up over long lifetimes. A process that
runs for weeks will outlive several deployments; if its state is a
stored snapshot, every new version must understand every old snapshot
format. If its state is derived from events, a new version can simply
fold the same events with new logic.

## Changing a process while instances are running

Do not silently swap the definition under running instances. If version
2 reorders the steps, an instance halfway through version 1 may skip a
step or do one twice. Options, from simplest:

1. **Pin instances to their starting version.** Old instances finish on
   the old code; new ones start on the new code. Deploy both until the
   old ones drain.
2. **Hand over explicitly.** The old version emits an event recording
   exactly where the instance stands (for example
   `PurchaseHandedOver{purchase_id, reserved: true, paid: false}`), and
   the new version subscribes to it and continues from that state.

## On the BEAM

A process manager instance maps well onto a supervised process
(GenServer or `gen_statem`) registered by correlation id, with timeouts
from `Process.send_after/3` or `gen_statem` state timeouts. Timeouts
must themselves be recorded or recomputed from events on restart, or a
crash will lose them.

## Sources

- Hector Garcia-Molina and Kenneth Salem, "Sagas", *Proceedings of ACM SIGMOD 1987* (free PDF). https://www.cs.cornell.edu/andru/cs711/2002fa/reading/sagas.pdf
- Microsoft patterns & practices, "Reference 6: A Saga on Sagas", *Exploring CQRS and Event Sourcing*, 2012 (free). https://learn.microsoft.com/en-us/previous-versions/msp-n-p/jj591569(v=pandp.10)
- Gregor Hohpe and Bobby Woolf, "Process Manager" and "Routing Slip", Enterprise Integration Patterns (free pattern pages). https://www.enterpriseintegrationpatterns.com/patterns/messaging/ProcessManager.html and https://www.enterpriseintegrationpatterns.com/patterns/messaging/RoutingTable.html
- Greg Young, *Patterns of Event Sourced Systems*, Leanpub (in progress, last updated 2025). https://leanpub.com/patternsofeventsourcedsystems
- Greg Young, *Versioning in an Event Sourced System*, Leanpub (last updated 2017; free to read online). https://leanpub.com/esversioning/read
