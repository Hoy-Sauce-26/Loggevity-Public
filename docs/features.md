# Features

Where the code for each user-visible feature lives. For *why* the model or
storage is shaped the way it is, see [architecture.md](architecture.md).

## Dashboard & score ring

| | |
| :--- | :--- |
| Entry point | `lib/screens/dashboard_page.dart` → `DashboardPage` |
| Flow | `providers.dart#currentWeekProvider` (Drift stream) → `MetricsRepository` → `HealthScoreCalculator` (`lib/scoring/calculator.dart`) → `ScoreRing` (`lib/widgets/score_ring.dart`) + `CategoryProgressTile` per category |
| State | `daily_entries`, `social_check_ins`, `app_settings` (read-only here) |
| Tests | `test/ui/dashboard_test.dart` |

Home screen: score ring, seven category rows, a floating Log button. The ring
is tappable to switch between `pace` and `fullWeek` readings (see
architecture.md, "Pace vs. progress"); the choice persists in
`app_settings.ringShowsPace`. The app bar opens Trends, Settings, and a "Your
data" menu (export JSON / export CSV / import).

## Logging an activity

| | |
| :--- | :--- |
| Entry point | `lib/widgets/quick_log_sheet.dart` (floating button) or `lib/widgets/category_detail_sheet.dart` (tap a category row) |
| Flow | `DaySelector` (`lib/widgets/day_selector.dart`) picks the date → `AmountDialog` / `NumberField` collects the value → `MetricsRepository` writes a `daily_entries` row |
| State | `daily_entries` |
| Tests | `test/ui/log_input_test.dart`, `test/ui/back_dating_test.dart`, `test/ui/category_detail_test.dart` |

**Logging is not assumed to be same-day.** `DaySelector` sits above the
categories in the quick-log sheet and above the detail sheet's input; the
edit dialog carries one too, so an entry on the wrong date can be moved. Days
after today are disabled. Back-dating matters most for sleep: without it a
night remembered the next afternoon both mis-credits today and scores the
night it actually happened as missing. Back-dated entries keep the current
time of day rather than landing at midnight, so the entry list stays ordered.

The date pickers cover the current week only — earlier weeks are sealed into
`weekly_snapshots`, and editing them would mean re-sealing history.

Tapping a category opens `CategoryDetailSheet`: a day-by-day chart for that
category, its own log input, and the week's entries grouped by day with
edit/delete.

## Social check-in (satisfaction mode)

| | |
| :--- | :--- |
| Entry point | `lib/widgets/social_check_in.dart` → `SocialCheckInControl` |
| Flow | Shown in `quick_log_sheet.dart` and `category_detail_sheet.dart` in place of the hours input when `socialModeProvider` is `SocialMode.satisfaction`; writes to `social_check_ins` keyed by `local_date` |
| State | `social_check_ins` (latest answer per day wins) |
| Tests | `test/ui/log_input_test.dart`, `test/data/metrics_repository_test.dart` |

Deliberately **not** a switch: the question has three states — yes, no, and
not yet asked — and collapsing "unanswered" into "no" on screen would hide
the thing the user needs to act on, even though the score treats them alike
(see architecture.md, "Satisfaction mode").

## Trends (analytics)

| | |
| :--- | :--- |
| Entry point | `lib/screens/analytics_page.dart` → `AnalyticsPage` |
| Flow | `providers.dart#snapshotsProvider` (`weekly_snapshots` stream) → `ScoreTrendChart` (`lib/widgets/charts/score_trend_chart.dart`) + `CategoryContributionChart` (`lib/widgets/charts/category_contribution_chart.dart`), both formatted via `lib/widgets/charts/chart_format.dart` |
| State | `weekly_snapshots` (read-only) |
| Tests | `test/ui/analytics_test.dart`, `test/ui/chart_format_test.dart` |

Fills the screen rather than scrolling: headings, stats, and legend take what
they need; the two charts sit in `Expanded` and split the remainder. Below
`AnalyticsPage._minimumHeightToFill` the charts take fixed heights and the
page scrolls instead.

Both axes are pinned to their ceiling exactly (`pinnedAxis` in
`chart_format.dart`) — composite to `kCompositeCeiling` (100%), contribution
to `ScoringConfig.maxPoints` — so a perfect week fills the plot and the two
charts always agree on height for the same week. Gridlines and tooltips are
drawn by the app, not fl_chart's defaults; see architecture.md's UI notes
inline in `chart_format.dart` and the chart widgets themselves for the
specific fl_chart defaults each one works around.

They are also positioned to agree horizontally, since they stack over the same
weeks. fl_chart centres a bar group at `(i + 0.5) x width/count` where a line
chart plots straight onto its axis, so `ScoreTrendChart` runs `minX: -0.5` to
`maxX: count - 0.5` to land on the same slot centres — which also gives a lone
week somewhere to sit, `0..0` being a zero-width axis that pins the dot to the
left frame. Both charts reserve `kAxisLabelWidth` for their y-labels, because
a chart reserving more shifts its whole plot relative to the other.

## Settings

| | |
| :--- | :--- |
| Entry point | `lib/screens/settings_page.dart` → `SettingsPage` |
| Flow | `NumberField` (`lib/widgets/number_field.dart`) validates per keystroke → writes `app_settings` → `WeekSealer.sealCompletedWeeks()` re-runs if the write changed the model |
| State | `app_settings` (single row, id 0) |
| Tests | `test/ui/settings_test.dart`, `test/scoring/config_test.dart` |

Holds everything the user may change: week start day, socializing target and
mode, and — behind an Advanced settings dialog carrying a standing warning —
`WeightsDialog` (`lib/widgets/weights_dialog.dart`) for category weights.
Saving is disabled while any value is unusable, so nothing invalid reaches
storage. See architecture.md, "What the user may change", for why these
specific settings are the exposed surface and nothing else.

## First-run tour

| | |
| :--- | :--- |
| Entry point | `lib/screens/dashboard_page.dart` → `_DashboardPageState._maybeShowTour()` |
| Flow | Post-frame after the first week loads → `showTutorial()` (`lib/widgets/tutorial_overlay.dart`) → writes `app_settings.tutorial_seen` |
| State | `app_settings.tutorial_seen` (schema v4) |
| Tests | `test/tutorial_overlay_test.dart` (the overlay), `test/ui/tutorial_wiring_test.dart` (the wiring) |

Five spotlit steps: score ring, a category row, the Log button, Trends,
Settings. The overlay is generic and app-agnostic; everything specific to
Loggevity is the target list and the trigger.

Four things about the wiring that are load-bearing:

- **The keys are owned by `_DashboardPageState`, not built in `build()`.** A
  `GlobalKey()` constructed during build is a new key every frame and never
  resolves to a widget.
- **It only runs when the ring is actually mounted.** The loading spinner and
  `LockedDatabaseView` render *instead of* the dashboard, so the tour would
  otherwise dim the app to explain widgets that do not exist. The locked
  screen is the worst possible moment for a walkthrough.
- **The flag is written on skip as well as on finish.** Somebody who skipped
  has answered; asking again next launch ignores that.
- **`tutorial_seen` is its own column rather than inferred from empty data.**
  A week with nothing logged looks exactly like a fresh install, and
  re-running the tour on a month-old user is worse than never running it.

Replay lives in Settings → About, which **pops with `true` rather than
starting the tour itself** — the tour describes the dashboard, so it has to
run there, with those keys mounted.

## Methodology page

| | |
| :--- | :--- |
| Entry point | `SettingsPage` → About → "How scoring works" → `lib/screens/methodology_page.dart` |
| Flow | Reads `ActivityCategory.riskReduction`, `.credibility` and `.researchWeight` directly; `scoringConfigProvider` only to flag that a custom weighting is in force |
| State | None — read-only, and it renders the *published* model rather than the weights actually in force |
| Tests | `test/ui/methodology_test.dart` |

The user-facing account of where the weights come from. Every figure is read
off the enum rather than written into the copy, so a reweighting updates the
page automatically — `methodology_test.dart` pins that by deriving its
expectations from `ActivityCategory` too.

Categories are listed heaviest first, which is deliberately not the enum's
order. The enum is a storage contract (see invariant 1 in
[architecture.md](architecture.md)) and cannot be rearranged for reading, so
the page sorts a copy.

## Import / export ("Your data" menu)

| | |
| :--- | :--- |
| Entry point | `DashboardPage`'s "Your data" menu → `lib/data/backup_service.dart` → `BackupService` |
| Flow | Export: `daily_entries` + `social_check_ins` (+ settings for JSON) → `lib/data/portability.dart` serialisers → share sheet. Import: file picker → `portability.dart` parsers → de-dup by fingerprint → `MetricsRepository` writes → settings diff shown to user before applying |
| State | Reads all tables; writes `daily_entries` / `social_check_ins` only (never `app_settings` without confirmation) |
| Tests | `test/data/portability_test.dart`, `test/data/backup_service_test.dart` |

See architecture.md, "Import / export", for the de-duplication fingerprint,
why settings are offered rather than applied, and the CSV encoding of
check-ins.

## Locked database (recovery)

| | |
| :--- | :--- |
| Entry point | `lib/widgets/locked_database_view.dart` → `LockedDatabaseView` |
| Flow | `lib/data/connection.dart` throws `DatabaseNotEncryptedException` / `MissingDatabaseKeyException` → caught at app root → `LockedDatabaseView` replaces the dashboard |
| State | None — the database is unreadable by definition while this shows |
| Tests | `test/ui/locked_database_test.dart`, `test/data/database_key_test.dart` |

Offers *Try again* (the keychain reads as absent until the device has been
unlocked once since boot) and a confirmed *Start fresh* that deletes the
file. See architecture.md, "Encryption", for why this state is unrecoverable
and how it's reached.
