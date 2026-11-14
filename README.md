# [Project Name]

> Short description of the app — what it does and who it's for.
> See `PRD.md` for the full product specification.

<!-- Add screenshots here once available -->

## Documentation

This repository is documented for both humans and AI coding agents.

| Document | Purpose |
|---|---|
| `AGENTS.md` | **Start here if you are an AI agent.** Working rules, commands, git conventions, and definition of done |
| `PRD.md` | Product requirements: features, priorities, user flows |
| `ARCHITECTURE.md` | Clean architecture structure, layers, DI, routing, flavors |
| `DESIGN_SYSTEM.md` | Design tokens, theming, and shared `App*` components |
| `CODE_STYLE.md` | Naming, formatting, and coding conventions |
| `TESTING.md` | Testing strategy and how to run tests |
| `SECURITY.md` | Secrets, auth tokens, network, and release security rules |

## Tech Stack

- **Flutter** `3.47.9` (Dart `3.13.7`) — managed via [FVM](https://fvm.app), pinned in `.fvmrc`
- **State management**: `flutter_bloc` (Bloc only, no Cubit)
- **Routing**: `go_router`
- **Dependency injection**: `get_it` + `injectable`
- **Networking**: `dio`
- **Local storage**: `drift` (SQLite), `shared_preferences`, `flutter_secure_storage`
- **Logging**: `AppLogger` abstraction (console / Firebase Crashlytics / remote API)
- **Flavors**: `staging`, `prod`

## Prerequisites

- macOS, Linux, or Windows
- [FVM](https://fvm.app/documentation/getting-started/installation) installed
- `make` (macOS/Linux built-in; on Windows use Git Bash, [chocolatey](https://community.chocolatey.org/packages/make), or run the raw commands — every Make target in the docs lists its raw equivalent)

## Setup

```bash
# 1. Clone
git clone <repo-url>
cd <project-name>

# 2. Install the pinned Flutter version
fvm install
fvm flutter --version   # should print 3.47.9

# 3. Install dependencies
fvm flutter pub get

# 4. Set up environment config
cp .env.example .env.staging   # fill in staging values
cp .env.example .env.prod      # fill in production values
# NOTE: .env files are gitignored and must never be committed. See SECURITY.md.

# 5. Run code generation (injectable, drift)
fvm dart run build_runner build --delete-conflicting-outputs

# 6. Run the app (staging flavor)
fvm flutter run --flavor staging -t lib/main_staging.dart
```

## Common Commands

All commands are wrapped in a **Makefile** for convenience on macOS/Linux (`make <target>`). The Makefile is created when the actual Flutter project starts — it does not exist in this docs starter kit. Until then, and on Windows, use the raw commands below — they are the source of truth. The required Make targets are exactly the ones listed in this table.

| Task | Make | Raw command |
|---|---|---|
| Install deps | `make get` | `fvm flutter pub get` |
| Codegen | `make gen` | `fvm dart run build_runner build --delete-conflicting-outputs` |
| Format | `make format` | `fvm dart format .` |
| Analyze | `make analyze` | `fvm flutter analyze` |
| All tests | `make test` | `fvm flutter test` |
| Tests per feature | `make test-feature f=<feature>` | `fvm flutter test test/features/<feature>/` |
| Coverage | `make test-coverage` | `fvm flutter test --coverage` |
| Integration test | `make test-integration` | `fvm flutter test integration_test/ --flavor staging -t lib/main_staging.dart` |
| Run staging | `make run-staging` | `fvm flutter run --flavor staging -t lib/main_staging.dart` |
| Run production | `make run-prod` | `fvm flutter run --flavor prod -t lib/main_prod.dart` |

## Project Structure

```
lib/
├── core/          # config, design_system, di, error, l10n, logger, network, router, utils
├── shared/        # widgets & models shared across features
├── features/      # one folder per feature, each with data / domain / presentation
├── main_staging.dart
└── main_prod.dart
test/              # mirrors lib/
integration_test/  # end-to-end flows
```

Full structure and rules: `ARCHITECTURE.md`.

## Contributing

1. Branch from `development` using `<type>/<short-description>` (e.g. `feature/login-email`).
2. Follow the rules in `AGENTS.md` and `CODE_STYLE.md`.
3. Make sure `make analyze` and `make test` are green.
4. Commit using [Conventional Commits](https://www.conventionalcommits.org/) in English (e.g. `feat(auth): add email login`).
5. Open a pull request to `development`.

## License

<!-- Add license, e.g. MIT -->
