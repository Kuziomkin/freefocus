# Research: Free Focus v1

**Date**: 2026-11-20 | **Plan**: [plan.md](plan.md) | **Spec**: [spec.md](spec.md)

Each entry gives the decision, the reason, and the alternatives rejected. Sources are listed where the answer depends on an external fact.

## 1. Application framework and platform split

- **Decision**: Flutter (stable channel, Dart 3) for all UI and logic, targeting macOS 14.0+. macOS-only behaviour goes through small Swift platform channels in `macos/Runner`.
- **Rationale**: Constitution Principle V mandates this. The source doc's overview also specifies Flutter, and its screen table names `tray_manager` and `window_manager`.
- **Conflict resolved**: Parts of the source doc (sections "Screen 1" to "Timer engine", "Todoist integration") describe a SwiftUI and SwiftData build (`MenuBarExtra`, `NSPanel`, `@Observable`, Swift Charts, `SMAppService`, SwiftData). Those sections contradict the Flutter overview and the constitution. The Flutter build follows the constitution. The behaviour in those sections is kept, and the implementation is re-expressed in Flutter plus Swift channels. The SwiftUI-specific choices are recorded as superseded here.
- **Alternatives rejected**: Native SwiftUI (violates Principle V and makes Windows port harder); Electron or Tauri (not in the constitution).

## 2. Toolchain version

- **Decision**: Pin Flutter to a stable release at or above 3.50 (Dart 3.14). The local SDK is 3.44.8 and must be upgraded before the first build.
- **Rationale**: `tray_manager` 0.9.0 and `window_manager` 0.7.0 both state "Flutter 3.50 / Dart 3.14 or later" as a requirement. Source: pub.dev package pages (fetched 2026-11-20).
- **Alternatives rejected**: Pinning older versions of `tray_manager` (legacy 0.5.x API is deprecated and needs Flutter 3.3).

## 3. Menu bar item and tray integration

- **Decision**: `tray_manager` ^0.9.0 for the status item, its tooltip, and the right-click context menu (Quit). The menu bar label (dot plus `MM:SS`) is updated once per second from the timer engine.
- **Rationale**: Named in the source doc's screen table. Maintained (published 8 days before fetch). macOS 10.15+ supported. Gives the right-click menu required by FR-001.
- **Verified details**: `setContextMenuTrigger(ContextMenuTrigger.rightClicked)` must be set, because the default trigger is none. Title text beside the icon uses `trayIcon.setTitle`.
- **Alternatives rejected**: Hand-written Swift `NSStatusItem` (about 150 lines of channel code; violates Principle IX's "about 50 lines" rule, since a package is justified).

## 4. Window management

- **Decision**: `window_manager` ^0.7.0 for sizing, positioning, always-on-top, frameless presentation, hidden title bar, and close interception.
- **Rationale**: Named in the source doc. Maintained (published 8 days before fetch). Provides `windowing.dart` for per-window control of secondary windows.
- **Risk**: The `desktop_multi_window` README (see §5) says "by now, you should use this fork version with a bit fix" and points to a git fork of `window_manager`. Pinning a git fork violates Principle IX's pinning in spirit, because a git ref is not a release. **Spike required** (task T000 in tasks): build the three secondary windows with `window_manager` 0.7.0 from pub.dev. If a defect blocks them, pin the upstream commit that fixes it and record the justification in this section.

## 5. Secondary windows (Settings, Report, Connect)

- **Decision**: `desktop_multi_window` ^0.3.1 to create Settings, Report, and Connect windows. Each window runs its own Flutter engine. The popover and mini window share the primary engine (Principle V).
- **Rationale**: Named in the source doc. Supports macOS. Published about 2 months before fetch, which is inside the 12-month window.
- **Alternatives rejected**:
  - Flutter's built-in windowing API. It is `@internal` in the SDK and gated behind the `enable-windowing` config on stable (`flutter_tools/lib/src/features.dart`), so it is not production-ready.
  - Making all secondary windows Swift-owned `NSWindow`s hosting separate `FlutterViewController`s. More Swift code than the package and it duplicates the plugin's behaviour.
- **Implication**: Each secondary engine needs the macOS plugins registered in `MainFlutterWindow.swift` per the package README. Each engine has its own isolate, so shared state goes through SQLite and `SharedPreferencesAsync` (see §7 and §8), not in-memory singletons.

## 6. Mini window (always on top, full-screen auxiliary)

- **Decision**: Implement the NSPanel behaviour in Swift: `NSPanel` with `.nonactivatingPanel` and `.borderless`, `level = .floating` (or `.normal` when the setting is off), `collectionBehavior = [.canJoinAllSpaces, .fullScreenAuxiliary]`, `hidesOnDeactivate = false`, and `isMovableByWindowBackground = true`. Exposed to Dart through a documented channel `freefocus/mini_panel`.
- **Rationale**: `window_manager` exposes `setAlwaysOnTop` and `setVisibleOnAllWorkspaces` but not the `.nonactivatingPanel` style or `.fullScreenAuxiliary` collection behaviour the spec requires (FR-050, SC-009). Writing those flags in Swift is about 40 lines.
- **Alternatives rejected**: Forcing the flags through `window_manager` (not exposed).

## 7. Persistence of session and task data

- **Decision**: `sqlite3` ^3.9.1 used directly, with a single database file in Application Support. WAL journal mode and a busy timeout, because the primary and secondary engines write to the same file.
- **Rationale**: The Report needs range queries and sums over sessions. Stored as a plain SQL table with a schema version. `sqlite3` bundles SQLite through build hooks, so no external install is needed. Maintained (published 12 days before fetch). Using it directly avoids the code generation of `drift` (one fewer dependency).
- **Concurrency**: The timer and session writes occur in the primary engine. Report and Settings only read, plus the "Delete all data" write. WAL plus a busy timeout is enough for one user on one Mac.
- **Alternatives rejected**: `drift` (adds a code generator and a larger dependency tree); `sqflite` (needs a background isolate wrapper for desktop); a JSON file (no range queries for the Report, and it risks corruption on concurrent writes from two engines).

## 8. Settings and small state

- **Decision**: `shared_preferences` ^2.5.6, using `SharedPreferencesAsync` (not `SharedPreferencesWithCache`).
- **Rationale**: The docs state that the cached API is unsafe across multiple engines because each isolate has its own cache. `SharedPreferencesAsync` reads the platform store on each call, so a Settings change in the Settings engine is visible to the popover engine on its next read.
- **Stored**: timer lengths, cycle length, the "+" increment, sound, mini-on-top, launch-at-login cache, mini auto-open, last mini window frame, and timer state (per FR-026). The token is never stored here (FR-018).
- **Alternatives rejected**: `hive` (another dependency for a few keys); the existing `UserDefaults` route via a channel (more code for no gain).

## 9. Keychain access

- **Decision**: Swift platform channel `freefocus/keychain` using `SecItemAdd`, `SecItemCopyMatching`, and `SecItemDelete` on a generic password item, service `app.todomato.todoist`, account `personal-token`, accessibility `kSecAttrAccessibleAfterFirstUnlock`.
- **Rationale**: Principle IX prefers about 50 lines to a package. The entry is a single item, so the Swift code is small. The service name keeps the source doc's identifier (see the naming assumption in the spec); it is internal and invisible to users.
- **Alternatives rejected**: `flutter_secure_storage` (a package for one item; it also widens the supply chain).

## 10. Local notifications and sound

- **Decision**: Swift channel `freefocus/notifications` wrapping `UNUserNotificationCenter`: request authorization, schedule a `UNTimeIntervalNotificationTrigger` for the interval end, cancel by identifier, and play `chime.caf` when the sound setting is on.
- **Rationale**: Principle IX. Scheduling is needed even while the app is suspended (FR-034), and the Swift API covers it directly.
- **Alternatives rejected**: `flutter_local_notifications` (a larger package with more platform surface than this use needs). Re-check: if its macOS scheduling is verified and it is smaller, the team may swap it, with a note in this section.

## 11. Launch at login

- **Decision**: Swift channel `freefocus/login_item` with `SMAppService.mainApp.register()`, `unregister()`, and `status`. Settings reads the real status when it opens (FR-070).
- **Rationale**: macOS 13+ API, no package needed. Matches the source doc.
- **Alternatives rejected**: `launch_at_startup` package (uses the legacy launch-agent mechanism).

## 12. HTTP client for Todoist

- **Decision**: `dart:io` `HttpClient` inside `TodoistHttp`, behind a small `HttpTransport` interface so tests inject responses without a network.
- **Rationale**: Principle IX. The API surface is four endpoints. Timeout 10 s, retry on 429 with 2, 4, and 8 s waits (FR-021), and no retries on other errors.
- **Alternatives rejected**: `package:http` (a standard package but unnecessary here; the interface already makes it replaceable for tests); `dio` (heavier).

## 13. Todoist API v1 (verified)

Verified 2026-11-20 against the official OpenAPI document at `https://developer.todoist.com/openapi.json` and the reference at `https://developer.todoist.com/api/v1/`. Base URL `https://api.todoist.com/api/v1`. Full details in [contracts/todoist-endpoints.md](contracts/todoist-endpoints.md).

- **Auth**: `Authorization: Bearer <token>`. The token is the personal token from the Integrations settings. The docs' examples are 40 hexadecimal characters, which matches FR-015. **Open check**: confirm the personal-token format against the live settings page during the first manual run (T001).
- **Current user**: `GET /user` returns `UserJSON` with `full_name`. Used for the connect check and the account name (FR-016, FR-017).
- **Due tasks**: `GET /tasks/filter?query=today%20%7C%20overdue` (the query value must be encoded). Paginated: response has `results` and `next_cursor`. The spec's "items" key is wrong; the key is `results`.
- **Projects**: `GET /projects` paginated (`results`, `next_cursor`). Project colour is a string name (`color`, e.g. `charcoal`), so the app maps names to hex (FR-009).
- **Close**: `POST /tasks/{task_id}/close`. Returns no body (`null`). Regular tasks move to history. Recurring tasks advance to their next occurrence (FR-044, User Story 3 scenario 6). The app refetches after closing.
- **Task fields used**: `id`, `content`, `project_id`, `priority` (1 normal to 4 urgent), `due` (object: `date`, `timezone`, `is_recurring`), `duration` (object: `amount`, `unit` where unit is `minute` or `day`), `checked`, `section_id`, `parent_id`, `order_key`.
- **Due time detection**: a task is time-blocked when `due.date` contains a time component (`YYYY-MM-DDTHH:mm...`). Its range runs from that start for `duration.amount` minutes when `duration.unit` is `minute`.
- **Errors**: 401 carries `error_code` 477 and `WWW-Authenticate: Bearer error="invalid_token"`. Map to `unauthorized` (FR-020). 429 is rate limiting (FR-021). **Open check**: the "Request limits" page was not read; confirm its 429 behaviour during T001.
- **Branding**: the Todoist docs require "not created by, affiliated with, or supported by Todoist" and forbid "Todoist" as the primary app name. Matches FR-082 and FR-083.

## 14. Fonts and licences

- **Decision**: Bundle Funnel Sans, Literata (variable, opsz axis), and Google Sans Code in `assets/fonts/`, declared in `pubspec.yaml` `fonts:` and registered for macOS with `ATSApplicationFontsPath`.
- **Licence**: The design file's header states all three are under the SIL Open Font License 1.1, with copyright notices. This resolves the source doc's open licence item for Google Sans Code. The licence texts go in `assets/fonts/LICENSES/`.
- **Fallback**: SF Mono for numeric faces if a face is removed, as the spec's assumption says.

## 15. Localisation

- **Decision**: ARB file `lib/l10n/app_en.arb` with `flutter_localizations` (SDK) and `intl` (pinned), generated by `gen-l10n`.
- **Rationale**: FR-004 and the source doc's "keep all user-facing strings in ARB files". `intl` is required by the generated code, so it is the one justified addition.

## 16. Testing approach

- **Decision**: `flutter_test` (SDK) for unit and widget tests. No mocking package: the `HttpTransport` and channel interfaces are implemented by hand in `test/support/`.
- **Rationale**: Principle IX. Constitution VII requires unit tests for `TimerEngine`, provider adapters with mocked HTTP, and the session store, plus widget tests for shared components.

## 17. Continuous integration

- **Decision**: GitHub Actions workflow `.github/workflows/ci.yml` on macOS runners running `flutter analyze`, `flutter test`, and the build.
- **Rationale**: Constitution VII: "CI MUST pass before any change merges". No CI exists yet, so creating it is part of the first implementation tasks.

## Open items carried into tasks

| Item | Owner task | Gate |
| --- | --- | --- |
| Upgrade local Flutter to 3.50+ | T000 | Before any build |
| `window_manager` 0.7.0 with `desktop_multi_window` 0.3.1 works without the git fork | T000 spike | Before secondary windows are built |
| Personal token format and the 429 back-off rule on the live Todoist settings and "Request limits" page | T001 | Before the Todoist client is merged |
| Confirm `tray_manager` right-click menu triggers with `ContextMenuTrigger.rightClicked` | T000 spike | Before the menu bar is built |
