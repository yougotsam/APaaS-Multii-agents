---
name: codebase-cartography
description: "Map, analyze, and document unfamiliar codebases to produce navigable architecture maps, dependency graphs, and onboarding guides. TRIGGER when: user asks to understand a codebase, map a repository, create architecture documentation, analyze code dependencies, onboard onto a new project, or asks 'how does this codebase work'. Also triggers on: 'map this repo', 'architecture overview', 'codebase tour', 'dependency analysis'. SKIP when: user is asking about a single file, writing new code, or debugging a specific error."
license: MIT
compatibility: "Requires filesystem read access. Works best with git repositories. Supports Python, TypeScript/JavaScript, Go, Rust, Java, and Ruby codebases."
metadata:
  category: "analysis"
  complexity: "advanced"
  token-budget: "5000"
---

# Codebase Cartography

You are mapping an unfamiliar codebase to produce a navigable understanding of its architecture, patterns, and boundaries. Your output is a structured analysis that a senior engineer can use to become productive in the codebase within hours instead of days.

## Mapping Protocol

Execute these phases in order. Do not skip phases. Each phase builds on the previous.

### Phase 1: Terrain Survey (5 minutes)

Gather raw structural data before forming any opinions.

```bash
# 1. Repository vitals
git log --oneline -20
git shortlog -sn --no-merges | head -10
wc -l $(find . -name '*.py' -o -name '*.ts' -o -name '*.tsx' -o -name '*.go' -o -name '*.rs' -o -name '*.java' -o -name '*.rb' 2>/dev/null) 2>/dev/null | tail -1

# 2. Directory skeleton (depth 3, ignore noise)
find . -type d -not -path '*/node_modules/*' -not -path '*/.git/*' -not -path '*/venv/*' -not -path '*/__pycache__/*' -not -path '*/dist/*' -not -path '*/build/*' | head -80

# 3. Entry points and config
ls -la package.json pyproject.toml Cargo.toml go.mod Gemfile pom.xml build.gradle Makefile Dockerfile docker-compose* 2>/dev/null

# 4. Dependency manifest
cat package.json 2>/dev/null | head -50
cat pyproject.toml 2>/dev/null | head -50
cat requirements.txt 2>/dev/null

# 5. Test infrastructure
find . -name '*test*' -o -name '*spec*' | grep -v node_modules | head -20
```

### Phase 2: Boundary Detection

Identify the major architectural boundaries. Every codebase has 3-7 major subsystems. Find them.

**Signals to look for**:
- Top-level directories that map to domains (e.g., `auth/`, `billing/`, `api/`, `workers/`)
- Separate package.json/pyproject.toml files (monorepo boundaries)
- Docker services in docker-compose (service boundaries)
- Import clusters (files that import heavily from each other form a module)
- Interface files (types.ts, models.py, schemas/) that define contracts between boundaries

**Output format**:
```markdown
## Architectural Boundaries

| Boundary | Location | Responsibility | External Dependencies |
|----------|----------|---------------|----------------------|
| API Layer | src/api/ | HTTP routing, request validation, auth middleware | Express, Zod |
| Domain Core | src/domain/ | Business logic, no framework deps | None (pure) |
| Persistence | src/db/ | Postgres queries, migrations, connection pool | pg, knex |
| Workers | src/jobs/ | Background job processing, queue consumers | BullMQ, Redis |
| Shared | src/lib/ | Utilities consumed by all other boundaries | - |
```

### Phase 3: Data Flow Tracing

Trace the path of a request through the system. Pick the most common operation (usually the primary CRUD flow) and follow it from entry point to storage and back.

```markdown
## Primary Data Flow: [Operation Name]

1. **Entry**: `POST /api/v1/orders` --> `src/api/routes/orders.ts:L45`
2. **Validation**: Zod schema at `src/api/schemas/order.ts:L12` validates body
3. **Auth**: Middleware at `src/api/middleware/auth.ts:L8` checks JWT, attaches `req.user`
4. **Business Logic**: `src/domain/orders/createOrder.ts:L20` orchestrates:
   - Checks inventory via `src/domain/inventory/checkStock.ts`
   - Calculates pricing via `src/domain/pricing/calculate.ts`
   - Creates order record
5. **Persistence**: `src/db/repositories/orders.ts:L34` inserts into `orders` table
6. **Side Effects**: Publishes `order.created` event to BullMQ at `src/jobs/publishers/orderEvents.ts:L15`
7. **Response**: Returns 201 with order ID and status
```

### Phase 4: Pattern Catalog

Document the recurring patterns the codebase uses. These are the "rules" an engineer needs to know to write code that fits.

```markdown
## Codebase Patterns

### Error Handling
- Pattern: [throw custom errors / return Result types / error codes / try-catch-forward]
- Example: `src/domain/errors.ts` defines `AppError` base class
- Convention: Domain errors extend `AppError`, API layer maps to HTTP status codes

### Data Access
- Pattern: [Repository pattern / Active Record / raw queries / ORM]
- Example: `src/db/repositories/users.ts` wraps all SQL behind typed methods
- Convention: No raw SQL outside `src/db/`. All queries use parameterized inputs.

### Authentication
- Pattern: [JWT / session / API key / OAuth]
- Flow: Token issued at login, validated per-request by middleware, user attached to request context

### Testing
- Pattern: [unit + integration / mostly integration / E2E heavy]
- Convention: Test files colocated with source (`*.test.ts`) vs. separate `__tests__/` directory
- Fixtures: [factory pattern / hard-coded / database seeding]

### State Management (if frontend)
- Pattern: [Redux / Zustand / Context / server-state-only (React Query)]
- Convention: Where global state lives, how components access it

### Configuration
- Pattern: [env vars / config files / feature flags]
- Convention: How secrets are managed, how environments differ
```

### Phase 5: Risk Map

Identify the areas that are fragile, complex, or dangerous to modify.

```markdown
## Risk Map

### High-Risk Areas (modify with extreme caution)
| Location | Risk | Reason |
|----------|------|--------|
| src/db/migrations/ | Data loss | Schema changes affect production data |
| src/api/middleware/auth.ts | Security | Auth bypass = full compromise |
| src/domain/billing/ | Financial | Incorrect calculations = revenue loss |

### Complexity Hotspots (high cyclomatic complexity or deep coupling)
- `src/domain/orders/createOrder.ts` (180 lines, 12 dependencies, 4 external API calls)
- `src/api/routes/index.ts` (registers 47 routes, single point of failure for routing)

### Dead Code Candidates (potentially removable)
- `src/legacy/` directory (last modified 8 months ago, no imports found)
- `src/utils/deprecated.ts` (marked deprecated, still imported by 2 files)

### Missing Test Coverage
- `src/domain/billing/` has 0 test files (highest-risk area with no safety net)
- `src/jobs/` has integration tests but no unit tests for job handlers
```

### Phase 6: Deliverable Assembly

Produce the final cartography document:

```markdown
# Codebase Cartography: [Repository Name]

## Quick Facts
- **Language**: [primary language] with [secondary]
- **Framework**: [web framework, ORM, test runner]
- **Architecture**: [monolith / microservices / modular monolith / serverless]
- **Size**: [X files, Y KLOC, Z contributors]
- **Age**: [first commit date] - [last commit date]
- **Health signals**: [CI passing?, test coverage?, recent activity?]

## Architecture Map
[Boundary table from Phase 2]

## How a Request Flows
[Data flow trace from Phase 3]

## Patterns You Must Follow
[Pattern catalog from Phase 4]

## Where the Dragons Live
[Risk map from Phase 5]

## Getting Productive
1. Start by reading: [3-5 most important files, in order]
2. Run locally: [exact commands to get the system running]
3. Make your first change in: [safest area to start contributing]
4. Avoid touching: [areas that require deep context before modifying]
```

## Gotchas

- **Monorepos**: Map each package/service as a separate boundary. The root package.json is infrastructure, not architecture.
- **Generated code**: Identify and exclude generated files (protobuf outputs, GraphQL codegen, ORM migrations) from pattern analysis. They follow their generator's patterns, not the team's.
- **Dead imports**: An import graph is not a dependency graph. Verify that imported modules are actually called, not just imported and unused.
- **Test fixtures as documentation**: Test files often contain the best examples of how to use internal APIs. Read them before reading the implementation.

## Completion Criteria

- All 6 phases are completed with output for each
- Boundary table has 3-7 entries (not 1, not 20)
- At least one data flow is fully traced from entry to storage
- Risk map identifies at least 2 high-risk areas
- The "Getting Productive" section has concrete file paths and commands, not vague advice
