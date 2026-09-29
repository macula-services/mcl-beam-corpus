---
title: Testing
layer: index
audience: [agent, human]
stage: stable
---

# Testing

*How to hold BEAM and event-sourced systems to a standard.*

---

## Notes

| Note | Covers |
|------|--------|
| [TESTING_EVENT_SOURCING](TESTING_EVENT_SOURCING.md) | Given/when/then command tests, replay tests, fault tests, tests as documentation |
| [PROPERTY_BASED_TESTING](PROPERTY_BASED_TESTING.md) | Invariants, generators, shrinking; PropEr and StreamData; stateful models |

## One-page summary

The BEAM's properties change what "unit test" means: processes and
message passing make tests that observe one module alone miss most of
the failure modes. The highest-value tests on this stack are:

1. **Property-based tests** for pure logic: state the invariant, let
   generators find the counterexample.
2. **Fault-injection tests** for supervision: kill a child, assert the
   tree recovers to the expected state.
3. **Replay tests** for projections: feed an event sequence, assert the
   read model — then feed it again from a restored checkpoint, assert
   idempotency.
