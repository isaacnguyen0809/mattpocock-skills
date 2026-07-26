---
name: tdd
description: Test-driven development. Use when the user wants to build features or fix bugs test-first, mentions "red-green-refactor", or wants integration tests.
---

# Test-Driven Development

TDD is the red → green loop. This skill is the reference that makes that loop produce tests worth keeping: what a good test is, where tests go, the anti-patterns, and the rules of the loop. Every section applies on every cycle: consult them before and during the loop, not after.

When exploring the codebase, read `CONTEXT.md` (if it exists) so test names and interface vocabulary match the project's domain language, and respect ADRs in the area you're touching.

## What a good test is

Tests verify behavior through public interfaces, not implementation details. Code can change entirely; tests shouldn't. A good test reads like a specification: "user can checkout with valid cart" tells you exactly what capability exists, and it survives refactors because it doesn't care about internal structure.

See [tests.md](tests.md) for examples and [mocking.md](mocking.md) for mocking guidelines.

## Seams: where tests go

A **seam** is the public boundary you test at: the interface where you observe behavior without reaching inside. Tests live at seams, never against internals.

**Test only at pre-agreed seams.** Before writing any test, write down the seams under test and confirm them with the user. No test is written at an unconfirmed seam. You can't test everything, so agreeing the seams up front is how testing effort lands on the critical paths and complex logic instead of every edge case.

Ask: "What's the public interface, and which seams should we test?"

Confirm **all** the session's seams in that one exchange. The rule is one confirmation round covering the whole run, not a round-trip per cycle: the loop below is meant to run without stopping to re-ask.

When the shape of that interface is itself in question (how deep the module is, where the seam belongs, what the interface should expose), call the Skill tool with "codebase-design" for the vocabulary. It is the shared source of the module, interface, depth, seam, adapter, leverage and locality terms, and it is a reference to consult, not a session to run.

## Anti-patterns

- **Implementation-coupled**: mocks internal collaborators, tests private methods, or verifies through a side channel (querying the database instead of using the interface). The tell: the test breaks when you refactor but behavior hasn't changed.
- **Tautological**: the assertion recomputes the expected value the way the code does (`expect(add(a, b)).toBe(a + b)`, a snapshot derived by hand the same way, a constant asserted equal to itself), so it passes by construction and can never disagree with the code. Expected values must come from an independent source of truth: a known-good literal, a worked example, the spec.
- **Horizontal slicing**: writing all tests first, then all implementation. Bulk tests verify _imagined_ behavior: you test the _shape_ of things rather than user-facing behavior, the tests go insensitive to real changes, and you commit to test structure before understanding the implementation. Work in **vertical slices** instead: one test → one implementation → repeat, each test a **tracer bullet** that responds to what the last cycle taught you.

## Rules of the loop

- **Red before green.** Write the failing test first, then only enough code to pass it. Don't anticipate future tests or add speculative features.
- **One slice at a time.** One seam, one test, one minimal implementation per cycle.
- **Refactoring is not part of the loop.** It belongs to the review stage (see the `code-review` skill), not the red → green implementation cycle.

## Running the tests

The loop is deliberately many small cycles, so anything you repeat per cycle you pay for N times over. Three rules keep that cost flat:

- **Establish the test command once.** At the start of the session, work out how this repo runs a single test file and write the exact command down. Don't re-derive it — or re-read the test config — every cycle.
- **Run the narrowest command that can go red.** During the loop, run only the file or test name under change (`vitest run path/to/file.test.ts -t "name"`, `pytest path::test_name`, and so on) — never the whole suite. Run the full suite **once** at the end of the session, and again only if a slice touches shared code. A full suite per cycle is the single largest cost in a TDD run and it tells you nothing the narrow run didn't.
- **Keep the failure, drop the transcript.** From a red run, carry forward the failing assertion and its message — not the full runner output, stack traces of passing tests, or coverage tables. From a green run, carry forward one line. Earlier cycles' output is dead weight: never quote it back, and don't re-run a cycle you've already closed to "check" it.

If the loop is long enough that context is filling with closed cycles, stop at a green and hand off (see the `handoff` skill) rather than pushing through — a fresh context resumes cheaper than a saturated one continues.
