# Test-Driven Development

The red → green loop, plus what makes that loop produce tests worth keeping. Every part below applies on every cycle, so read it before the loop and consult it during, not after.

**What a good test is**: Tests verify behavior through public interfaces, not implementation details. Code can change entirely; tests shouldn't. A good test reads like a specification: "user can checkout with valid cart" tells you exactly what capability exists, and it survives refactors because it doesn't care about internal structure.

**Seams: where tests go**: A **seam** is the public boundary you test at, the interface where you observe behavior without reaching inside. Tests live at seams, never against internals.

The seams are already decided. They arrive in the `Test seams` column of the plan's Work units table, approved by the user along with the plan, so test at those and add none. A unit whose seams are `none` writes no tests at all, because what it changes isn't behaviour, and the plan's `## Verification` checks it instead. You can't test everything, and fixing the seams before implementation starts is how the effort lands on critical paths and complex logic instead of spreading over every edge case. If implementation shows that a listed seam is wrong, or that a critical one is missing, report it as a deviation from the plan rather than quietly testing somewhere else.

**Anti-patterns**:

- **Implementation-coupled**: mocks internal collaborators, tests private methods, or verifies through a side channel (querying the database instead of using the interface). The tell: the test breaks when you refactor but behavior hasn't changed.
- **Tautological**: the assertion recomputes the expected value the way the code does (`expect(add(a, b)).toBe(a + b)`, a snapshot derived by hand the same way, a constant asserted equal to itself), so it passes by construction and can never disagree with the code. Expected values must come from an independent source of truth: a known-good literal, a worked example, the spec.
- **Horizontal slicing**: writing all tests first, then all implementation. Bulk tests verify _imagined_ behavior: you test the _shape_ of things rather than user-facing behavior, the tests go insensitive to real changes, and you commit to test structure before understanding the implementation. Work in **vertical slices** instead: one test, one implementation, repeat, each test a **tracer bullet** that responds to what the last cycle taught you.

**Rules of the loop**:

- **Red before green.** Write the failing test first, then only enough code to pass it. Don't anticipate future tests or add speculative features.
- **Watch it fail, for the right reason.** Run the new test before you write any implementation and read the failure. It has to fail because the behaviour is missing, not because of a typo, an import error, or a broken test setup, and it must not pass on the first run. A test whose first failure you never read is a test you can't claim exercises anything, and a test that passes immediately is testing behaviour that already existed.
- **Stub the target before you write the test.** A test against a module that doesn't exist yet fails on the import, and an import error is the one failure the rule above tells you not to count. So create the function, type, or endpoint first, returning a fixed value that is honestly wrong: `ok: true` where the case expects a rejection, an empty list where it expects rows. The test then runs, compares, and fails on the value, which is a failure about the behaviour being missing. Three lines of stub is what buys you the reading this loop depends on. Don't make the stub clever, though. A stub that guesses at the real logic can satisfy a case by accident, and a case that passes on its first run tells you nothing.
- **One slice at a time.** One seam, one test, one minimal implementation per cycle.
- **Refactoring is not part of the loop.** Once the tests pass, report what you'd restructure as a note in your report rather than doing it inside the red → green cycle.
- **A missing dependency isn't a red test.** A test or typecheck that fails because a module, package, binary, or generated type can't be resolved means the repo isn't set up: run its setup once (`CLAUDE.md`, the README, a `setup` or `prepare` script, a Makefile target) and re-run. Never fix that in code, and never count it as the failure **Watch it fail, for the right reason** asks you to read.
