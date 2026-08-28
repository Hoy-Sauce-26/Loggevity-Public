# Loggevity

A local-first weekly health tracker. You log seven kinds of activity; it scores
your week against mortality-risk curves derived from epidemiological hazard
ratios and shows how you are tracking.

No account, no backend, no network calls, no analytics or ad SDKs. Everything
lives in an encrypted SQLite database on the device, and the only way data
leaves is a JSON or CSV export that you initiate.

- **Flutter** 3.44.9 / **Dart** 3.12.2 (stable)
- **Targets**: iOS, Android (macOS builds but is not a supported target)
- **State**: Riverpod 3 · **Storage**: Drift + SQLCipher · **Charts**: fl_chart

---

## Quick start

```bash
flutter pub get
flutter test          # 335 tests, ~5s
flutter analyze       # must be clean
flutter run           # pick a device, or -d <id>
```

After changing anything in [`lib/data/database.dart`](lib/data/database.dart)
you must regenerate the Drift code, or the build will fail on missing symbols:

```bash
dart run build_runner build
```

`lib/data/database.g.dart` is generated and checked in. Never edit it by hand.

---

## Repository map

```
lib/
  main.dart                  App entry, theme (single seed colour)
  providers.dart             Riverpod graph — the wiring diagram for the app
  scoring/                   Pure Dart. No Flutter, no I/O, no clock.
    curves.dart              Categories, research weights, lookup grids
    config.dart              ScoringConfig — the parts the user may move
    interpolation.dart       Piecewise linear interpolation (clamped)
    calculator.dart          HealthScoreCalculator
    models.dart              WeeklyTotals, CategoryScore, ScoreResult
  data/                      Storage and domain services
    database.dart            Drift schema + DAOs  (edit → run build_runner)
    connection.dart          SQLCipher connection; the encryption seam
    database_key.dart        Key generation + keychain storage
    week.dart                WeekRange, local-date bucketing
    combine_latest.dart      Two drift streams → one, teardown-safe
    metrics_repository.dart  Reactive bridge: rows → scores
    week_sealer.dart         Week-boundary engine
    portability.dart         JSON/CSV serialisation (pure)
    backup_service.dart      File I/O + share/pick for import/export
  screens/                   DashboardPage, AnalyticsPage, SettingsPage
  widgets/                   Presentation
test/
  scoring/  data/  ui/       Mirrors lib/
```

The dependency direction is one-way: `scoring/` knows nothing about `data/`,
and `data/` knows nothing about `widgets/`.

---

## Documentation

- **[docs/architecture.md](docs/architecture.md)** — why the app is shaped
  this way: the scoring model, storage schema, encryption, weekly sealing,
  import/export rules, invariants. Read this before changing anything.
- **[docs/features.md](docs/features.md)** — where the code for each
  user-visible feature lives, entry point to storage.
- **[docs/privacy-policy.md](docs/privacy-policy.md)** — the published
  privacy policy, with a table of how each claim in it can be re-verified.
- **[docs/play-data-safety.md](docs/play-data-safety.md)** — Play Console
  Data Safety answers and the reasoning behind each one.
- **[docs/methodology.md](docs/methodology.md)** — where every number in the
  scoring model comes from, source by source, and how much confidence each
  deserves. The document to argue with.

---

## Build flavours

Debug and profile builds install **alongside** release builds rather than
replacing them, so a dev build cannot clobber real data:

| Build | Application ID | Display name |
| --- | --- | --- |
| debug / profile | `com.nttech.loggevity.dev` | Loggevity Dev |
| release | `com.nttech.loggevity` | Loggevity |

Android via `applicationIdSuffix` and `resValue` in
[`android/app/build.gradle.kts`](android/app/build.gradle.kts); iOS via
per-configuration `PRODUCT_BUNDLE_IDENTIFIER` and `APP_DISPLAY_NAME`, which
`Info.plist` reads through `$(APP_DISPLAY_NAME)`. Separate containers and
separate keychain entries — see [docs/architecture.md](docs/architecture.md)
for why the dev build cannot read the release build's database.

---

## Testing

```bash
flutter test                                   # everything
flutter test test/scoring/                     # pure model, fastest signal
flutter test test/ui/dashboard_test.dart -r compact
```

335 tests across `test/scoring/`, `test/data/`, `test/ui/`. See
[docs/architecture.md](docs/architecture.md#testing) for the gotchas that
cost people an hour the first time (Drift + widget test teardown, mainly).
