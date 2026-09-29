---
title: "Event Sourcing: CQRS"
layer: guide
audience: [agent, human]
stage: stable
---

# Event Sourcing: CQRS

*Command Query Responsibility Segregation: split the service into one that changes state and one that answers reads. That is the whole pattern.*

---

## The definition

Given a service with mixed methods:

```
FooService.do_foo(id, x)        # mutates
FooService.do_some_other_foo()  # mutates
FooService.read_foo(id)         # returns
FooService.read_bar(id)         # returns
```

CQRS splits it on the operation type:

```
FooWriteService.do_foo(id, x)        # commands
FooWriteService.do_some_other_foo()
FooReadService.read_foo(id)          # queries
FooReadService.read_bar(id)
```

That is all it is: two services where there was one, divided by whether a
method mutates state or returns data.

---

## The invariant

- **Commands** mutate state and do not return domain data.
- **Queries** return data and do not mutate state.

The guarantee this buys: a call that returns a value can be assumed not to
have changed anything, and a call that changes things will not be used as
a read. In a system where commands are also *events being appended*, this
invariant is what keeps the write path and the read path from entangling.

---

## What CQRS is not

- **Not distributed by definition.** Two modules in one service is CQRS;
  the split to separate services is a deployment choice, not the pattern.
- **Not dependent on event sourcing.** You can segregate responsibilities
  over a plain CRUD database. Event Sourcing happens to make the read side
  (projections) fall out naturally, but the two are independent choices.
- **Not inherently complex.** The confusion in the wild comes from people
  bundling every hard problem into the word. The pattern itself is a
  one-line split.

---

## Choosing

- Start with one service, split by *operation type* (interfaces), not by
  process, if that is all you need.
- Split to separate deployments when the read side needs different
  scaling, different data (denormalized read models), or different
  availability than the write side.
- Do not adopt CQRS to look correct; adopt it when the write path and the
  read path want to optimise differently. See
  [COMMANDS_AND_QUERIES](COMMANDS_AND_QUERIES.md).
