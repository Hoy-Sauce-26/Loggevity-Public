# Architecture

Why Loggevity is built this way, and what breaks if you change it. Read this
before touching `lib/scoring/`, `lib/data/`, or the Trends charts.

## The few facts everything follows from

1. **No backend.** There is no server, no account, no sync. Every design
   decision about storage, migration, and export follows from data living
   only on the device.
2. **No background execution.** The app cannot run code at midnight to notice
   a week ended, or between launches at all. Anything that reacts to time
   passing must run on launch and catch up for however long the app was
   closed.
3. **The scoring model changes over the product's life** (new categories,
   reweighting, a moved socializing target), but past weeks must still be
   re-explainable under the model in force *then* or *now*, depending on the
   view. This is why raw values are stored and scores are derived on read.

## Dependency direction

```
lib/
  scoring/   — pure Dart, no Flutter, no I/O, no clock
  data/      — storage and domain services; knows nothing about widgets/
  screens/, widgets/ — presentation; depends on data/ and scoring/
```

One-way. `scoring/` cannot import Flutter or `dart:io` — `purity_test.dart`
fails the build if it does. This keeps the model testable without a widget
harness and reusable if the app ever needs a second frontend.

## The scoring model

[`lib/scoring/`](../lib/scoring/) and its tests are the *definitive* statement
of the model — there is no external workbook or spec to reconcile against.

### Curves are scores, not hazard ratios

Each category's curve is a **score** curve derived from hazard ratios, not the
hazard ratios themselves:

```
score = (1 - HR) / k * 10
```

where `k` is the maximum achievable risk reduction for that category. An HR of
1.00 ("no benefit") maps to a score of **0**, not 10.

| Category | k | Curve (weekly minutes → score) |
| --- | --- | --- |
| Moderate PA | 0.35 | (0,**0**) (75,2.286) (160,5.143) (250,6.286) (900,10) (10080,10) |
| Vigorous PA | 0.15 | (0,0) (100,8.667) (215,**10**) (900,6.667) |
| Resistance | 0.20 | (0,0) (15,4) (22,6.5) (45,**10**) (60,**10**) (80,8) (100,6.5) (140,0) (160,−2.5) (200,**−9**) |

- **Vigorous PA is a horseshoe** — CVD-mortality benefit peaks at 215 min/week
  (HR 0.85), then declines back to HR 0.90 by 900.
- **Resistance training goes negative.** Past 140 min/week it is a net harm,
  bottoming at −9 at 200 min; sleep can reach −10. **Negative sub-scores are
  intentional and never clamped to zero** — a composite can legitimately be
  negative, and the UI renders that rather than hiding it. The negative tail
  is the source's own claim: Momma 2022 finds benefit to about 140 min/week
  "with possible harm at progressively higher doses". Don't flatten it on the
  grounds that absence of benefit is not harm — the authors say harm.

The remaining four categories are linear to a weekly target, capped at 10:
flexibility 45 min, nature 120 min, socializing (user-set, default 14 h),
sleep (see below).

Out-of-range inputs **clamp** to the terminal value; curves are undefined
beyond their last anchor. Extrapolating the final segment instead would give
unbounded penalties — 300 min of resistance would score −25.25.

### Sleep

Raw hours are converted per night to "adjusted hours". Nights inside a 7–9h
band cost nothing; outside it, each raw hour of deviation costs adjusted
hours — **2× short, 3× long**:

```
A(h) = 7.5 − 2(7 − h)   if h < 7
     = 7.5              if 7 ≤ h ≤ 9
     = 7.5 − 3(h − 9)   if h > 9

S_sleep = Σ A(hᵢ) / 52.5 × 10
```

**The band is the source's own.** GeroScience 2025 defines short sleep as
under 7h (HR 1.14) and long as 9h or more (HR 1.34), leaving 7–9 as the range
carrying no measured penalty. An earlier version ran 7.5–9.0, which blessed
8–9 and penalised 7–7.5 against the very paper it cited.

**The asymmetry is the source's own too.** On the raw numbers an hour of
oversleep is worth about twice an hour of undersleep (RRR 25.4 vs 12.3, ratio
2.06). The shipped ratio is 1.5, discounted because the long arm is the one
contaminated by reverse causation — illness causes oversleeping at least as
much as the reverse. That discount lives in the slope rather than the sleep
weight because the weight would discount both tails equally and only one has
the problem. The short arm is untouched at 2×.

One artefact: because the long arm is steeper, a week of 14h nights scores
below a week of zero-hour nights. Seven nights of no sleep is not a state a
user can be in, so the model is not tuned for it.

**Store raw hours, never adjusted.** The reference week's seven nights total
58.25 raw hours, which compress to 52.5 adjusted. Storing 52.5 would be wrong
and would make the adjustment formula unrecoverable.

### Composite

```
composite % = Σ(subScore × weight) / (10 × Σweights) × 100
```

Under the research weights the denominator is 1020 — not a constant: a custom
weighting has its own denominator computed from the weights in force, so a
perfect week reads exactly 100% whatever the user has set.

**Every weight is `RRR × credibility`**, where RRR is the category's maximum
relative reduction in mortality in percentage points, and credibility
discounts that for how far the source sits from what the app actually
measures. The set therefore sums to roughly 100, and a weight reads as
"percentage points of mortality risk this category can buy you".

| Category | RRR | Credibility | Weight | Max points |
| --- | --- | --- | --- | --- |
| Moderate PA | 35 | 1.0 | 35 | 350 |
| Socializing | 26.2 | 0.9 | 24 | 240 |
| Sleep | 18.8 | 0.9 | 17 | 170 |
| Vigorous PA | 15 | 0.7 | 11 | 110 |
| Resistance | 17 | 0.33 | 6 | 60 |
| Nature | 15 | 0.33 | 5 | 50 |
| Flexibility | 12 | 0.33 | 4 | 40 |
| **Total** | | | **102** | **1020** |

**Vigorous PA is the one category scored on CVD mortality** rather than
all-cause — shape and magnitude both. The reverse-J past 215 min is the entire
reason vigorous is a separate category, and it exists only in the CVD data;
all-cause vigorous is monotonic to 900 min (HR 0.81). Sourcing the magnitude
from all-cause while drawing the CVD shape would claim a benefit the curve
then denies. The cost is one mixed endpoint in an otherwise all-cause
composite, and a 15% reduction in CVD mortality is not 15 points of all-cause
mortality — CVD is roughly a third of deaths. State that plainly rather than
papering over it.

`docs/methodology.md` derives every figure from its source. Two conversion
rules that the first version of this table got wrong, both of which silently
inflate a weight:

- **Everything is a risk *reduction*.** A source reporting "34% higher
  mortality" (HR 1.34) is a reduction of `1 − 1/1.34` = 25%, not 34.
- **Everything is a *risk* ratio.** Odds ratios overstate risk ratios when the
  outcome is common, and death over a long follow-up is very common. The
  social OR of 1.50 for survival is a risk reduction of 26%, not 50.

**Credibility is kept separate from the curve on purpose.** Folding the
discount into the curve's `k` would hide it where nobody can audit it. The
cost is that `points = RRR × 1000` only holds exactly where credibility is 1
and the endpoints match — so the composite is a weighted preference score
informed by mortality research, *not* a risk reduction, and should never be
described to a user as one.

> **Vigorous PA counts toward the denominator by design.** Including its
> weight makes vigorous activity required rather than bonus credit, and gives
> the composite a hard 100% ceiling. Dropping it from the divisor would turn
> vigorous into overflow credit and let a strong week exceed 100%. This is a
> product decision — don't "simplify" it away.

Weights are **relative** — only their ratios matter, so doubling all seven
changes nothing. Zeroing one removes that category from the score entirely,
numerator and denominator alike.

Two further caveats that belong in any user-facing description of the score:
every source is observational with heavy residual confounding, and summing
the categories assumes they are independent, which they are not. Someone who
exercises, sleeps well *and* has friends is a different person, not a sum of
three effects.

### What the user may change

Everything above is fixed except what lives on
[`ScoringConfig`](../lib/scoring/config.dart), threaded through every scoring
call rather than read from a global — two configs must be able to score the
same week side by side (this is how the settings-change re-seal and the
research/custom comparison both work).

| Setting | Default | Range |
| --- | --- | --- |
| Socializing target | 14 h/week | 0.5 – 112 |
| Socializing mode | Hours | Hours / Satisfaction |
| Category weights | Research | 0 – 1000 each, total > 0 |

**Why socializing and not the rest.** Company is strongly protective, but how
much of it a week needs is not the same number for an introvert and an
extrovert. The research figure is 21 h/week (`kSocialResearchTargetHours`);
the shipped default is 14 (`kSocialDefaultTargetHours`), because a target
nobody reaches stops being a target. Both constants stay in the code.

**Satisfaction mode** replaces the stopwatch with one question a week: *were
you satisfied with your social life?* Yes scores 10, no scores 0, and an
unanswered week also scores 0 — "not recorded" and "did not happen" are the
same thing everywhere else in the model. The answer never prorates. Hours
logged while the check-in is in force are kept, not scored, and count again on
switching back.

**Custom weights** sit behind an explicit warning, because they quietly
detach the score from the evidence behind it. They are stored as `null` until
the user overrides them — not as a copy of today's numbers — so a later
change to the published model still reaches everyone who never touched them.
Typing the defaults back in resets rather than pins.

**A scoring change rescores history.** Snapshots store sub-scores, not
weighted points, so weighting is applied on read; the socializing sub-score
depends on the target and mode, so every settings write that touches the
model re-runs `WeekSealer`. Without that, Trends keeps reporting the old model
until the next launch.

### Pace vs. progress

Every formula is a weekly total, but the dashboard is live mid-week, so
`ScoreBasis` offers two readings of the same data:

- **`pace`** — targets scaled to days elapsed. An on-track Wednesday reads
  ~100%. Drives the ring by default.
- **`fullWeek`** — raw progress toward the whole week. That same Wednesday
  reads ~43%. Drives the category bars.

The ring is tappable to switch between them; the choice persists in settings.

### The golden test

[`test/scoring/calculator_test.dart`](../test/scoring/calculator_test.dart)
pins a canonical reference week (Mon 2026-07-06 – Sun 2026-07-12):

```
524 min moderate · 0 vigorous · 50 resistance · 50 flexibility
136 min nature · 22 h social · nights [9, 9, 8.5, 8.75, 8, 7.75, 7.25]
→ 81.84%      (+200 min vigorous → 92.44%)
```

Every sub-score is asserted to 9 decimal places — **this is the test that
tells you whether you broke the model.** A separate test in the same
directory re-derives all three piecewise curves from their hazard ratios,
keeping the HR→score relationship provable rather than just pinned.

## State and ownership

### Schema (Drift, `schemaVersion = 4`)

| Table | Rows | Key | Written by | Read by |
| --- | --- | --- | --- | --- |
| `daily_entries` | one per logged activity | `occurred_at` (UTC, ordering) + `local_date` (`YYYY-MM-DD`, bucketing) | quick-log sheet, category detail, import | `MetricsRepository`, `WeekSealer`, export |
| `weekly_snapshots` | one per sealed week | unique `week_start_date` | `WeekSealer` | Trends (`AnalyticsPage`) |
| `app_settings` | single row, id 0 | — | `SettingsPage`, plus the dashboard for `tutorial_seen` | `providers.dart` graph |
| `social_check_ins` | one per answered day | unique `local_date` | quick-log / detail sheet's check-in control | `MetricsRepository`, `WeekSealer`, `earliestActivityDate()` |

**Why check-ins are keyed by day, not by week.** The week start is
user-configurable; keying an answer to a week start would orphan it, or drag
it into the wrong week, the moment that setting changed. A week takes its
**latest** answer, so changing your mind on Sunday supersedes Tuesday without
a special case.

**Why custom weights are one JSON column** on `app_settings` rather than
seven numeric columns. The meaningful state is "custom or not", which one
nullable column expresses exactly, and adding a category to the model then
needs no migration. Weights are keyed by name inside it, never by ordinal —
same rule as exports, below.

**Why two timestamp columns on `daily_entries`.** `local_date` decides which
day — and therefore which week — an entry belongs to. A log at 11pm stays on
the day you experienced it regardless of timezone travel or DST. The key
sorts lexicographically, so week-range queries are plain string comparisons.

### Weeks

`WeekRange` ([`lib/data/week.dart`](../lib/data/week.dart)) is a half-open
range of seven local days. **The start day is user-configurable**
(`DateTime.monday`…`sunday`) — nothing may assume Monday. Snapshots record the
start day in force when sealed, so changing the setting later cannot
retroactively reinterpret history.

### Weekly sealing

`WeekSealer` ([`lib/data/week_sealer.dart`](../lib/data/week_sealer.dart))
runs on launch (`sealOnLaunchProvider`) and walks **every** completed week
since the first entry — the user may have been away for months, not one week.
This follows directly from fact 2 above (no background execution).

It is idempotent and **recomputes rather than skips** weeks that already have
a snapshot: a snapshot is derived data, so editing a past entry corrects
history instead of leaving it stale. Weeks with nothing recorded are skipped
— a gap in logging is not a zero-scoring week.

The walk is bounded by `earliestActivityDate()`, which considers **check-ins
as well as entries**: under satisfaction mode a whole week can consist of one
answer and nothing else, and a walk bounded by entries alone would never
reach it.

It also runs on demand after any settings change that moves the model (see
"a scoring change rescores history" above).

## Encryption

The database file is encrypted with SQLCipher. A 256-bit key is generated on
first launch and stored in the platform keychain
([`database_key.dart`](../lib/data/database_key.dart));
[`connection.dart`](../lib/data/connection.dart) applies it via `PRAGMA key`.

Two guards, both fail closed:

- **`DatabaseNotEncryptedException`** — on plain SQLite, `PRAGMA key` is an
  unrecognised no-op that *reports success*. The app would run perfectly
  while writing plaintext. So `PRAGMA cipher_version` is checked **before**
  the key is applied and before any write can occur.
- **`MissingDatabaseKeyException`** — a database with no key throws rather
  than minting a replacement, which would render the file permanently
  unreadable and silently destroy the user's history.

The key lives only in this device's keychain (`first_unlock_this_device`, so
it is not in an iCloud backup). It goes missing in two real situations: a
restore onto another device, and a reinstall under different signing that
leaves the app container's `Documents` intact. Neither is recoverable —
nothing can decrypt the file — so `LockedDatabaseView` states that plainly and
offers *Try again* (the keychain reads as absent until the device has been
unlocked once since boot) and a confirmed *Start fresh* that deletes the
file. The dashboard withholds logging, export, and Trends while locked. A
zero-length file does not count as an existing database — SQLite leaves one
behind if interrupted before the first page is written, and it would
otherwise strand the app on this error over nothing.

**`flutter test` cannot prove encryption.** Host tests use an in-memory
database and the plugin's native library never loads, so the Dart VM falls
back to system SQLite. Unit tests cover key management only. To verify for
real, run on a device and check the file header:

```bash
C=$(xcrun simctl get_app_container <udid> com.nttech.loggevity.dev data)
xxd -l 16 "$C/Documents/loggevity.sqlite"
```

A plaintext database begins with the ASCII `SQLite format 3`. An encrypted
one begins with random bytes. `sqlite3 <file> .tables` should fail with
`file is not a database (26)`.

### Backup is disabled, deliberately

`android:allowBackup="false"`, plus a `data_extraction_rules.xml` that
excludes every storage domain from both cloud backup and device-to-device
transfer.

This is not caution, it is arithmetic. The database is encrypted and its key
is in the Android Keystore, which is non-exportable and never leaves the
device. Backing the database up therefore restores, onto the new phone, a file
that nothing on it can read — and the user meets `LockedDatabaseView` telling
them their history is gone. Android does this **by default**: `allowBackup` is
true when the attribute is absent, which it was until 2026-08-27.

Two attributes are needed, not one. `allowBackup="false"` disables cloud
backup on every version, but **on Android 12+ device-to-device transfer is
governed only by `dataExtractionRules`** — so without the rules file a new
phone still drags the unreadable database across.

There is nothing else to lose: settings live in the same encrypted file, so
excluding everything costs the user nothing they could have kept.

**This makes JSON export the real migration path**, not a convenience. It is
plaintext precisely so it survives a change of device, and anything that makes
it harder to find is a regression in disaster recovery, not just in polish.

`test/release_config_test.dart` pins all of this, along with the absence of
any `INTERNET` permission. None of it can fail a normal test — it is native
configuration read at package and install time — and it sits in files that
tooling rewrites without asking.

**Never add `sqlite3_flutter_libs`.** It conflicts with
`sqlcipher_flutter_libs`; whichever loads first wins, and that is exactly how
plaintext writes sneak in.

### Migrations

`onUpgrade` in [`database.dart`](../lib/data/database.dart) is additive and
version-guarded. `test/data/migration_test.dart` opens a real v1 database,
migrates it, and asserts the data survives. Add a step there for every schema
bump.

## Import / export

Pure serialisation lives in
[`portability.dart`](../lib/data/portability.dart); file I/O and the
share/pick dialogs live in
[`backup_service.dart`](../lib/data/backup_service.dart). Both JSON and CSV
carry the same entries and social check-ins; only JSON carries the scoring
settings, because a single flat table has nowhere honest to put them.

- **Categories serialise by name, never by ordinal.** Ordinals are a storage
  detail; exporting them would silently recategorise everything if the enum
  ever moved.
- **Import is additive and de-duplicated** on a fingerprint of
  `localDate|category|value|occurredAt`. Re-importing the same file is a
  no-op rather than a way to double your week. Import never deletes.
- **Bad rows are reported, not fatal.** A malformed entry is skipped with a
  message naming the row and reason; the rest still import. Only a file that
  isn't a Loggevity export at all is rejected outright.
- **Check-ins overwrite where entries skip.** A later answer for a day
  already answered is a correction, not a duplicate. In CSV they ride in the
  entry table under the reserved category `socialSatisfaction` with a `1`/`0`
  value; the parser routes them back out, accepting whatever spelling of a
  boolean a spreadsheet produced.
- **Settings are offered, never applied.** An import merges data; adopting
  someone else's weights would rewrite the importer's whole history without
  them asking. If a file's model differs from the one in force, the user is
  shown what would change and chooses. Weights marked as the research set
  come back as the default rather than as an override.

Export schema version is **2** (v1 files still import; they simply carry
neither check-ins nor settings).

## Build flavours

Debug and profile builds install **alongside** release builds rather than
replacing them, so a dev build cannot clobber real data:

| Build | Application ID | Display name |
| --- | --- | --- |
| debug / profile | `com.nttech.loggevity.dev` | Loggevity Dev |
| release | `com.nttech.loggevity` | Loggevity |

Android via `applicationIdSuffix` and `resValue` in
[`android/app/build.gradle.kts`](../android/app/build.gradle.kts); iOS via
per-configuration `PRODUCT_BUNDLE_IDENTIFIER` plus an `APP_DISPLAY_NAME` build
setting that `Info.plist` reads through `$(APP_DISPLAY_NAME)`. Separate
containers, separate keychain entries, so **the dev build cannot read the
release build's database.**

**Check this after any Xcode project edit.** These four settings were silently
wrong until 2026-08-27: iOS had Debug and Release *swapped*, so debug builds
installed under the release identifier — sharing the real container, which is
exactly what the separation exists to prevent — while Release carried `.dev`.
`Info.plist` also hardcoded the display name, so the iOS dev build never
showed "Loggevity Dev" at all. Xcode rewrites this file whenever settings are
touched in the GUI, and nothing in the test suite can catch a regression,
because the flavour only exists at native build time. To verify:

```bash
flutter build ios --simulator --debug
plutil -p build/ios/iphonesimulator/Runner.app/Info.plist \
  | grep -i "bundleident\|displayname"
```

A debug build must report `com.nttech.loggevity.dev` and `Loggevity Dev`.

## Errors that escape

`installErrorHandlers()` runs before `runApp` and covers three separate escape
routes, none of which catches the others:

| | |
| :--- | :--- |
| `FlutterError.onError` | Errors raised inside the framework — build, layout, paint, gestures. |
| `PlatformDispatcher.onError` | Uncaught async errors that never reach the framework, such as a rejected Future nobody awaited. Returns `true`; returning `false` hands the error to the platform, which on Android is a process kill. |
| `ErrorWidget.builder` | What is *drawn* where a widget failed to build. Without it, release shows a bare grey rectangle. |

**None of them can report anything.** The app has no network permission and no
crash reporting, so what these buy is not diagnosis: it is a failure that does
not look like data loss, and text the user can copy into a bug report.

`AppErrorView` is **deliberately self-sufficient** — its own `Directionality`,
no `Theme`, no `Material`, literal colours. `ErrorWidget.builder` can be
invoked for a failure anywhere in the tree, including above `MaterialApp`
where none of those ancestors exist; a fallback that inherits them throws
while handling an error and replaces a legible failure with an unreadable one.
`error_handling_test.dart` pins that by rendering it with no ancestors at all.

Its most important line is the one saying logged data is unaffected. An error
screen in a health tracker reads as lost history, and a UI failure leaves the
database untouched.

`ErrorLog` keeps the last ten errors **in memory only**. Writing crash state
into the encrypted database from an error handler risks compounding a failure
whose cause might be the database.

**Testing note:** the harness asserts `ErrorWidget.builder` is back to its
default as soon as a test body returns — before any `tearDown`. A test that
installs the handlers has to restore it inside the body, the same constraint
that governs unmounting Drift-backed widgets.

## Releasing

Three scripts, deliberately not one:

| | |
| :--- | :--- |
| `scripts/bump_version.sh [patch\|minor\|major]` | Advances the version, commits. Builds nothing. |
| `scripts/build_appbundle.sh` | Signed `.aab` — the Play Store artifact. |
| `scripts/build_apk.sh` | Signed `.apk` — sideloading and device testing. |

`android/next_version_name.txt` and `next_build_number.txt` hold the
**pending** release; `pubspec.yaml` holds the one just set. So bumping writes
pubspec and moves the counters on, and the build scripts simply read pubspec —
no `--build-name`/`--build-number` needed.

**Building is separated from bumping because a build number is spent
forever.** Play requires `versionCode` to strictly increase, so a failed
build, or a rebuild of the same version as a different artifact type, must not
consume another one. Under the old combined script every attempt burned a
number whether or not anything shippable came out.

**Play needs the `.aab`.** App Bundles are mandatory for any listing created
after August 2021; `.apk` is accepted only for older listings. The APK script
stays because sideloading a real signed build onto a device is still the
fastest way to check something.

## Release signing

Release builds are signed with an **upload key** whose credentials live in
`android/key.properties` — gitignored, alongside `*.jks` and `*.keystore`.
The file is absent on a fresh clone, so every read in
`android/app/build.gradle.kts` tolerates it not existing.

```properties
storeFile=/absolute/path/outside/the/repo/upload-keystore.jks
storePassword=…
keyAlias=upload
keyPassword=…
```

Create the keystore yourself; nothing in this repo should ever generate it or
know the password:

```bash
keytool -genkey -v -keystore ~/upload-keystore.jks \
  -keyalg RSA -keysize 2048 -validity 10000 -alias upload
```

**Without `key.properties`, a release build silently falls back to the debug
key.** That is deliberate — `flutter run --release` should still work for
somebody who has no business holding the upload key — but it is the most
dangerous state the build can be in, because a debug-signed artifact installs
and runs exactly like a real one and only fails at the Play Console.

Gradle logs a warning in that case, and **the warning is not enough**: it does
not survive Flutter's output filtering, so nobody sees it. The real guard is
`scripts/lib/signing.sh`, which reads the certificate back out of the built
artifact and refuses to go on if it says `CN=Android Debug`. Both build
scripts call it before renaming anything, so a failure leaves nothing behind.

It uses **`apksigner`, not `keytool`**. APKs are signed with signature scheme
v2/v3, which `keytool -printcert -jarfile` cannot read: it reports "Not a
signed jar file", a grep for the debug certificate then finds nothing, and the
check passes precisely when it can see least. For the same reason, a missing
`apksigner` fails the release rather than skipping the check.

## Testing

| Area | Files | Tests |
| --- | --- | --- |
| Scoring | `test/scoring/` | 63 |
| Data | `test/data/` | 131 |
| UI | `test/ui/` | 123 |
| Tutorial overlay | `test/tutorial_overlay_test.dart` | 14 |
| Release config | `test/release_config_test.dart` | 4 |
| | | **335** |

```bash
flutter test                                   # everything
flutter test test/scoring/                     # pure model, fastest signal
flutter test test/ui/dashboard_test.dart -r compact
```

`purity_test.dart` is not a normal test — it parses `lib/scoring/` source and
fails the build if anything there imports Flutter or `dart:io`. Don't "fix"
it by loosening the check; fix the import instead.

### Gotchas that will cost you an hour

- **Drift + widget tests deadlock if you await stream cancellation.** Drift
  defers stream cleanup to a zero-duration timer that cannot fire during
  widget disposal; a hand-rolled `switchMap` awaiting `inner.cancel()` hung
  `pumpWidget` forever. Compose streams in Riverpod
  ([`combine_latest.dart`](../lib/data/combine_latest.dart)), not by hand.
- **Unmount inside the test body.** The framework checks for pending timers
  as soon as the body returns — before any `addTearDown` — so the tree must
  come down while pumps are still available. The `withDashboard` helper ends
  with `pumpWidget(SizedBox())` then a single bounded `pump`.
- **Never `pumpAndSettle` after unmounting.** It pumps while frames stay
  scheduled and Drift teardown keeps rescheduling them. One
  `pump(Duration(milliseconds: 10))` is enough and cannot loop.
- **An indeterminate `CircularProgressIndicator` never settles.** If a
  provider doesn't emit, `pumpAndSettle` spins rather than failing fast.
- **Widget tests use a tall viewport** so finders don't miss content a phone
  would push below the fold. `ListView` builds lazily — an off-screen row
  does not exist as far as `find` is concerned.

## Invariants

Break these and data corrupts silently — no crash, no test failure unless you
add one.

1. **Never reorder `ActivityCategory`.** Drift persists `intEnum` columns as
   the ordinal, so reordering recategorises every stored row. Pinned by
   `database_test.dart`.
2. **Store raw values, derive everything else.** Sleep especially: raw hours
   in, adjusted hours computed.
3. **Never clamp negative sub-scores to zero.** They are the model's signal
   for overtraining and sleep deprivation.
4. **Never assume the week starts on Monday.**
5. **Keep `lib/scoring/` pure** (no Flutter, no `dart:io`, no clock).
6. **Never ship `sqlite3_flutter_libs` alongside SQLCipher.**
7. **Export categories by name, not ordinal.**

## Known gaps

- Encryption is verified manually on-device, not in CI.
- `flutter build ios --simulator` needs a simulator runtime matching the
  installed SDK; targeting a booted device directly (`flutter run -d <udid>`)
  works regardless.
- macOS builds compile but keychain entitlements are not configured, so
  `flutter_secure_storage` will not work there.
