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
- **Tests**: unit (isolated, fakes or stubs for ports) and integration (in-memory database, HTTP endpoints
  including filtering by `gameId`). AAA structure. Names: `should_[expected_result]_when_[condition]`.
  Test behaviour, not implementation details.
- Code, comments, commits, pull requests and the README are written **in English**.

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
- Significant uses get a curated entry in `AI_USAGE.md` (`/open-pr` drafts them). "Team modifications" and
  "Lessons learned" come from the human: ask, never invent.
- Architecture, generics, patterns and principles are **team decisions**: lay out options and trade-offs,
  do not decide silently. Every team member must be able to explain any line of the codebase.
- Generated code ships with tests. Cite non-trivial algorithms or external sources in a comment.
- Never put credentials, API keys or user data in prompts, code or logs.
