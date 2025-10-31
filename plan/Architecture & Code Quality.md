## Architecture & Code Quality Guide

Practical standards to design scalable Flutter apps with a Riverpod-first approach, offline-first needs, and production quality. Use this as a baseline for features, reviews, and releases.

### Versions & Tooling
- Flutter/Dart: keep to current stable; lock CI to the same channel
- Lints: `flutter_lints` + `riverpod_lint` (treat as errors in CI)
- Formatting: `dart format --fix` in pre-commit; import sort enforced
- Static checks: analyzer, unused code, dead code removal

### Project Structure
- Feature-first modules with clear layers:
  - Presentation (widgets)
  - Application/Services (use-cases, orchestration)
  - Domain (entities, value objects, policies)
  - Data/Infrastructure (repos, API/DB, adapters)
- `core/` for shared primitives: result types, errors, http, logging, env
- Avoid cross-feature imports; communicate via services/repositories

### Clean Architecture in Flutter
- Dependencies flow inward: UI → services → domain → abstractions → infra
- Domain models are UI-agnostic; enforce invariants at construction
- Repositories expose domain types; map DTOs at the boundary
- Keep adapters thin; centralize mapping and validation

### Dependency Injection (Riverpod-first)
- Providers for clients (HTTP, DB), repositories, services, and feature state
- Families for parameterized state; `autoDispose` for UI-scoped providers
- `keepAlive` selectively for caches; prefer explicit invalidation after writes
- Provider graphs documented per feature; override in tests and per-env

### Error Handling & Resilience
- Prefer domain-safe errors (sealed classes / Result<E,T>) at boundaries
- Use `AsyncValue.guard` in async flows; surface actionable messages to UI
- Networking: retries with backoff, idempotent writes, cancellation support
- Offline-first: write-through cache; conflict resolution strategy (last-write-wins, vector clock, or user-merge)

### Logging, Analytics, Observability
- Logger abstraction with redaction; avoid logging secrets/PII
- Crash reporting (Sentry/Crashlytics) + breadcrumbs for critical flows
- Performance budgets: startup P95, frame build/raster times, jank budget

### Code Quality Standards
- Naming: descriptive, no abbreviations; domain language in domain layer
- Immutability by default; prefer small, focused modules
- Forbidden patterns: global singletons, logic in widgets, “god” providers
- Documentation: per-feature README with provider graph diagram and data flow

### Testing Strategy
- Unit: domain/services/notifiers with `ProviderContainer` overrides
- Widget: screens with `ProviderScope(overrides: [...])`, golden tests
- Integration: offline→online sync paths; deterministic fixtures
- Contract tests: DTO↔entity mappers, API schemas

### Performance Guidelines
- Minimize rebuilds: use `select`, split providers, scope ProviderScope
- Heavy work off UI thread: isolates for parsing/crypto/image IO
- Lists: prefer slivers, itemExtent, caching, debounce inputs
- Startup: lazy init non-critical providers; consider SkSL warm-up on Android

### Security & Privacy
- Secrets in env/CI, not in code; secure storage for tokens
- TLS/pinning when applicable; input validation at boundaries
- PII minimization; data export/delete flows (GDPR basics)

### i18n, a11y, theming
- Localization via ARB; test text scaling; right-to-left readiness
- Contrast and focus order; keyboard/mouse support for desktop/web
- Theming centralized; no inline hard-coded colors/metrics

### Release Engineering
- Versioning (semver), CHANGELOG, release notes automation
- Feature flags/remote config for risky changes
- CI gates: analyze, format, tests, build, size/perf budgets

### Architecture Review Checklist (per feature)
- [ ] Clear boundaries: presentation / services / domain / data separated
- [ ] Provider graph documented; DI via providers, no global singletons
- [ ] Domain entities immutable with invariants
- [ ] Repositories expose domain types; DTO mapping isolated
- [ ] Error strategy defined (AsyncValue/Result) and user messages sensible
- [ ] Offline behavior defined (cache strategy, conflict resolution)
- [ ] Tests: unit + widget; provider overrides used; fixtures deterministic
- [ ] Performance: rebuilds measured; long tasks off UI thread
- [ ] Security: secrets, PII, permissions handled

### Code Review Checklist (per PR)
- [ ] Small, focused changes; clear title/description
- [ ] Lints pass; formatted; no dead code
- [ ] Names precise and consistent; no business logic in widgets
- [ ] Providers scoped; `select` used where appropriate
- [ ] Errors surfaced appropriately; no swallowed exceptions
- [ ] Tests updated/added; golden diffs reviewed when applicable
- [ ] Docs/READMEs updated; ADR added for significant decisions

### Metrics & SLAs
- Crash-free sessions ≥ 99.8%
- Startup cold P95 reduced and tracked per release
- Key screens frame build < 8ms; raster < 8ms on target devices
- Coverage targets on critical modules (services, notifiers, mappers)

### Migration Notes
- Move from ad-hoc singletons to providers with overrides
- Extract mapping and validation to adapters; keep repositories thin
- Split monolith state into multiple providers; adopt `select` in UI


