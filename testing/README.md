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

None yet. See [Planned](#planned).

## Planned

- ExUnit: async tests, the database sandbox problem, doctests
- Property-based testing: PropEr/StreamData, generators, shrinking,
  stateful models
- Fault injection: killing processes, partitioning, testing supervision
- Testing projections: replay determinism, checkpoint resumption

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
