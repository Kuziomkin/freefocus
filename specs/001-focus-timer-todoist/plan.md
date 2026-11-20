# Implementation Plan: Free Focus v1 — Menu Bar Focus Timer for Todoist

**Branch**: `001-focus-timer-todoist` | **Date**: 2026-11-20 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `/specs/001-focus-timer-todoist/spec.md`

## Summary

Free Focus is a macOS menu bar Pomodoro timer. It shows today's and overdue tasks from Todoist, times one task at a time, and closes tasks at the source. The primary Flutter window hosts the popover and the mini window. Settings, Report, and Connect are secondary windows. macOS-only behaviour (always-on-top mini panel, Keychain, notifications, login item) sits behind small Swift platform channels. All timing comes from wall-clock end dates in a pure-Dart `TimerEngine`. Task and session data are stored locally in SQLite. The only network host is `api.todoist.com`.

The technical approach and its rationale are in [research.md](research.md). Entities are in [data-model.md](data-model.md). The Todoist endpoints are in [contracts/todoist-endpoints.md](contracts/todoist-endpoints.md). The internal provider boundary is in [contracts/task-provider.md](contracts/task-provider.md). Validation steps are in [quickstart.md](quickstart.md).

## Technical Context

**Language/Version**: Dart 3.14 (Flutter stable 3.50 or later); Swift 5.9 or later for platform channels.

**Primary Dependencies** (each justified in research.md; all pinned at the version shown):
- `tray_manager` 0.9.0: menu bar item and right-click menu (research §3)
- `window_manager` 0.7.0: window size, position, and styling (research §4)
- `desktop_multi_window` 0.3.1: secondary windows (research §5)
- `sqlite3` 3.9.1: session and task storage (research §7)
- `shared_preferences` 2.5.6: settings and timer state via `SharedPreferencesAsync` (research §8)
- `intl` (pinned to the version that matches the Flutter SDK): required by generated localisation code (research §15)
- SDK packages only otherwise: `flutter`, `flutter_localizations`, `flutter_test`

**Storage**: SQLite file in Application Support for focus sessions and completed tasks; JSON cache for tasks and projects; `SharedPreferencesAsync` for settings and timer state; macOS Keychain for the token only.

**Testing**: `flutter_test` for unit and widget tests; hand-written fakes for `HttpTransport`, channels, and `TaskProvider`. CI on GitHub Actions macOS runners.

**Target Platform**: macOS 14.0 or later. Everything outside `macos/` stays platform-neutral.

**Project Type**: Desktop menu bar application (single Flutter project with a macOS runner and Swift channels).

**Performance Goals** (plan-level targets; the spec has only the timer and sync targets):
- Popover shows the cached task list within 300 ms of opening.
- Menu bar label and popover timer update at least once per second with at most 1 s error against the wall clock.
- Completing a task shows the strike-through within 100 ms; the provider call finishes within 5 s on a normal connection (SC-007).

**Constraints**:
- No backend, accounts, analytics, or telemetry (Principle III).
- Network only to `api.todoist.com` (Principle III, FR-005).
- Light mode only; English only in v1 but all strings in ARB files.
- Memory: the app runs as a menu bar utility, so the plan targets a steady-state budget of 150 MB resident, to be verified manually during quickstart step 3.

**Scale/Scope**: One user per install. Expected sessions: about 4 to 12 focus intervals per day, so roughly 4,000 rows per year. SQLite with the indexes in data-model.md is more than enough; no pagination of local data is needed.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | How the plan complies |
| --- | --- | --- |
| I. Scope Discipline | Pass | Exactly five screens (popover, mini window, Report, Settings, Connect). One provider (Todoist). The Quit control, Delete all data, and diagnostic log export come from approved clarifications, not extras. No features outside the spec. |
| II. Provider Independence | Pass | `TaskProvider` interface and `FocusTask` model in `lib/core/tasks/`. Todoist code only in `lib/providers/todoist/`. `FakeProvider` in `test/support/` drives every screen test (contracts/task-provider.md). |
| III. Local-First Privacy | Pass | No backend or analytics. Network only to `api.todoist.com`. Token only in Keychain via the Swift `freefocus/keychain` channel. Diagnostic log excludes tokens and titles. |
| IV. Timer Correctness | Pass | `TimerEngine` is pure Dart, with no UI or provider imports. Time comes from `endDate`. Test-first: failing tests for each rule in data-model.md are written before the engine (tasks T010 to T014). State persists on every change. |
| V. Platform Boundaries | Pass | Flutter stable, Dart 3, macOS 14+. macOS-only code only in `macos/Runner` with a documented channel list (`macos/Runner/CHANNELS.md`). Popover and mini share one Flutter window. Secondary windows use `desktop_multi_window`. |
| VI. Design Fidelity | Pass | All visual values in `lib/theme/tokens.dart` (design-token list in spec). Funnel Sans, Literata, and Google Sans Code bundled under `assets/fonts/` with OFL licences. Light mode only. Reviewers reject any widget literal outside tokens. |
| VII. Testing | Pass with work | Unit tests for `TimerEngine`, the Todoist adapter (with a fake `HttpTransport`), and the session store. Widget tests for shared components. CI workflow created in T004 before any merge. |
| VIII. External API Discipline | Pass with check | Endpoints verified against the official OpenAPI document and reference on 2026-11-20 (research §13, contracts/todoist-endpoints.md). **Open**: the personal-token format and the 429 rule must be confirmed on the live page before the client merges (T001). The spec's "items" key is corrected to `results`. |
| IX. Dependencies | Pass with spike | Six Dart packages (plus `intl`), each justified in research.md, pinned, and released within the last 12 months (tray_manager 8 days, window_manager 8 days, desktop_multi_window about 2 months, sqlite3 12 days, shared_preferences 45 days). Keychain, notifications, login item, and the mini panel are Swift code, not packages. **Open**: `desktop_multi_window`'s README asks for a git fork of `window_manager`. A git ref is not a release, so the spike T000 must show the pub.dev release works before the fork is considered. |
| X. Accessibility and Brand | Pass | Icon-only controls get semantic labels (FR-080). Text contrast at least 4.5:1: the smallest caption colour `muted2` #737373 on white is about 4.7:1. "Free Focus" as the name, with no "Todoist" in it. The Doist disclaimer appears in Settings and the Connect window (FR-072, FR-076). |

**Result**: No violations. Two items are "Pass with work/spike", which are tracked as tasks and must close before the corresponding code merges. Under Governance, a failed check blocks implementation, and these are not failures.

**Deviation recorded**: The source doc's SwiftUI and SwiftData sections are superseded by Principle V (see research §1). No constitution rule is broken by this; the spec's behaviour is unchanged.

## Project Structure

### Documentation (this feature)

```text
specs/001-focus-timer-todoist/
├── plan.md                    # This file
├── research.md                # Phase 0 output
├── data-model.md              # Phase 1 output
├── quickstart.md              # Phase 1 output
├── contracts/
│   ├── todoist-endpoints.md   # Verified external API contract
│   └── task-provider.md       # Internal provider boundary
├── checklists/
│   └── requirements.md
└── tasks.md                   # Created by /speckit.tasks, not by this command
```

### Source Code (repository root)

```text
lib/
├── main.dart                        # Entry; routes by window argument to the right root
├── app/
│   ├── primary_window.dart          # Popover + mini window host (one Flutter window)
│   └── secondary_windows.dart       # Settings, Report, Connect roots (desktop_multi_window)
├── theme/
│   ├── tokens.dart                  # All colours, text styles, radii, shadows, spacing
│   └── theme.dart                   # ThemeData (light only) built from tokens
├── l10n/
│   └── app_en.arb                   # All user-facing strings
├── core/
│   ├── timer/
│   │   ├── timer_engine.dart        # Pure Dart state machine (Principle IV)
│   │   └── timer_state.dart
│   ├── tasks/
│   │   ├── task_provider.dart       # Interface (contracts/task-provider.md)
│   │   ├── focus_task.dart          # Normalised model
│   │   ├── provider_registry.dart   # One line per provider
│   │   └── task_cache.dart          # Offline JSON cache
│   ├── store/
│   │   ├── database.dart            # sqlite3 open, WAL, schema version
│   │   └── session_store.dart       # Sessions, completed tasks, Report queries, CSV
│   ├── settings/
│   │   └── settings_store.dart      # SharedPreferencesAsync wrapper
│   ├── credentials/
│   │   └── credential_store.dart    # Keychain via channel
│   ├── log/
│   │   └── diagnostic_log.dart      # Local, redacted log
│   └── platform/
│       ├── keychain_channel.dart
│       ├── notifications_channel.dart
│       ├── login_item_channel.dart
│       └── mini_panel_channel.dart
├── providers/
│   └── todoist/                     # The only folder that imports Todoist code
│       ├── todoist_provider.dart
│       ├── todoist_client.dart      # HTTP, errors, retries
│       ├── todoist_models.dart      # JSON decoding
│       ├── todoist_priority.dart
│       └── todoist_colors.dart      # TodoistColor enum -> hex
└── ui/
    ├── shared/                      # PrimaryButton, SecondaryButton, IconButton, ModePill,
    │                                # TaskCheckbox, TaskRow, Toggle, GroupedPanel, Eyebrow, ProgressLine
    ├── popover/
    ├── mini/
    ├── report/
    ├── settings/
    └── connect/

assets/
├── fonts/                           # Funnel Sans, Literata, Google Sans Code + LICENSES/
└── sounds/chime.caf

macos/Runner/
├── MainFlutterWindow.swift          # Registers plugins for secondary engines
├── KeychainChannel.swift
├── NotificationsChannel.swift
├── LoginItemChannel.swift
├── MiniPanelChannel.swift           # NSPanel with floating / full-screen auxiliary flags
├── Info.plist                       # LSUIElement=YES, ATSApplicationFontsPath
└── CHANNELS.md                      # Documents every channel name, method, and arguments (Principle V)

test/
├── support/
│   ├── fake_provider.dart           # TaskProvider double (Principle II)
│   ├── fake_http_transport.dart
│   └── fake_channels.dart
├── unit/
│   ├── timer/timer_engine_test.dart
│   ├── tasks/                       # Sort rule, priority mapping
│   ├── providers/todoist/           # Client with fake transport; decoding
│   └── store/session_store_test.dart
└── widget/
    └── shared/                      # One test file per shared component

.github/workflows/ci.yml             # analyze, test, build on macOS
```

**Structure Decision**: One Flutter project at the repository root with a `macos/` runner. All Dart is platform-neutral except `lib/core/platform/` (channel clients, which call into `macos/` and have Dart-side fakes) and the windowing setup in `lib/app/`. The Todoist code is isolated in `lib/providers/todoist/` so that a Windows build or a second provider adds a folder and one registry line.

## Phases and key ordering

Full task breakdown is produced by `/speckit.tasks`. The plan fixes the order that tasks must follow:

1. **Spikes (T000, T001)**: upgrade Flutter; prove the pinned windowing packages work together; confirm the token format and the 429 rule. No feature code before these close.
2. **Tokens and shared components**: `tokens.dart`, fonts, theme, and the shared widgets with widget tests.
3. **Timer engine**: failing tests first for every rule in data-model.md (Principle IV), then the engine.
4. **Storage and settings**: `database.dart`, `session_store.dart` with tests, and `settings_store.dart`.
5. **Todoist adapter**: client and decoding with a fake transport, then the provider. Endpoints must match contracts/todoist-endpoints.md.
6. **Platform channels**: Keychain, notifications, login item, mini panel. Each documented in `CHANNELS.md`.
7. **Popover, then Connect, then mini window, then Settings, then Report**, matching the source doc's build order.
8. **Acceptance**: run quickstart.md end to end, then the spec's checklist.

## Complexity Tracking

| Violation | Why Needed | Simpler Alternative Rejected Because |
| --- | --- | --- |
| Six pinned Dart packages plus `intl` | Named in the source doc; each does work that would otherwise be hundreds of lines | Each is justified individually in research.md. Hand-writing the tray and window layers would add far more code than the packages. |
| Four Swift-plus-Dart channels for macOS features (Keychain, notifications, login item, mini panel) | Mini panel flags, Keychain, and login-item status are not exposed by the chosen packages | Using packages would need APIs the chosen packages don't offer (research §6, §9, §11), or extra packages that Principle IX discourages. |
