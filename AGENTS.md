# VetHub — Architecture

VetHub is a veterinary clinic management application: a Spring Boot REST API and a SvelteKit web client for managing owners, pets, vets, and visits. The two sub-projects are independent and communicate over HTTP/JSON, with a generated OpenAPI spec as the contract between them.

```
┌──────────────────┐    HTTP/JSON     ┌──────────────────┐
│  client/         │ ───────────────► │  server/         │
│  SvelteKit SPA   │ ◄─────────────── │  Spring Boot API │
└──────────────────┘   /api/v1/*      └────────┬─────────┘
                                               │ JDBC
                                      ┌────────▼─────────┐
                                      │  H2 (in-memory)  │
                                      │  + Liquibase     │
                                      └──────────────────┘
```

The OpenAPI JSON emitted by the server is downloaded and turned into TypeScript types the client consumes — schema drift between the two is therefore a build-time error, not a runtime one.

---

## 1. Tech stack

### Server (`server/`)
| Concern             | Choice                                                            |
| ------------------- | ----------------------------------------------------------------- |
| Language / runtime  | Java 25 (Temurin), virtual threads enabled                        |
| Framework           | Spring Boot 4 (web-mvc), Spring Security                          |
| Build               | Gradle (Kotlin DSL)                                               |
| Persistence         | Spring Data JPA + Hibernate, **H2 in-memory**                     |
| Schema migrations   | Liquibase (XML changesets, `prd` / `tst` contexts)                |
| DTO mapping         | MapStruct (`SharedMapperConfig` from `jframe`)                    |
| Boilerplate         | Lombok                                                            |
| Validation          | `io.github.jframe.validation` (fluent) + JPA column constraints   |
| API docs            | springdoc-openapi (webmvc-ui)                                     |
| Quality gates       | Spotless, Checkstyle, PMD, SpotBugs, CodeNarc, `-Werror`          |

### Client (`client/`)
| Concern        | Choice                                                  |
| -------------- | ------------------------------------------------------- |
| Runtime / pkg  | Bun 1.3                                                 |
| Framework      | SvelteKit 2 / Svelte 5 (**runes**, not legacy `$:`)     |
| Build / dev    | Vite                                                    |
| Styling        | Tailwind CSS v4 (PostCSS)                               |
| Components     | shadcn-svelte + bits-ui                                 |
| HTTP           | `openapi-fetch` typed against generated `api.d.ts`      |
| Codegen        | `openapi-typescript` from server's OpenAPI JSON         |

### Tooling
- Tool versions pinned in `mise.toml` (Java Temurin 25, Bun 1.3.0, Node 22.20.0). Run `mise install` once at repo root.
- Helper scripts in `scripts/`: `openapi-sync.sh`, `common.sh`.

---

## 2. Project structure

```
vethub/
├── server/                          # Spring Boot REST API
│   ├── build.gradle.kts
│   ├── gradle.properties            # artifactName = ilionx-pet-store
│   └── src/
│       ├── main/java/dev/ilionx/workshop/
│       │   ├── api/                 # one package per domain (see §3)
│       │   │   ├── Paths.java       # centralized URL constants
│       │   │   ├── owner/
│       │   │   ├── pet/
│       │   │   ├── vet/
│       │   │   └── visit/
│       │   └── common/              # cross-cutting
│       │       ├── config/          # Spring config + properties
│       │       ├── exception/       # ApiErrorCode + handlers
│       │       └── security/filter/
│       ├── main/resources/
│       │   ├── application.yml
│       │   ├── application-dev.yml
│       │   └── db/changelog/        # Liquibase
│       ├── quality/config/          # checkstyle/pmd/spotbugs/codenarc/spotless
│       └── test/                    # UnitTest / IntegrationTest base classes
│
├── client/                          # SvelteKit frontend
│   ├── package.json
│   └── src/
│       ├── routes/                  # file-based routing
│       │   ├── owners/
│       │   ├── vets/
│       │   └── visits/
│       └── lib/
│           └── types/api.d.ts       # GENERATED — do not edit
│
└── scripts/
    ├── openapi-sync.sh              # boot server, dump spec, gen types, stop server
    └── common.sh
```

`server/openapi.json` is generated and gitignored; it is the artifact passed from server to client.

---

## 3. Server architecture conventions

### Domain packages are uniform

Each domain (`owner`, `pet`, `vet`, `visit`) has the **same internal structure** — once you know one, you know all of them:

```
<domain>/
├── controller/           # REST endpoints, paths from Paths.java
├── service/              # business logic, @Transactional boundaries
├── repository/           # Spring Data JpaRepository
└── model/
    ├── <Entity>.java     # JPA entity
    ├── request/          # inbound DTOs (Create*, Update*)
    ├── response/         # outbound DTOs
    ├── mapper/           # MapStruct entity → response
    └── validator/        # request validators (jframe)
```

Adding a new domain is mechanical: copy the layout, register routes in `Paths.java`, write a Liquibase changeset.

### Layered request flow

```
HTTP  ─►  Controller  ─►  Service  ─►  Repository  ─►  H2
                            │              │
                            │              └─ Hibernate / JPA
                            ├─ Validator (write ops, where present)
                            └─ Mapper (entity → response DTO)
```

Conventions enforced by code review and quality gates:
- **Controllers are thin.** They validate, delegate, map, return `ResponseEntity` with explicit status. Routes use constants from `Paths.java`; no inline URL strings.
- **Services own the transaction.** `@Transactional(readOnly = true)` on queries, `@Transactional` on writes. Services work with entities, never DTOs leaking outward.
- **Entities stay behind the mapper.** They are never serialized to clients directly.
- **Constructor injection only**, via Lombok `@RequiredArgsConstructor` over `final` fields.
- **Errors are typed.** Throw `DataNotFoundException(ApiErrorCode.X)`; the global handler maps to HTTP status + JSON body. No raw 500s for expected paths.

### DTO mapping — three boundaries, three mechanisms

| Direction              | Mechanism                                                           |
| ---------------------- | ------------------------------------------------------------------- |
| JSON → request DTO     | Jackson, driven by Lombok `@Data` POJOs annotated with `@Schema`    |
| Request DTO → entity   | **Manual field copy in the service** (so it can resolve associations via repositories) |
| Entity → response DTO  | **MapStruct** abstract `@Mapper(config = SharedMapperConfig.class)` |

MapStruct generates `*MapperImpl` at compile time. Lombok runs first so generated getters are visible. No reflection at runtime.

### Validation — two layers

1. **Request validators** (`*/model/validator/*Validator.java`) — fluent rules (`result.rejectField(...).whenNull(...).orWhen(...)`) called explicitly from the controller before invoking the service. Throws `ValidationException` → 400.
2. **JPA column constraints** (`@Column(nullable = false)`) — last-line DB safety net. If you only have this layer (e.g. `Pet` has no validator), the failure surfaces as a 500 from a `DataIntegrityViolationException`. Prefer adding a validator.

### Schema management — Liquibase, not Hibernate DDL

- Master file: `server/src/main/resources/db/changelog/db.changelog-master.yaml` does `includeAll: db/changelog/changesets`. Files run in filename order — hence the `YYYYMMDDhhmm-` prefix.
- **Contexts** (`application.yml`):
  - `prd` — schema and production reference data (vet/petType IDs 1–6, specialty IDs 1–3 are reserved seeds).
  - `tst` — demo data for integration tests, opted in per test.
- Hibernate DDL generation is **off**. `CamelCaseToUnderscoresNamingStrategy` only maps `firstName` → `first_name` against the columns Liquibase already created.
- Dev profile recreates the in-memory DB on each boot, so changesets effectively re-run from scratch in dev.

### API documentation — springdoc

Spec is reflected from controllers + DTO `@Schema` annotations at boot (`pre-loading-enabled: true`). No hand-written YAML. Endpoints (under `server.servlet.context-path: /api`):

- `GET /api/v1/public/docs` → OpenAPI 3 JSON
- `GET /api/v1/public/docs/openapi.html` → Swagger UI
- `GET /api/v1/public/actuator/health` → health

`/public/*` is whitelisted from authentication; everything else requires basic auth (dev: `user` / `password`).

### Server quality gate

`JavaCompile` `dependsOn("spotlessApply")` (`build.gradle.kts:224`) — Spotless formats on every compile. Compiler flag `-Werror` — every warning fails the build. Checkstyle, PMD, SpotBugs, CodeNarc all run under `./gradlew check`. Configs in `server/src/quality/config/`.

---

## 4. Client architecture conventions

- **Routing is file-based** under `src/routes/` (`owners/`, `vets/`, `visits/`, plus root `+layout.svelte` / `+page.svelte`).
- **API calls go through `openapi-fetch`** typed against `src/lib/types/api.d.ts`. The generated types are the single source of truth — if the server changes a payload, the client fails type-check until regenerated.
- **`api.d.ts` is a generated artifact.** Never edited by hand.
- **Per-domain API controllers** live in `src/lib/api/<domain>/<Domain>Controller.ts` (e.g. `OwnerController.ts`, `VetController.ts`). They are thin wrappers around the shared `client` from `src/lib/api/client.ts`, exporting plain async functions (`getOwners`, `createOwner`, …). DTO type aliases are re-exported from `src/lib/api/models.ts`.
- **Client error convention:** every call destructures `{ data, error }` from `openapi-fetch` and does `if (error) throw error;` — errors propagate as exceptions, no Result/Either wrapper. A global middleware in `client.ts` logs non-2xx responses to the console.
- **Svelte 5 runes** (`$state`, `$derived`, `$effect`) — not the legacy `$:` reactive syntax.
- **shadcn-svelte + bits-ui** for component primitives, Tailwind v4 utility classes for styling.

---

## 5. How the layers connect

### Runtime: client → server

1. Browser loads SvelteKit app from Vite dev server (`http://localhost:5173`).
2. UI calls `openapi-fetch` client → `http://localhost:8080/api/v1/...`.
3. CORS allows it via `CORS_ALLOWED_ORIGIN` (default `http://localhost:*`).
4. Spring Security checks basic auth (dev creds `user` / `password`).
5. Controller → Validator → Service (`@Transactional`) → Repository → H2.
6. Entity → MapStruct → response DTO → Jackson → JSON.

### Build-time: server → client (the contract bridge)

```
springdoc emits OpenAPI    ──►   GET /api/v1/public/docs
                                    │  (scripts/openapi-sync.sh)
                                    ▼
                          server/openapi.json  (gitignored)
                                    │  (openapi-typescript)
                                    ▼
                       client/src/lib/types/api.d.ts
                                    │
                                    ▼
                         openapi-fetch (typed calls)
```

Run `bun run sync:api` (or `bash scripts/openapi-sync.sh`) after **any** server API change. The script stops Gradle daemons, starts a fresh `bootRun`, downloads the spec, regenerates types, and stops the daemon again — don't run it while you have a `bootRun` you want to keep.

---

## 6. Developer workflow cheatsheet

### Server (`server/`)
```bash
./gradlew bootRun           # dev server, H2 in-memory, dev profile
./gradlew build             # compile + test + quality
./gradlew test              # all tests
./gradlew test --tests "dev.ilionx.workshop.api.vet.controller.VetControllerTest"
./gradlew check             # quality only
./gradlew spotlessApply     # auto-format (also runs on every compile)
```

### Client (`client/`)
```bash
bun install
bun run dev                 # Vite dev server
bun run build               # production build
bun run check               # svelte-kit sync + svelte-check
bun run sync:api            # full OpenAPI codegen (boots server)
bun run generate:api        # regenerate types from existing server/openapi.json
```

### Test conventions (server)

Two abstract base classes in `dev.ilionx.workshop.support` — extend one of them, do **not** mix the two. They are plain `abstract class` bases, not annotations.

#### `extends UnitTest` — fast, isolated, no Spring context
Use for: services, validators, mappers, helpers, anything you can exercise with mocks.

- Annotated `@ExtendWith(MockitoExtension.class)` — declare `@Mock` / `@InjectMocks` directly.
- No Spring context, no DB. Tests run in milliseconds.
- Provides static factories for in-memory entities: `aValidOwner()`, `aValidPet()`, `aValidPetType()`, `aValidVisit()`, `aValidVet()`, `aValidSpecialty()` (`UnitTest.java:19`). All return populated entities with `id = 1`.
- Use these factories instead of `new Owner()` so test setup stays consistent across the suite.

#### `extends IntegrationTest` — full Spring context + MockMvc + real H2
Use for: controllers, end-to-end request/response flows, anything that needs the real persistence layer or Spring wiring.

- Inherits from `WebMvcConfigurator` — boots Spring with MockMvc (`webAppContextSetup`). **No running server required.**
- DB is cleaned in `@BeforeEach` and `@AfterEach` (`IntegrationTest.java:43-51`), but **seeded reference rows are preserved** by ID:
  - vets and pet-types with `id <= 6`
  - specialties with `id <= 3`
- Consequences for writing tests:
  - Don't assume IDs 1–6 (vet/petType) or 1–3 (specialty) are yours to overwrite — they belong to seed data.
  - New entities you create will get IDs ≥ 7 (or ≥ 4 for specialties) and will be wiped between tests.
  - Owners, pets, and visits have **no reserved IDs** — they are fully wiped.
- Two flavors of factory:
  - **Persistence factories** — `aSavedOwner()`, `aSavedPet(owner)`, `aSavedVisit(pet)` write through repositories and return managed entities (`IntegrationTest.java:73-100`).
  - **Request factories** — `aCreateOwnerRequest()`, `anUpdatePetRequest()`, `aCreateVisitRequestWithPet(petId)`, etc. (`IntegrationTest.java:105-210`) build valid DTOs for MockMvc calls.
- Tests need the `tst` Liquibase context to load seed data; that's already wired up by `WebMvcConfigurator`.

### Environment variables (all have working defaults)
| Variable                  | Default                   | Notes                            |
| ------------------------- | ------------------------- | -------------------------------- |
| `APPLICATION_URL`         | `http://localhost:5173`   | Frontend URL used for CORS       |
| `CORS_ALLOWED_ORIGIN`     | `http://localhost:*`      | CORS origin pattern              |
| `SPRING_PROFILES_ACTIVE`  | `dev`                     | `prd` for production behavior    |
| `SERVER_PORT`             | `8080`                    |                                  |

---

## 7. Gotchas worth knowing

- Gradle artifact name is `ilionx-pet-store` (from `gradle.properties`), not `vethub`.
- `server/openapi.json` is the spec location — not under `client/`.
- Spotless runs on every compile; if it reformats your code, that's expected.
- Some entities intentionally lack validation (e.g. `Pet.birthDate` allows future dates; `Owner.firstName/lastName` are not `@NotBlank`) — these are teaching gaps, not features.
- Spring Security is **on** in dev; unauthenticated calls to non-`/public/` endpoints return 401.
