# HexaBlock Suite — API (backend)

REST API of HexaBlock Suite: users, progress, scores and
level definitions for the four variants (Hexa Lines, Hexa Fit, Hexa Sort, Hexa Merge).
The client lives in the sibling repository `UCAB-hexablock-game`.

## Status

- **Stack: not decided yet.** Do not scaffold a project, add dependencies or pick a language or framework
  until the team records the decision here. If a task needs it, stop and ask.
- Commands (fill in once the stack exists): install `TBD` · lint `TBD` · test `TBD` · run `TBD`

## Engineering rules

- **Clean Architecture**, four layers, dependencies point inward only. Domain and use cases import nothing
  from frameworks, the database or HTTP.
- **`gameId` is a mandatory dimension**: one progress entry per user per variant; leaderboards per `gameId`
  (and per level for Hexa Fit and Hexa Sort). No use case, endpoint or table is duplicated per game.
- **One generic level schema** with a `gameId`-specific section. Levels can be added without an app update.
- JWT authentication, Swagger / OpenAPI documentation, consistent HTTP error responses.
- **SOLID and GoF patterns**: each one used must be documentable in the README with a real code reference.
- **AOP without AOP libraries**: decorators or proxies over use cases (logging, error handling, auth, cache).
  No logger calls inside business code.
- **Tests**: TDD, see "Testing" below.
- Code, comments, commits, pull requests and the README are written **in English**.

## Testing (TDD, statement section 3.6)

- **TDD is mandatory.** Red → green → refactor: write a failing test that states the expected behaviour, run
  it and see it fail for the right reason, write the minimum code to pass, then refactor with the suite green.
  No production code without a test that required it. Bug fixes start with a test that reproduces the bug.
- **AAA** in every test: `// Arrange`, `// Act`, `// Assert` blocks, one action per test.
- **Names**: `should_[expected_result]_when_[condition]`, e.g.
  `should_clear_three_lines_when_piece_completes_all_axes`.
- **Behaviour, not implementation.** Assert on outputs, returned state, persisted data, HTTP responses or what
  the user sees, through the public API of the unit. Do not assert on private methods, internal fields, call
  order or how many times a collaborator was called, unless that interaction *is* the behaviour (e.g. a
  cache decorator must not hit the repository twice). A refactor that keeps behaviour must not break tests.
- **Test doubles only at the boundaries** (ports: repositories, network, clock, random generator with a fixed
  seed). Prefer hand-written fakes over mocking libraries; never mock the unit under test or value objects.
- Tests are deterministic and independent: no real network, no shared mutable state, no order dependence.
- All suites run in CI on every pull request; a red suite blocks the merge.

Required suites:

- **Unit**: every entity, use case and service in isolation, with fakes for the ports. Includes the AOP
  decorators (logging, error handling, auth, cache) over a fake use case.
- **Integration**: use cases with real repositories on an in-memory database; REST endpoints through an
  HTTP test client, including filtering by `gameId`, auth failures and error responses; save-then-load
  progress end to end for **each** of the four variants.
- **Contract**: provider verification of the consumer contracts published by `UCAB-hexablock-game`
  (Pact or equivalent, https://docs.pact.io/). A change that breaks a contract fails CI.

## Git workflow (enforced by `.githooks/`, CI and branch protection)

- `main` ← `develop` ← working branches. Never commit or push to `main` or `develop`. Never use
  `--no-verify` or `--force`.
- Branches: `<type>/<kebab-case>`, type in `feature|fix|docs|chore|refactor|test|ci`. Start with `/start-task`.
- Commits: Conventional Commits, `type(scope): description` — English, lowercase start, no final period,
  at most 100 characters. Typical scopes: `auth`, `progress`, `leaderboard`, `levels`, `ci`, `readme`.
- **Small, frequent commits**: one logical change each. Never bundle
  unrelated work and never commit on behalf of another member.
- Finish with `/open-pr`: pull request into `develop`.

## AI usage (must always be documented)

- Every prompt is appended automatically to `docs/ai-usage/prompts/<member>.md`. Never edit or delete
  entries there, and stage that file with every commit.
- `AI_USAGE.md` holds **one entry per main use** (a theme such as "agentic harness" or "generic engine"),
  not one per session or PR. Continuing a theme updates its entry (dates, branches, one line in Prompts and
  Result); only a new theme adds an entry. `/open-pr` drafts it. "Team modifications" come from the human:
  ask, never invent.
- Architecture, generics, patterns and principles are **team decisions**: lay out options and trade-offs,
  do not decide silently. Every team member must be able to explain any line of the codebase.
- Generated code ships with tests. Cite non-trivial algorithms or external sources in a comment.
- Never put credentials, API keys or user data in prompts, code or logs.
