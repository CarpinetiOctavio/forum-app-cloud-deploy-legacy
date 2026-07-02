# ADR-008 — Spanish Test Descriptions Are Kept Intentionally

## Status

Accepted

## Date

2026

## Context

A portfolio/code-quality pass over the frontend test suite found that several
`describe()`/`test()` blocks in `CreatePost.test.tsx`, `CommentForm.test.tsx`,
`Login.test.tsx`, `authService.test.ts`, and `postService.test.ts` use
Spanish-language descriptions (e.g. `test('crea post exitosamente', ...)`),
while the rest of the codebase — source code, variable names, coverage
tooling, and CI output — is entirely in English.

Translating these description strings was considered as a cleanup step.

## Decision

Do not translate or otherwise edit the existing Spanish test descriptions.
Instead, an English comment was added directly above each Spanish
`describe()`/`test()` line, explaining what the test verifies, without
touching the test code itself.

## Rationale

- **Risk of editing files that also carry UI-coupled assertions.** These test
  files interleave the test description string with assertions bound to
  literal UI text (`screen.getByText('Create New Post')`,
  `getByRole('button', { name: /publish post/i })`, etc.). `docs/rules/testing.md`
  and `docs/rules/constraints.md` explicitly forbid modifying the existing
  test suite "unless a test is provably broken by a change introduced in this
  project" — editing description strings is exactly the kind of touch that
  risks an unintended edit to a nearby assertion, for no functional benefit.
- **Coverage and logic are unaffected.** Jest and Go coverage tooling, the
  70% coverage gate, and SonarCloud analysis all operate on code paths and
  assertions — never on the literal text of a test's description string.
  Coverage numbers (86.5% backend / 92.44% frontend, per `testing.md`) do not
  change whether a test is named in English or Spanish.
- **Comments give portfolio readers the same information without the risk.**
  An English comment placed above each Spanish test name lets anyone
  scanning the file understand what is being tested, without touching test
  code that constraints.md protects.

## Alternatives considered

| Alternative | Reason not chosen |
| --- | --- |
| Translate the Spanish description strings to English | Requires editing the same lines/files that contain UI-coupled assertions; violates the "do not modify the existing test suite" constraint for no functional gain — the string is a label, not a coverage or CI input. |
| Leave the Spanish descriptions with no annotation | Leaves a portfolio reader without an explanation, since not every reader will read Spanish. |
| Rewrite each test into `Should_When`/AAA naming per `testing.md`'s convention for *new* tests | That convention applies to newly added tests, not to the existing inherited suite, which `testing.md` says must be preserved exactly. |

## Consequences

- All 21 previously Spanish-only `test()`/`it()` blocks now have an English
  comment immediately above them.
- The test suite's structure, names, and assertions are unchanged —
  coverage numbers and CI behavior are unaffected.
- Any future contributor reading these files in English can understand each
  test's intent from the comment, without needing to modify test code that
  is off-limits per `constraints.md`.
