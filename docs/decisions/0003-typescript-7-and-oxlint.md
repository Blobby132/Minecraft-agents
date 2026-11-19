# 0003. TypeScript 7 with oxlint (fallback: TypeScript 6 + typescript-eslint)

- Status: Proposed
- Date: 2026-11-19
- Milestone: M1

## Context

`typescript@latest` is 7.0.2, the native Go compiler. 6.0.3 is the last JavaScript-based release. typescript-eslint 8.73 declares `typescript >=4.8.4 <6.1.0`, so ESLint type-aware linting does not support TS 7. oxlint 1.94 offers type-aware rules through `oxlint-tsgolint`, which is built on the TS 7 engine. Vite and Vitest transform TypeScript without `tsc`, and tsx uses esbuild, so neither depends on the TS version.

## Decision

- TypeScript 7.0.x for `typecheck` and `build` (`tsc -b`, project references). `strict`, `erasableSyntaxOnly` (which keeps Node's native type stripping available as an option), `verbatimModuleSyntax`.
- oxlint (with tsgolint type-aware rules) for linting, Prettier for formatting.
- If oxlint's type-aware mode turns out to be immature during M1 setup, switch to TS 6.0.3 + ESLint + typescript-eslint and record that by superseding this ADR.

## Consequences

- Fast type-checks. A newer linter ecosystem with fewer rules than typescript-eslint.

## Alternatives considered

- TS 6.0.3 + typescript-eslint: the most compatible today, but it is the last JS-based compiler and a transitional release.
- TS 7 with ESLint without type-aware rules: loses the most valuable rules, such as floating promises.
