# AGENTS.md

## Repo structure

Two independent sub-projects; always run commands from the correct directory.

```
vethub/
├── server/   # Spring Boot 4 REST API (Java, Gradle)
├── client/   # SvelteKit 2 / Svelte 5 frontend (Bun, Vite)
└── scripts/  # openapi-sync.sh, common.sh
```

## Prerequisites

Tool versions are pinned in `mise.toml`. Run `mise install` from repo root before anything else.

| Tool | Version |
|------|---------|
| Java | Temurin 25 |
| Bun  | 1.3.0 |
| Node | 22.20.0 |

## Server commands (`server/`)

```bash
./gradlew bootRun           # start dev server (H2 in-memory, dev profile)
./gradlew build             # compile + test + quality checks
./gradlew test              # run all tests
./gradlew test --tests "dev.ilionx.workshop.api.vet.controller.VetControllerTest"  # single test class
./gradlew check             # all quality checks without building JAR
./gradlew spotlessApply     # auto-format (runs automatically on every compile anyway)
```

**Critical:** `JavaCompile` depends on `spotlessApply` — Spotless runs on every compile. Compilation uses `-Werror`; all warnings are errors.

Quality tools: Checkstyle, PMD, SpotBugs, CodeNarc, Spotless. Config in `server/src/quality/config/`.

Dev server endpoints:
- API: `http://localhost:8080/api/v1/`
- OpenAPI JSON: `http://localhost:8080/api/v1/public/docs`
- Swagger UI: `http://localhost:8080/api/v1/public/docs/openapi.html`
- Health: `http://localhost:8080/api/v1/public/actuator/health`

Spring Security is active; dev profile uses basic auth `user` / `password`.

## Client commands (`client/`)

```bash
bun install          # install deps
bun run dev          # Vite dev server
bun run build        # production build
bun run check        # svelte-check type-check
bun run sync:api     # full OpenAPI codegen (see below)
bun run generate:api # generate types only (requires openapi.json already downloaded)
```

## OpenAPI / codegen — required after any server API change

`client/src/lib/types/api.d.ts` and `openapi.json` are **generated artifacts — never edit manually**.

Full sync (starts server, downloads spec, generates types, stops daemons):
```bash
# from repo root
bash scripts/openapi-sync.sh
# OR from client/
bun run sync:api
```

Manual steps:
1. Start server: `./gradlew bootRun` in `server/`
2. `bun run download:api` in `client/` — saves spec to `client/src/lib/types/openapi.json`
3. `bun run generate:api` in `client/` — outputs `client/src/lib/types/api.d.ts`

## Server architecture conventions

- Each domain (`owner`, `pet`, `vet`, `visit`) has identical layout: `controller/`, `service/`, `repository/`, `model/` with sub-packages `request/`, `response/`, `mapper/`, `validator/`.
- **MapStruct** for entity↔DTO mapping (annotation-processed at compile time).
- **Lombok** for boilerplate. Add `@Data`/`@Builder` rather than writing getters/setters.
- API paths centralized in `Paths.java`.
- **Liquibase** manages schema; changesets in `server/src/main/resources/db/changelog/changesets/`. Use `prd` context for production data, `tst` context for test-only seed data. Dev profile drops and recreates DB on startup.
- H2 in-memory DB — no external database required.
- Gradle artifact name is `ilionx-pet-store` (from `gradle.properties`), not `vethub`.

## Test conventions

- `UnitTest` base class: Mockito extension + static factory methods for valid domain objects.
- `IntegrationTest` base class: MockMvc (`webAppContextSetup`), cleans DB in `@BeforeEach`/`@AfterEach` but **preserves seeded rows** (vet/petType IDs 1–6, specialty IDs 1–3). Do not assume those IDs are available to overwrite or that newly created entities will get those IDs.
- Integration tests do not require a running server; MockMvc handles it.

## Client architecture

- SvelteKit 2 / Svelte 5 (uses **runes** — not the legacy `$:` reactive syntax).
- Tailwind CSS v4 with PostCSS.
- **shadcn-svelte** + **bits-ui** component primitives.
- **openapi-fetch** for type-safe API calls via the generated `api.d.ts` types.
- File-based routing under `src/routes/`: `owners/`, `vets/`, `visits/`.

## Environment variables

All have working defaults for local dev; no `.env` file required.

| Variable | Default | Notes |
|----------|---------|-------|
| `APPLICATION_URL` | `http://localhost:5173` | Frontend URL used for CORS |
| `CORS_ALLOWED_ORIGIN` | `http://localhost:*` | CORS allowed origin pattern |
| `SPRING_PROFILES_ACTIVE` | `dev` | Switch to `prd` for production behavior |
| `SERVER_PORT` | `8080` | |
