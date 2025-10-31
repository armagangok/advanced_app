## Riverpod Implementation Roadmap (Up-to-date, recommended practices)

This guide outlines a practical, modern Riverpod setup for production Flutter apps, with emphasis on offline-first architecture, testability, and performance.

### 0) Versions and Packages
- flutter_riverpod: latest stable (v3+)
- riverpod_annotation + riverpod_generator + build_runner (codegen for Notifier/AsyncNotifier)
- riverpod_lint (enable recommended lints)

### 1) Project setup
- Add dependencies in pubspec and run codegen:
  - dependencies: flutter_riverpod, riverpod_annotation
  - dev_dependencies: riverpod_generator, build_runner, riverpod_lint
- Wrap app root with ProviderScope (optionally add observers for logging).
- Create a `lint` configuration enabling riverpod_lint rules (select, avoid context misuse, etc.).

### 2) Architecture principles
- Layered by feature: presentation (widgets) → application/services (use-cases) → domain (entities, value objects) → infrastructure (repositories, API/DB).
- Providers model dependencies, not UI trees. Prefer provider composition over single “god” providers.
- Keep widgets declarative: no business logic inside widgets; use providers/notifiers for side-effects.

### 3) Provider types (modern guidance)
- Provider: constant/read-only dependencies (e.g., HTTP client, config).
- StateProvider: trivial mutable primitives (rare in complex features).
- Notifier/AsyncNotifier (+ codegen): primary choice for feature state and async flows.
- FutureProvider/StreamProvider: quick wiring to async but prefer AsyncNotifier for cohesive logic.
- Families: parameterize state (e.g., noteProvider(noteId)).
- autoDispose: default for UI-scoped state; keepAlive selectively for caches.

#### 3.1) State modeling guidelines (when to create a class?)
- Simple/primitive state (counter, toggle): use primitives via `StateProvider` or fields inside a feature Notifier; no extra class.
- Async data screens: start with `AsyncValue<T>` alone. If you need extra UI flags (filters, selection), upgrade to a small immutable state class that wraps `AsyncValue<T>`.
- Medium/complex screens (list + filters + error + pagination): create an immutable state class (prefer freezed) with `copyWith` and equality.
- Multi-phase flows (CRUD, wizards): model as sealed unions (freezed) like `Initial | Loading | Data | Empty | Error | Submitting`.
- Performance: split large state into multiple providers; in widgets use `select` to watch only changed fields.

### 4) Codegen-first approach (recommended)
- Use riverpod_annotation to declare providers:
  - @riverpod for Provider
  - @riverpod class X extends Notifier<T>
  - @riverpod class X extends AsyncNotifier<T>
- Benefits: safer ref access, generated provider names, fewer manual mistakes, easier refactoring.

### 5) Provider graph for offline-first Notes
- Core deps
  - apiClientProvider: HTTP client with interceptors/retry
  - databaseProvider: Drift/Hive instance
  - noteRepositoryProvider: composes API + DB with merge policy
- Feature state
  - notesListProvider (AsyncNotifier<List<Note>>): reads repo, exposes paging/filter
  - noteProvider.family (AsyncNotifier<Note?>): single note
  - syncOrchestratorProvider (Notifier/AsyncNotifier): schedules background sync, conflict resolution, ref.invalidate on changes
- UI
  - Widgets watch scoped providers; `select` to avoid broad rebuilds
  - Reactions via `ref.listen` for one-off effects (snackbars, navigation)

### 6) Concurrency, caching, lifecycle
- Prefer AsyncValue for async state; use `AsyncValue.guard` to capture errors.
- Use autoDispose for view-scoped providers; selectively keepAlive for caches.
- Tune lifetimes: cacheTime and disposeDelay (when available in your version) to reduce churn.
- Cancel stale requests on param change using families + autoDispose.
- Use `ref.onDispose` for cleanup, `ref.invalidate` to force refresh (e.g., after mutations).

### 7) Performance practices
- Minimize rebuilds: watch the smallest provider possible; prefer `select`.
- Scope providers near usage using ProviderScope overrides at subtree.
- Avoid heavy work on UI thread; delegate to isolates/services, then update provider.
- Use DevTools: track rebuild counts and frame times; restructure providers if needed.

### 8) Side-effects & reactions
- Keep side-effects in services/notifiers, not inside build.
- Use `ref.listen(provider, (prev, next) { ... })` for one-shot effects.
- Navigation: expose intents from notifier; perform navigation in UI when state changes.

### 9) Testing strategy
- Unit-test notifiers/services with ProviderContainer; override dependencies.
- Widget tests with ProviderScope(overrides: [...]) to inject fakes.
- Integration: test offline→online flows by toggling fake network layer and verifying merges.
- Golden tests for key screens; ensure loading/error/data paths with AsyncValue.

### 10) CI integration
- Enforce riverpod_lint; fail on violations.
- Run build_runner in CI (dart run build_runner build --delete-conflicting-outputs).
- Add test coverage gates for notifiers and repositories.

### 11) Migration tips (if coming from Bloc/Provider)
- Start per-feature: replace one slice with AsyncNotifier first.
- Map Events→methods, States→AsyncValue/data.
- Replace global singletons with providers; inject via overrides in tests.

### 12) Common pitfalls and fixes
- Watching too much: use `select` or split providers.
- Async in build: avoid side-effects; move to notifier or `ref.listen`.
- Leaky long-lived state: add autoDispose or explicit invalidation.
- Parameter changes not reflected: use family and/or invalidate.

### 13) Minimal example outline (pseudocode)
- @riverpod Database database(DatabaseRef ref) => openDatabase();
- @riverpod NoteRepository noteRepository(NoteRepositoryRef ref) => Repo(ref.watch(databaseProvider), ref.watch(apiClientProvider));
- @riverpod class NotesList extends AsyncNotifier<List<Note>> { build() => repo.fetchAllCached(); refresh() => state = AsyncValue.guard(repo.syncAndRead); }
- In UI: `ref.watch(notesListProvider)` and `ref.listen(notesListProvider, ...)` for effects

### 14) References (keep updated)
- Riverpod docs: `https://riverpod.dev`
- riverpod_lint: `https://pub.dev/packages/riverpod_lint`
- riverpod_generator: `https://pub.dev/packages/riverpod_generator`
- DevTools: `https://docs.flutter.dev/tools/devtools/overview`
- Drift: `https://drift.simonbinder.eu/`

### 15) Action checklist
- Enable ProviderScope and riverpod_lint
- Add codegen with riverpod_annotation/generator
- Model provider graph (deps → repos → notifiers)
- Implement AsyncNotifiers for list/detail + sync orchestrator
- Add tests with overrides; measure rebuilds; tune autoDispose/keepAlive


