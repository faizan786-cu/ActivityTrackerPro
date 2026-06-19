# ActivityTracker Pro

A personal device-usage tracker for Android 14, built with Kotlin, Jetpack
Compose (Material 3), Room, and Hilt.

## Opening the project
1. Open the `ActivityTrackerPro` folder in Android Studio (Koala or newer).
2. Let Gradle sync — it will pull all dependencies from Google/Maven Central.
3. Run on a device or emulator running API 26+ (targets/compiles against API 34).

## How tracking works
- `UsageTrackingService` is a foreground service that polls
  `UsageStatsManager.queryEvents()` every 5 seconds and pairs
  `MOVE_TO_FOREGROUND` / `MOVE_TO_BACKGROUND` events per package into
  closed sessions. Background activity never appears in this event
  stream, so it's naturally excluded.
- Each closed session is sent to `UsageRepository.recordSession()`, which
  inserts a `SessionEntity` row and folds the session into that day's
  `AppUsageEntity` (opens, first open, last close, total usage).
- Unlocks are counted via an `ACTION_USER_PRESENT` broadcast receiver
  registered inside the service.
- `reportDate` on every entity is a local "yyyy-MM-dd" key computed by
  `DateUtils`, so midnight rollovers and timezone changes are handled in
  one place.
- `BootCompletedReceiver` restarts the service after a reboot if Usage
  Access is already granted.

## Permissions
- `PACKAGE_USAGE_STATS` — special permission granted through Settings,
  not a runtime dialog. `OnboardingScreen` deep-links the user to
  `Settings.ACTION_USAGE_ACCESS_SETTINGS`.
- `POST_NOTIFICATIONS` — requested at first launch on Android 13+, needed
  to show the (minimum priority) ongoing tracking notification required
  for the foreground service.
- `FOREGROUND_SERVICE` / `FOREGROUND_SERVICE_DATA_SYNC` — for
  `UsageTrackingService`.
- `RECEIVE_BOOT_COMPLETED` — to resume tracking after reboot.

## Reports
- `ReportPdfGenerator` renders the four required sections (Daily Summary,
  App Usage Statistics, Detailed Session Timeline, Usage Totals) with a
  page footer (timestamp + page number) using Android's native
  `PdfDocument`/`Canvas`, so no extra PDF dependency is required.
- Generated PDFs are written to `filesDir/reports/` and exposed through a
  `FileProvider` for in-app viewing (`ReportsScreen`) and for the email
  intent.
- `EmailSender` opens an `ACTION_SEND` chooser with the PDF attached and
  the subject pre-filled as `Daily Activity Report - [Date]`. The app
  never sends email itself or stores SMTP credentials — the user's own
  email client does the sending.

## Architecture
MVVM + Clean Architecture:
- `data/local` — Room entities/DAOs/database.
- `data/repository` — single source of truth for usage data and settings.
- `di` — Hilt modules.
- `service` — foreground tracking service + boot receiver.
- `pdf`, `email` — report generation and sharing.
- `ui` — Compose screens + ViewModels, one package per screen.
- `navigation` — Compose Navigation graph (Onboarding → Dashboard ↔ Settings/Reports).

## Notes / known simplifications
This is a complete, runnable skeleton covering every requirement in the
spec. A few areas you may want to extend before shipping:
- App icons are currently loaded ad hoc; consider caching `Drawable`
  icons in a small LRU cache for the Top Used Apps list.
- The polling interval (5s) balances accuracy against battery; tune via
  `UsageTrackingService.POLL_INTERVAL_MS` if needed.
- Add a `WorkManager` job (dependency already included) if you want
  reports to auto-generate at a fixed time daily, in addition to the
  manual "Generate Report" button.
