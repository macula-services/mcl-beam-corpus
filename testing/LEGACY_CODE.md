---
title: "Testing: Legacy Code"
layer: guide
audience: [agent, human]
stage: stable
---

# Testing: Legacy Code

*Code without tests is not testable by accident. The seam is the tool: find where behaviour changes, break the dependency there, and test outward from the change you must make.*

---

## The definition that matters

Legacy code is not "old code" — it is **code without tests**, and
therefore code you cannot change safely. The testing problem and the
change problem are the same problem: you cannot modify what you
cannot verify.

---

## Seams — the lever

A **seam** is a place where behaviour can be changed without editing
the code: a function boundary, a module boundary, an injected
dependency, a message handler. Legacy code's problem is that its
seams were never exposed — everything was welded together.

The workflow:

1. **Pick the change** you must make — never refactor for its own
   sake.
2. **Find the seam** around the change: the smallest boundary that
   encloses what you will alter.
3. **Break the dependency** — introduce the seam (a function
   parameter, a behaviour, a mockable boundary).
4. **Test the seam** — pin the current behaviour with tests *before*
   changing it. These are characterization tests: they assert what
   the code *does*, not what it *should*.
5. **Change, then verify.** The characterization tests are the safety
   net that proves the change did not alter anything else.

---

## Characterization tests

The honest test for untested code: run it, observe the output, and
assert *that*. The tests document the code's actual behaviour — bugs
and all — so that the next change's side effects become visible.

Two rules:

- **Pin behaviour before refactoring.** Refactoring without
  characterization tests is archaeology without a rope.
- **Name the weirdness.** A characterization test that asserts a
  surprising output is documentation: "yes, this returns the string
  backwards — and this test proves it is still doing so."

---

## The dependency-breaking toolbox

| Technique | What it does |
|-----------|--------------|
| Extract function | Pull a chunk into a named, testable function |
| Introduce parameter | The hard-coded dependency becomes an argument |
| Introduce behaviour / protocol | A module boundary the test can substitute |
| Wrap the dependency | The untestable call sits behind a function you own |
| Spawn-and-message (BEAM) | A process boundary — the natural seam on this stack |

The BEAM's processes are the built-in answer: a GenServer API *is* a
seam, and `start_supervised!` with a substitute module makes legacy
process code testable without touching its internals.

## Rules of thumb

- **Test around the change, not the module.** The goal is a safe
  change, not a fully tested legacy system.
- **The seam is the investment.** Each seam you introduce is the
  interest that pays for every future change nearby.
- **Characterize first, refactor second, feature third.** In that
  order, or the refactor is a rewrite without a parachute.

## Why it matters

The mesh inherits code like anyone else: a legacy module in an
`mcl-*` service, a helper nobody dares touch. The legacy-code
discipline is what turns "we cannot change that" into "here is the
seam, here are the tests pinning it, here is the change" — the same
loop, applied to the code that scared everyone.
