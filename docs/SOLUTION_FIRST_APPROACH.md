# Solution first approach — API (backend)

> **Read this first.**
>
> - **This is not the architecture of the project.** It is not even a first version of it. It is a first
>   approach: one possible way to lay out a solution, written down so the team has something concrete to
>   look at, discuss and take apart.
> - **It was generated with AI** (Claude Code, Claude Fable 5.2, on 2026-11-21) from the project requirements
>   and the team's early design notes. Its purpose is to visualise ideas, not to make decisions.
> - **Nothing here is decided.** Architecture, patterns and principles are team decisions. Every choice
>   below is a proposal, and section 12 lists the ones that most need a human answer.
> - **It has not been validated with code.** No stack is chosen yet, so nothing was compiled or run.
> - **The backend was explored less than the game.** The team's early notes only fixed the data model;
>   endpoints, use cases and layering below are new in this document and deserve a harder look.
> - The pseudocode is language-neutral. Names are suggestions.
>
> The sibling document for the mobile client is `docs/SOLUTION_FIRST_APPROACH.md` in `UCAB-hexablock-game`.

## 1. Ideas the approach rests on

- **The server stores and serves; it does not play.** The game engine lives in the client. The API keeps
  users, progress, level definitions and rankings.
- **`gameId` is data, not code.** No endpoint, use case or table is duplicated per game. A new variant is
  a row in `games`.
- **Syncing is a merge, never an overwrite.** The server joins what it has with what the client sends by
  taking the best of each field. The result is the same whichever device syncs first, and sending the same
  data twice changes nothing.
- **A level is a common envelope plus one game-specific section.** The server understands the envelope;
  the specific section is checked by a small validator chosen by `gameId`.
- **Use cases know nothing about HTTP, SQL or JWT.** They receive plain inputs, talk to ports and return
  plain outputs or domain errors.

## 2. Layers

```
4. Frameworks    web framework, database driver / ORM, JWT and hashing libraries, migrations, CompositionRoot
        │ depends on
3. Adapters      controllers, DTOs and mappers, auth middleware, error mapper, repository implementations,
        │        token service, password hasher
        │ depends on
2. Application   use cases, ports, AOP decorators
        │ depends on
1. Domain        User, Game, PlayerProgress, LevelRecord, Level, value objects, domain errors
```

Dependencies point inward only. Domain and use cases import nothing from the framework, the database or HTTP.

A possible folder layout, to be adapted to the stack:

```
src/
  domain/          entities, value objects, errors
  application/     use-cases/, ports/, aspects/
  adapters/
    http/          controllers, dto, middleware, error mapper
    persistence/   repository implementations, row mappers
    security/      token service, password hasher
  framework/       server bootstrap, database connection, migrations, config, composition-root
```

## 3. Layer 1: domain

```
class User          { id, username, email, passwordHash, role, createdAt }     // role: player | admin
class Game          { id: GameId, name, progressModel }                        // catalogue of variants
class Level         { id, gameId, number, version, difficulty, spec }          // spec = game-specific section
class LevelRecord   { levelNumber, stars, bestScore, completed }

class PlayerProgress {                    // aggregate root, one per (userId, gameId)
  userId, gameId, highScore, records: List<LevelRecord>, updatedAt
  merge(other: PlayerProgress): PlayerProgress
}
```

Value objects: `GameId` (`hexa-lines`, `hexa-fit`, `hexa-sort`, `hexa-merge`), `Email`, `Stars` (0 to 3).

**The merge rule**, identical to the client's `PlayerProgress.merge()`:

| Field | Result |
|---|---|
| `highScore` | the higher of the two |
| `stars` of a level | the higher of the two |
| `bestScore` of a level | the higher of the two |
| `completed` of a level | true if either is true |
| a level present on one side only | kept as is |

Invariants protected by the aggregate: stars never go down, scores are never negative, and there is at
most one record per level.

Domain errors, raised by entities and use cases and translated to HTTP only in layer 3:
`ValidationError`, `InvalidCredentials`, `Forbidden`, `NotFound`, `Conflict`.

`LeaderboardEntry { rank, username, score }` is a read model, not an entity: it is computed by a query and
never stored.

## 4. Layer 2: application

### 4.1 Use cases

All implement `IUseCase<TIn, TOut>` with a single `execute()`. `Actor { userId, role }` is the
authenticated caller; the controller builds it from the verified token.

| Use case | Input | Output |
|---|---|---|
| `RegisterUserUseCase` | username, email, password | token + user |
| `LoginUseCase` | email, password | token + user |
| `GetProgressUseCase` | actor, `gameId` | `PlayerProgress` (empty if none yet) |
| `SyncProgressUseCase` | actor, `gameId`, client progress | merged `PlayerProgress` |
| `GetLeaderboardUseCase` | `gameId`, optional level number, limit | list of `LeaderboardEntry` |
| `GetLevelsUseCase` | `gameId` | list of `Level` |
| `UpsertLevelUseCase` | actor, `gameId`, level | saved `Level` |

`SyncProgressUseCase`, step by step:

1. Check that the `gameId` exists (`IGameRepository`).
2. Check that every level record points to an existing level of that game.
3. Load the stored progress, or start from an empty one.
4. `stored.merge(incoming)`.
5. Save the result and return it.

`UpsertLevelUseCase`: check the envelope, look up the `ILevelSpecValidator` for that `gameId`, validate the
specific section, then save. A `gameId` with no registered validator cannot have levels.

### 4.2 Ports

| Port | Purpose |
|---|---|
| `IUserRepository` | find by email or username, save |
| `IGameRepository` | does this `gameId` exist |
| `IProgressRepository` | load and save a `PlayerProgress` by (userId, gameId) |
| `ILevelRepository` | list by `gameId`, find by (gameId, number), save |
| `ILeaderboardQuery` | top N by game, top N by game and level |
| `IPasswordHasher` | hash, verify |
| `ITokenService` | issue a token for a user, verify a token |
| `ILevelSpecValidator` | validate the specific section of a level; one implementation per `gameId` with levels |
| `IClock`, `ILogger` | time and logging |

### 4.3 AOP

`UseCaseDecorator` implements `IUseCase` and wraps another `IUseCase`. No AOP library; no logger calls
inside business code.

- **`LoggingDecorator`**: use case name, duration and outcome. Never logs passwords or tokens.
- **`AuthorizationDecorator(requiredRole)`**: rejects with `Forbidden` when the actor lacks the role. Wraps
  `UpsertLevelUseCase` with `admin`.

A cache decorator over `GetLeaderboardUseCase` is a natural third one, but only worth adding if the query
turns out to be slow.

Two cross-cutting concerns sit at the HTTP edge instead, because they are about HTTP itself: verifying the
token (auth middleware) and turning domain errors into responses (error mapper).

## 5. Layer 3: adapters

- **Controllers** (`AuthController`, `ProgressController`, `LeaderboardController`, `LevelsController`):
  parse the request into a DTO, build the use case input, call the use case, map the output to a response
  DTO. No business rules.
- **Auth middleware**: reads the bearer token, verifies it through `ITokenService`, attaches the `Actor`.
- **Error mapper**: the single place that knows which domain error becomes which HTTP status.
- **Repository implementations**: one per port, plus row-to-entity mappers. `LeaderboardQuery` holds the
  two ranking queries.
- **`JwtTokenService`** and **`PasswordHasher`**: wrap the JWT and hashing libraries behind the ports.
- **`FitLevelSpecValidator`** and **`SortLevelSpecValidator`**.
- **OpenAPI document**: generated from, or kept next to, the controllers.

## 6. Layer 4: frameworks

Web framework, database driver or ORM, migrations, JWT and hashing libraries, configuration from
environment variables, and the `CompositionRoot` that wires ports to adapters and wraps use cases with
their aspects.

## 7. HTTP contract

| Method | Path | Auth | Use case | Success |
|---|---|---|---|---|
| `POST` | `/auth/register` | none | `RegisterUser` | `201` token + user |
| `POST` | `/auth/login` | none | `Login` | `200` token + user |
| `GET` | `/games/{gameId}/progress` | user | `GetProgress` | `200` progress |
| `PUT` | `/games/{gameId}/progress` | user | `SyncProgress` | `200` merged progress |
| `GET` | `/games/{gameId}/leaderboard?level={number}&limit={n}` | user | `GetLeaderboard` | `200` entries |
| `GET` | `/games/{gameId}/levels` | none | `GetLevels` | `200` levels |
| `PUT` | `/games/{gameId}/levels/{number}` | admin | `UpsertLevel` | `200` or `201` level |

Plus the Swagger UI and the OpenAPI document. `gameId` is in the path of everything that depends on it,
so it can never be forgotten.

**Progress** (request and response of `PUT /games/hexa-fit/progress` share the shape):

```json
{
  "highScore": 0,
  "levels": [
    { "number": 1, "stars": 3, "bestScore": 1200, "completed": true },
    { "number": 2, "stars": 1, "bestScore": 640,  "completed": true }
  ]
}
```

**Level**: one schema for every game, with the game-specific part under `spec`. The `spec` fields mirror
the client's `FitLevel` and `SortLevel`; the exact shapes below are only illustrative.

```json
{
  "gameId": "hexa-fit",
  "number": 3,
  "version": 1,
  "difficulty": "easy",
  "spec": {
    "figure": [[0, 0], [1, 0], [0, 1], [1, 1]],
    "pieces": [{ "id": "A", "cells": [[0, 0], [1, 0]] }, { "id": "B", "cells": [[0, 0], [1, 0]] }],
    "starThresholds": { "moves": [2, 4, 6] }
  }
}
```

```json
{
  "gameId": "hexa-sort",
  "number": 1,
  "version": 1,
  "difficulty": "easy",
  "spec": { "goalTarget": 3, "colors": ["red", "blue", "green"], "seed": 42 }
}
```

**Errors**: one body shape for every failure.

```json
{ "error": { "code": "VALIDATION_ERROR", "message": "stars must be between 0 and 3", "details": [] } }
```

| Domain error | Status | Typical cause |
|---|---|---|
| `ValidationError` | `400` | malformed body, stars out of range, bad level `spec` |
| `InvalidCredentials` | `401` | wrong email or password, missing or expired token |
| `Forbidden` | `403` | a player calling an admin endpoint |
| `NotFound` | `404` | unknown `gameId` or level |
| `Conflict` | `409` | email or username already registered |
| anything else | `500` | unexpected; logged, never leaked in the body |

## 8. Data model

Five tables.

| Table | Main fields | Notes |
|---|---|---|
| `users` | `id`, `username`, `email`, `password_hash`, `role`, `created_at` | `username` and `email` unique |
| `games` | `id` (`hexa-lines`, `hexa-fit`…), `name`, `progress_model` | Seed data. A fifth variant is a new row |
| `player_progress` | `id`, `user_id`, `game_id`, `high_score`, `updated_at` | Unique on (`user_id`, `game_id`) |
| `level_records` | `id`, `progress_id`, `level_id`, `stars`, `best_score`, `completed` | Unique on (`progress_id`, `level_id`) |
| `levels` | `id`, `game_id`, `number`, `version`, `difficulty`, `common_json`, `specific_json` | Unique on (`game_id`, `number`) |

There is no leaderboard table. The ranking is a query:

- **By game:** `player_progress` filtered by `game_id`, ordered by `high_score` descending.
- **By level:** `level_records` joined to `player_progress`, filtered by game and level, ordered by
  `best_score` descending.

Ties go to the earlier `updated_at`. Storing a ranking separately would duplicate scores that already
live in these two tables.

## 9. Patterns

Fewer patterns arise naturally here than in the game. The ones that would have real code behind them:

| Pattern | Category | Where | Used by |
|---|---|---|---|
| Strategy | Behavioural | `ILevelSpecValidator`, chosen by `gameId` | `UpsertLevelUseCase` |
| Decorator | Structural | `UseCaseDecorator` and the aspects | `CompositionRoot` |
| Adapter | Structural | Repository implementations, `JwtTokenService`, `PasswordHasher` | Use cases |
| Singleton by injection | Creational | Database connection pool, configuration | `CompositionRoot` |

Repository and DTO are also used, though they are not GoF patterns. Nothing else was added just to
raise the count.

## 10. SOLID

| Principle | Example |
|---|---|
| SRP | `PlayerProgress` owns the merge rule, `SyncProgressUseCase` the steps, `ProgressController` the HTTP translation, the repository the SQL |
| OCP | A variant with levels is one new `ILevelSpecValidator` and one registration. No use case or controller is modified |
| LSP | Any `IProgressRepository` works for the use cases: the real one in production, an in-memory one in tests |
| ISP | Ranking reads go through `ILeaderboardQuery`, separate from `IProgressRepository`. `IPasswordHasher` and `ITokenService` are two ports, not one "security service" |
| DIP | `LoginUseCase` depends on `ITokenService` and `IPasswordHasher`, not on a JWT or hashing library |

## 11. Adding a fifth variant, and testing

**A fifth variant** on the backend is one row in `games`. If it has levels, it is also one
`ILevelSpecValidator` class and its registration in the `CompositionRoot`. No endpoint, use case, table or
migration changes.

**Testing:**

- **Unit, domain:** `PlayerProgress.merge()` — stars never go down, merging is order-independent, merging
  twice changes nothing.
- **Unit, use cases:** with in-memory fakes for every port. Wrong password, duplicate email, unknown
  `gameId`, level record for a level of another game.
- **Unit, aspects:** `AuthorizationDecorator` rejects a player and lets an admin through.
- **Integration:** HTTP endpoints against an in-memory database, including filtering by `gameId` (progress
  and rankings of one game never show up in another).
- **Contract (optional):** a consumer-driven contract with the game client for progress and levels.

## 12. Open questions for the team

These are the places where this approach took a side or could not. Each one needs a team decision.

| Question | What this approach assumes | Alternative and its cost |
|---|---|---|
| Can the server trust scores? | Yes, within range checks. It does not run the engine, so it cannot verify a score | Replaying moves on the server: real verification, but the engine would have to exist on both sides |
| What ranks a level-based game "by game"? | `high_score`, without saying what it means for Hexa Fit and Hexa Sort | Total stars, or the sum of best scores, computed by the server on each sync |
| Who may update levels? | A `role` column on `users` and an `admin` claim in the token | A separate admin API key: no role in the data model, one more secret to manage |
| Does the server check level `spec`? | Yes, with one validator per `gameId` | Store it as opaque JSON and let the client validate: less backend code, but a broken level reaches every device |
| Is `common_json` needed? | Kept from the early data model | Drop it: `number`, `version` and `difficulty` are already columns |
| Is `GET /progress` needed? | Yes, so the client's remote repository can load and save like the local one | Only `PUT`, which already returns the merged progress |
| Does a higher score always win? | Yes, in every variant | A per-game direction, if some variant ranks by fewest moves or shortest time |
| Do the pattern requirements apply to this repository on its own? | Not assumed either way | If they do, section 9 is short and needs a second look |
| Token lifetime and refresh | Not addressed | — |
| Which stack and which database? | None. Nothing here depends on it | — |

Still to do once the team has reviewed this: choose the stack, draw the layer and data diagrams, and
replace this file with the real architecture documentation.
