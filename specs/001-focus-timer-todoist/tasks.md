# Tasks: Free Focus v1 — Menu Bar Focus Timer for Todoist

**Input**: Design documents from `/specs/001-focus-timer-todoist/`

**Prerequisites**: plan.md (required), spec.md (required for user stories), research.md, data-model.md, contracts/, quickstart.md

**Tests**: Included. Constitution VII requires unit tests for `TimerEngine`, every provider adapter (mocked HTTP), and the session store, plus widget tests for shared components. Constitution IV requires timer tests to be written first and to fail before the engine exists.

**Organization**: Tasks are grouped by user story (spec.md priorities) so each story can be implemented and tested independently.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies on incomplete tasks)
- **[Story]**: US1 to US6, mapped to spec.md user stories. Setup, Foundational, and Polish carry no story label.
- Every task names an exact file path.

## Path Conventions

Single Flutter project at the repository root with a `macos/` runner. Paths follow the Project Structure in plan.md.

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Clear the two plan-level blockers, then create the project and CI.

- [ ] T000 Upgrade the local Flutter SDK to stable 3.50 or later (Dart 3.14), then run `flutter pub add tray_manager:0.9.0 window_manager:0.7.0 desktop_multi_window:0.3.1 sqlite3:3.9.1 shared_preferences:2.5.6` and `flutter pub add intl` in pubspec.yaml. Spike: build a throwaway window with `window_manager` 0.7.0 plus `desktop_multi_window` 0.3.1 without the git fork. Record the outcome and any required pin in specs/001-focus-timer-todoist/research.md §4 and §5. Stop and escalate if the fork is required.
- [ ] T001 Verify the Todoist personal-token format (40 hexadecimal characters) on the live Integrations › Developer page and read the "Request limits" page for the 429 rule. Record the results in specs/001-focus-timer-todoist/research.md §13 and contracts/todoist-endpoints.md. Update FR-015 and the backoff values in the spec if they differ.
- [ ] T002 Create the Flutter project for macOS with `flutter create --platforms=macos --org app.freefocus --project-name free_focus .` at the repository root. Set the minimum macOS deployment target to 14.0 in macos/Runner/Configs/AppInfo.xcconfig and macos/Podfile. Set the bundle display name to "Free Focus" in macos/Runner/Info.plist.
- [ ] T003 [P] Configure static analysis in analysis_options.yaml with `strict-casts`, `strict-inference`, and `strict-raw-types`, and add the `prefer_const_constructors` lint.
- [ ] T004 [P] Create the CI workflow in .github/workflows/ci.yml on `macos-14` that runs `flutter pub get`, `flutter gen-l10n`, `flutter analyze`, `flutter test`, and `flutter build macos --debug`. Pin the Flutter version to the one used in T000.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Tokens, shared components, the timer engine, storage, and the provider boundary. Every user story depends on these.

**CRITICAL**: No user story work can begin until this phase is complete.

### Design tokens and theme

- [ ] T005 [P] Create every colour, text style, radius, shadow, and spacing value from the spec's Design Tokens section in lib/theme/tokens.dart. Use only `static const` values. Include the priority ring colours and the three mode colour sets (focus, short break, long break).
- [ ] T006 [P] Bundle Funnel Sans (400, 500, 600), Literata (600, variable opsz), and Google Sans Code (500, 600) under assets/fonts/. Copy the OFL licence texts into assets/fonts/LICENSES/. Declare the families in pubspec.yaml under `flutter: fonts:`.
- [ ] T007 Register the bundled fonts for macOS by adding `ATSApplicationFontsPath` pointing to the bundled font folder in macos/Runner/Info.plist. Depends on T006.
- [ ] T008 [P] Create the light-only `ThemeData` built from tokens in lib/theme/theme.dart. Set `themeMode: ThemeMode.light` on every app root.
- [ ] T009 [P] Create the localisation setup: lib/l10n/app_en.arb (initial strings only), l10n.yaml at the repository root, and `flutter_localizations` in pubspec.yaml.

### Shared components (each has a widget test first, per Constitution VII)

- [ ] T010a [P] Write failing widget test for `PrimaryButton` (normal, pressed, disabled, icon left) in test/widget/shared/primary_button_test.dart, then implement lib/ui/shared/primary_button.dart.
- [ ] T010b [P] Write failing widget test for `SecondaryButton` (normal and pressed fill) in test/widget/shared/secondary_button_test.dart, then implement lib/ui/shared/secondary_button.dart.
- [ ] T010c [P] Write failing widget test for `IconButton` (28×28 hit area, hover fill, semantic label required) in test/widget/shared/icon_button_test.dart, then implement lib/ui/shared/icon_button.dart.
- [ ] T010d [P] Write failing widget test for `ModePill` (focus, short break, long break labels and colours) in test/widget/shared/mode_pill_test.dart, then implement lib/ui/shared/mode_pill.dart.
- [ ] T010e [P] Write failing widget test for `TaskCheckbox` (ring colour per priority, done state) in test/widget/shared/task_checkbox_test.dart, then implement lib/ui/shared/task_checkbox.dart.
- [ ] T010f [P] Write failing widget test for `Toggle` (on and off, 150 ms animation) in test/widget/shared/toggle_test.dart, then implement lib/ui/shared/toggle.dart.
- [ ] T010g [P] Write failing widget test for `ProgressLine` (fraction, 4 pt and 3 pt heights) in test/widget/shared/progress_line_test.dart, then implement lib/ui/shared/progress_line.dart.
- [ ] T010h [P] Write failing widget test for `GroupedPanel` and `Eyebrow` in test/widget/shared/grouped_panel_test.dart, then implement lib/ui/shared/grouped_panel.dart and lib/ui/shared/eyebrow.dart.

### Timer engine (test first, Constitution IV)

- [ ] T010 [P] Write failing unit tests for start, pause, and resume using a fake clock in test/unit/timer/timer_engine_test.dart. Assert remaining time comes from `endDate` and never from tick counts (data-model.md, rules 1–3). Run `flutter test` and confirm failure before writing the engine.
- [ ] T011 [P] Write failing unit tests for "+N" in test/unit/timer/timer_extension_test.dart. Assert the remaining time, planned total, and running end date each grow by exactly N minutes in every mode (FR-028).
- [ ] T012 [P] Write failing unit tests for interval end in test/unit/timer/timer_cycle_test.dart. Assert a completed focus session is produced, the cycle count advances, a long break follows the configured cycle length, and the cycle resets after the long break (FR-029, FR-032).
- [ ] T013 [P] Write failing unit tests for Skip in test/unit/timer/timer_skip_test.dart. Assert an interrupted session is produced only at 60 s or more and the cycle count never changes (FR-031).
- [ ] T014 [P] Write failing unit tests for restoring state after relaunch in test/unit/timer/timer_restore_test.dart. Assert an interval whose end date has passed finishes immediately on restore (FR-026, User Story 2 scenario 9).
- [ ] T015 Implement the pure-Dart timer state and engine in lib/core/timer/timer_state.dart and lib/core/timer/timer_engine.dart so T010 to T014 pass. The engine imports no UI, provider, or platform code. Depends on T010 to T014.
- [ ] T016 Implement timer state persistence in lib/core/settings/timer_state_store.dart: write on every change, read on launch. Test it in test/unit/settings/timer_state_store_test.dart. Depends on T015.

### Storage

- [ ] T017 [P] Write failing unit tests for the database opener in test/unit/store/database_test.dart: WAL journal mode, busy timeout set, `PRAGMA user_version = 1`, and the schema from data-model.md (indexes on `startedAt` and `todoistTaskId`).
- [ ] T018 Implement lib/core/store/database.dart to pass T017. Use `sqlite3` and create the FocusSession and CompletedTask tables with the constraints in data-model.md. Depends on T017.
- [ ] T019 [P] Write failing unit tests for session insert, completed-task insert, and "delete all" in test/unit/store/session_store_test.dart.
- [ ] T020 Implement lib/core/store/session_store.dart so T019 passes. Depends on T018, T019.
- [ ] T021 [P] Write failing unit tests for the settings store in test/unit/settings/settings_store_test.dart: default values, clamping to the ranges in FR-068 on write, and rejection of non-digit input.
- [ ] T022 Implement lib/core/settings/settings_store.dart with `SharedPreferencesAsync` (not the cached API, per research §8) so T021 passes. Depends on T021.

### Platform channels and credentials

- [ ] T023 [P] Implement the Keychain channel in macos/Runner/KeychainChannel.swift (`SecItemAdd`, `SecItemCopyMatching`, `SecItemDelete`; service `app.todomato.todoist`, account `personal-token`, accessibility `kSecAttrAccessibleAfterFirstUnlock`), and register it in macos/Runner/MainFlutterWindow.swift.
- [ ] T024 [P] Write failing unit test for the Dart Keychain client in test/unit/platform/keychain_channel_test.dart using a fake channel, then implement lib/core/platform/keychain_channel.dart and lib/core/credentials/credential_store.dart.
- [ ] T025 [P] Create macos/Runner/CHANNELS.md documenting every channel name, method, arguments, and return value. Add an entry for the Keychain channel now; later channels add their entries in their own tasks.

### Task model and provider boundary

- [ ] T026 [P] Create lib/core/tasks/focus_task.dart and lib/core/tasks/project.dart with the fields and validation rules in data-model.md, including the sort rule (time-blocked by due time first, then provider order) in a function `sortTasks`.
- [ ] T027 [P] Write failing unit test for `sortTasks` in test/unit/tasks/sort_tasks_test.dart covering time-blocked tasks first, then provider order (FR-008).
- [ ] T028 Implement `sortTasks` so T027 passes in lib/core/tasks/focus_task.dart. Depends on T026, T027.
- [ ] T029 [P] Create the `TaskProvider` interface, `ProviderAccount`, and `ProviderException` hierarchy in lib/core/tasks/task_provider.dart exactly as contracts/task-provider.md specifies.
- [ ] T030 [P] Create lib/core/tasks/provider_registry.dart with an empty `providerFactories` list of type `List<TaskProvider Function(CredentialStore)>`. T049 adds `TodoistProvider.new` in US1.
- [ ] T031 [P] Create the `FakeProvider` test double in test/support/fake_provider.dart implementing `TaskProvider` with scripted results and call recording (Principle II).
- [ ] T032 [P] Write failing tests for the offline cache in test/unit/tasks/task_cache_test.dart: write after a successful fetch, read back with fetch time, never store a token.
- [ ] T033 Implement lib/core/tasks/task_cache.dart so T032 passes, writing `tasks-cache.json` in Application Support. Depends on T032.

### Diagnostic log

- [ ] T034 [P] Write failing unit test for the diagnostic log in test/unit/log/diagnostic_log_test.dart: writes lines with timestamp and category, redacts any string matching the token pattern, caps at 512 KB, and keeps two files.
- [ ] T035 Implement lib/core/log/diagnostic_log.dart so T034 passes. Depends on T034.

### Foundation window host

- [ ] T036 Create lib/main.dart, which reads the window argument and routes to the primary root or a secondary root, and a placeholder lib/app/primary_window.dart and lib/app/secondary_windows.dart. Depends on T008.

**Checkpoint**: Foundation ready. Timer engine, storage, settings, the provider boundary, and the shared components are in place and tested.

---

## Phase 3: User Story 1 - Connect Todoist and see today's tasks (Priority: P1) 🎯 MVP

**Goal**: A new user pastes a token, the app verifies it with Todoist, stores it in the Keychain, and shows today's and overdue tasks in the popover.

**Independent Test**: On a clean account, launch the app, paste a valid token, and confirm the popover lists today's tasks. A bad token shows the rejected state with no network call (spec User Story 1 scenarios 1–5).

### Tests for User Story 1 (write first, must fail)

- [ ] T037 [P] [US1] Write failing unit tests for token-format validation in test/unit/providers/todoist/token_format_test.dart: trims whitespace, accepts exactly 40 hexadecimal characters, rejects others without calling the transport (FR-015).
- [ ] T038 [P] [US1] Write failing unit tests for the Todoist HTTP client in test/unit/providers/todoist/todoist_client_test.dart using a fake transport: 200 success, 401 and 403 mapped to `ProviderUnauthorized`, timeout after 10 s and other failures mapped to `ProviderUnreachable`, 429 retried after 2, 4, and 8 s then abandoned (FR-016, FR-021).
- [ ] T039 [P] [US1] Write failing unit tests for decoding task pages in test/unit/providers/todoist/todoist_models_test.dart: `results` and `next_cursor` pagination loop, `priority` mapping 4→p1 through 1→p4, `due` with and without time, `duration` with `unit` `minute`, colour name mapping with unknown names falling back to grey (contracts/todoist-endpoints.md).
- [ ] T040 [P] [US1] Write failing unit tests for `TodoistProvider.connect`, `fetchTodayTasks`, `fetchProjects`, and `completeTask` in test/unit/providers/todoist/todoist_provider_test.dart using the fake transport.
- [ ] T041 [P] [US1] Write failing widget test for the Connect window states (empty, typed, checking, rejected, network error, success) in test/widget/connect/connect_screen_test.dart using `FakeProvider`.
- [ ] T042 [P] [US1] Write failing widget test for the popover task list states (loading skeleton, no tasks today, cached with offline line, disconnected with Reconnect) in test/widget/popover/task_list_states_test.dart.

### Implementation for User Story 1

- [ ] T043 [P] [US1] Implement the `dart:io` HTTP transport behind the `HttpTransport` interface in lib/providers/todoist/todoist_client.dart, with a 10 s timeout, Bearer header, and no logging of headers (research §12). Test double in test/support/fake_http_transport.dart.
- [ ] T044 [P] [US1] Implement token-format validation in lib/providers/todoist/todoist_token.dart so T037 passes.
- [ ] T045 [US1] Implement the decoding models in lib/providers/todoist/todoist_models.dart so T039 passes, using the field list in contracts/todoist-endpoints.md. Depends on T043.
- [ ] T046 [P] [US1] Implement lib/providers/todoist/todoist_priority.dart (priority mapping, research §13) and lib/providers/todoist/todoist_colors.dart (`TodoistColor` enum mapping colour names to the published hex values).
- [ ] T047 [US1] Implement `TodoistClient` with `fetchUser`, `fetchTodayTasks` (`GET /tasks/filter?query=today%20%7C%20overdue`), `fetchProjects` (`GET /projects`), and `closeTask` (`POST /tasks/{id}/close`) in lib/providers/todoist/todoist_client.dart, with the retry and error rules from FR-021 and FR-016. Depends on T038, T043, T045.
- [ ] T048 [US1] Implement lib/providers/todoist/todoist_provider.dart: the `TaskProvider` implementation that delegates to `TodoistClient`, converts models to `FocusTask` and `Project`, and maps errors to the provider exception hierarchy. Depends on T040, T046, T047.
- [ ] T049 [US1] Register the provider by adding `TodoistProvider.new` to `providerFactories` in lib/core/tasks/provider_registry.dart. Depends on T048.
- [ ] T050 [US1] Implement the Connect screen in lib/ui/connect/connect_screen.dart with the layout and states from spec Screen 5, using only tokens and shared components. Include the privacy line and the Doist non-affiliation line (FR-076, FR-083). Depends on T041, T010a to T010h.
- [ ] T051 [US1] On success, store the token through lib/core/credentials/credential_store.dart, store the account name, and request notification permission only after connect (FR-017). Depends on T024, T050.
- [ ] T052 [US1] Implement the secondary window for Connect in lib/app/secondary_windows.dart using `desktop_multi_window` and `window_manager`: 420 pt wide, not resizable, centred, minimise and zoom disabled. Depends on T036, T050.
- [ ] T053 [US1] Register the macOS plugins for secondary engines in macos/Runner/MainFlutterWindow.swift per the `desktop_multi_window` README. Depends on T000.
- [ ] T054 [US1] Implement the task list and the `TaskRow` component in lib/ui/popover/task_list.dart and lib/ui/shared/task_row.dart (checkbox, title, meta line with time range and project, selected state) to satisfy T042. Depends on T042, T010e.
- [ ] T055 [US1] Implement the offline and error states in lib/ui/popover/task_list.dart: skeleton rows, "Nothing due today", the offline line with the cache time, and the "Todoist disconnected" state with Reconnect (FR-022, FR-020). Depends on T054.
- [ ] T056 [US1] Implement the connection-state controller in lib/core/tasks/task_controller.dart that loads the cache first, then fetches on popover open if older than 60 s (FR-023), and on wake (FR-023). Depends on T033, T048.
- [ ] T057 [US1] Add the "Connect Todoist" button in the popover empty state that reopens the Connect window (FR-014). Depends on T052.
- [ ] T058 [US1] Update CHANNELS.md with the notification-permission call used during connect. Depends on T025.

**Checkpoint**: User Story 1 works. A user can connect and see today's tasks. Independent test: quickstart.md section 1 and section 2.

---

## Phase 4: User Story 2 - Run focus and break intervals from the menu bar (Priority: P1)

**Goal**: The menu bar shows the remaining time; the popover runs focus and break intervals in a fixed cycle with Start, Pause, Resume, "+N", Skip, notifications, and sleep-safe timing.

**Independent Test**: Start focus, observe the countdown, pause and resume, add time, complete intervals to a long break, and verify the cycle label and notification (spec User Story 2 scenarios 1–9).

### Tests for User Story 2 (write first, must fail)

- [ ] T059 [P] [US2] Write failing widget test for the popover header and cycle label (focus "1 of 4", break "up next: focus") in test/widget/popover/popover_header_test.dart.
- [ ] T060 [P] [US2] Write failing widget test for the popover controls (primary label per state, "+N" label from settings, skip icon with a semantic label) in test/widget/popover/popover_controls_test.dart.
- [ ] T061 [P] [US2] Write failing unit test for the menu bar label text (`MM:SS`, grey dot idle at 25:00, colour by mode) in test/unit/app/menu_bar_label_test.dart.
- [ ] T062 [P] [US2] Write failing unit test for the notification request scheduling in test/unit/platform/notifications_scheduler_test.dart: schedule on start and "+N", cancel on pause, skip, and reset, notification titles from FR-035.

### Implementation for User Story 2

- [ ] T063 [P] [US2] Implement the notifications channel in macos/Runner/NotificationsChannel.swift (`UNUserNotificationCenter` authorization, schedule with a time-interval trigger, cancel by identifier, play `chime.caf` when enabled) and register it in macos/Runner/MainFlutterWindow.swift. Add the entry to macos/Runner/CHANNELS.md.
- [ ] T064 [P] [US2] Implement lib/core/platform/notifications_channel.dart and the scheduler in lib/core/platform/notifications_scheduler.dart so T062 passes. Depends on T062, T063.
- [ ] T065 [P] [US2] Add the chime sound file assets/sounds/chime.caf and declare it in pubspec.yaml.
- [ ] T066 [P] [US2] Implement the system-event channel for wake from sleep in macos/Runner/SystemEventsChannel.swift (`NSWorkspace.didWakeNotification`), register it in macos/Runner/MainFlutterWindow.swift, and document it in macos/Runner/CHANNELS.md.
- [ ] T067 [P] [US2] Implement lib/core/platform/system_events_channel.dart that emits a wake event to the timer controller. Depends on T066.
- [ ] T068 [US2] Implement the timer controller in lib/core/timer/timer_controller.dart: reads durations from settings at interval creation (FR-033), persists state on every change (T016), schedules notifications (T064), records sessions (T020), re-evaluates on wake (T067), and finishes overdue intervals on launch. Depends on T015, T016, T022, T064, T067.
- [ ] T069 [US2] Implement the menu bar item in lib/app/menu_bar_item.dart using `tray_manager` (research §3): a status item whose label is the dot and `MM:SS`, updated once per second, with a right-click menu containing Quit (FR-001). Set the context-menu trigger to `ContextMenuTrigger.rightClicked`. Depends on T061, T068.
- [ ] T070 [US2] Implement the primary window in lib/app/primary_window.dart: a 360 pt frameless popover, opened by the tray item, closed on click outside and Escape (FR-038, FR-047). Use `window_manager` for the frameless style. Depends on T036, T069.
- [ ] T071 [US2] Implement the popover header and timer display in lib/ui/popover/popover_header.dart and lib/ui/popover/timer_display.dart, using `ModePill`, the three-part timer with the coloured colon, and `ProgressLine`, to satisfy T059. Depends on T059, T010d, T010g.
- [ ] T072 [US2] Implement the popover controls in lib/ui/popover/popover_controls.dart: the primary button (Start focus, Pause, Resume), the "+N" button, and the icon-only skip button with a semantic label, satisfying T060. Depends on T060, T010a, T010b, T010c.
- [ ] T073 [US2] Implement the Quit control in the popover footer in lib/ui/popover/popover_footer.dart with the focused-today total and the Report link placeholder (FR-040, FR-001). Depends on T070.
- [ ] T074 [US2] Implement the Space shortcut for start and pause in lib/ui/popover/popover_keyboard.dart (FR-047). Depends on T072.
- [ ] T075 [US2] Implement break colour switching in the popover (pill, progress, primary button, menu bar dot) using the mode colour set from tokens (FR-049). Depends on T071, T072.
- [ ] T076 [US2] Implement the auto-start rule in lib/core/timer/timer_controller.dart: breaks start automatically when focus ends, and focus waits paused at full length after a break (FR-030). Test it in test/unit/timer/timer_auto_start_test.dart. Depends on T068.

**Checkpoint**: User Stories 1 and 2 both work. The MVP is complete: connect, see tasks, run the timer from the menu bar. Independent test: quickstart.md sections 3 and 4.

---

## Phase 5: User Story 3 - See and complete tasks from the popover (Priority: P2)

**Goal**: The user picks the current task and closes tasks in Todoist from the popover, with optimistic updates and rollback.

**Independent Test**: Select a task, run an interval, close a different task, and confirm it is closed in Todoist. Revoke the network to check rollback (spec User Story 3 scenarios 1–6).

### Tests for User Story 3 (write first, must fail)

- [ ] T077 [P] [US3] Write failing unit tests for task selection in test/unit/timer/task_selection_test.dart: selecting a task sets the current task, and a running interval's session is attributed to the task current at interval start (User Story 3 scenario 2).
- [ ] T078 [P] [US3] Write failing unit tests for completion in test/unit/tasks/complete_task_test.dart using `FakeProvider`: optimistic strike-through, rollback on failure with an inline error, next open task becomes current when the current task is closed, a `CompletedTask` record is written (FR-044, FR-078).
- [ ] T079 [P] [US3] Write failing unit tests for the recurring-task refetch in test/unit/tasks/recurring_refetch_test.dart: after a successful close, the list is refetched (contracts/todoist-endpoints.md).
- [ ] T080 [P] [US3] Write failing widget test for `TaskRow` interactions (title click selects, checkbox click completes, selected row styling) in test/widget/popover/task_row_interaction_test.dart.

### Implementation for User Story 3

- [ ] T081 [US3] Implement task selection in lib/core/timer/timer_controller.dart so T077 passes. Depends on T068, T077.
- [ ] T082 [US3] Implement completion in lib/core/tasks/task_controller.dart: optimistic removal, call `completeTask`, record the `CompletedTask`, roll back on failure, refetch after success (T078, T079). Depends on T056, T078, T079.
- [ ] T083 [US3] In lib/ui/popover/task_list.dart, wire `TaskRow` title and checkbox taps to selection and completion, and add the inline error line above the list (FR-044). Satisfies T080. Depends on T054, T080, T082.
- [ ] T084 [US3] Implement the Return key on a focused row to select it in lib/ui/popover/popover_keyboard.dart (FR-047). Depends on T074, T083.
- [ ] T085 [US3] Implement popover refetch rules in lib/core/tasks/task_controller.dart: on open if older than 60 s, after a close, and on wake (FR-023). Depends on T082.

**Checkpoint**: User Story 3 works independently of the Report and Settings. Independent test: quickstart.md section 6.

---

## Phase 6: User Story 4 - Keep the timer visible in a mini window (Priority: P2)

**Goal**: A small floating mini window shows the countdown above all apps, including full-screen apps, remembers its position, and turns teal or blue on breaks.

**Independent Test**: Open the mini window, switch to a full-screen app, move it, quit and relaunch, and confirm the position and the break colour (spec User Story 4 scenarios 1–7).

### Tests for User Story 4 (write first, must fail)

- [ ] T086 [P] [US4] Write failing widget test for the mini window layout (top row, timer, controls, progress line) and the break variant with "· next: {task}" in test/widget/mini/mini_window_test.dart.
- [ ] T087 [P] [US4] Write failing unit test for mini frame persistence and first placement (top-right, 24 pt from the right, 12 pt below the menu bar) in test/unit/mini/mini_placement_test.dart.
- [ ] T088 [P] [US4] Write failing unit test for mini window open and close rules in test/unit/mini/mini_visibility_test.dart: open from popover, auto-open on timer start, closes on expand and Escape, stays open when an interval ends (FR-052 to FR-055).

### Implementation for User Story 4

- [ ] T089 [P] [US4] Implement the mini panel channel in macos/Runner/MiniPanelChannel.swift: `NSPanel` with `.nonactivatingPanel` and `.borderless`, level `.floating` or `.normal`, `collectionBehavior = [.canJoinAllSpaces, .fullScreenAuxiliary]`, `hidesOnDeactivate = false`, `isMovableByWindowBackground = true`. Register it and document it in macos/Runner/CHANNELS.md.
- [ ] T090 [P] [US4] Implement lib/core/platform/mini_panel_channel.dart with `setLevel`, `setFrame`, `show`, `hide`, and a frame-changed event. Depends on T089.
- [ ] T091 [US4] Implement the mini window UI in lib/ui/mini/mini_window.dart (288 × 108 pt, `timerMini` digits, ModePill-equivalent eyebrow, "+N" and play/pause buttons, bottom progress line) to satisfy T086. Depends on T086, T010a, T010b, T010g.
- [ ] T092 [US4] Implement mini frame persistence and first placement in lib/ui/mini/mini_placement.dart to satisfy T087, storing `miniFrame` via the settings store. Depends on T087, T022, T090.
- [ ] T093 [US4] Implement the open and close rules in lib/ui/mini/mini_visibility.dart so T088 passes, including the setting-driven `miniOnTop` level (FR-056). Depends on T088, T090.
- [ ] T094 [US4] Wire the popover mini-window button and the expand button into lib/ui/mini/mini_visibility.dart. Opening the mini closes the popover (FR-053). Depends on T093, T070.
- [ ] T095 [US4] Implement break colour switching in lib/ui/mini/mini_window.dart (FR-049). Depends on T091, T075.

**Checkpoint**: The mini window works independently. Independent test: quickstart.md section 8.

---

## Phase 7: User Story 5 - Review focus time in a weekly Report (Priority: P3)

**Goal**: A Report window shows the week's totals, a day bar chart, per-task time for a chosen day, and exports every session to CSV with the exact header.

**Independent Test**: Complete sessions across two days, open the Report, check totals and per-task times against the database, export the CSV, and check the header (spec User Story 5 scenarios 1–5).

### Tests for User Story 5 (write first, must fail)

- [ ] T096 [P] [US5] Write failing unit tests for weekly queries in test/unit/store/report_queries_test.dart: focused total (completed focus only), pomodoro count, tasks completed count, per-day totals, per-task totals for a day, empty week, and Monday as the first day (FR-058, FR-059).
- [ ] T097 [P] [US5] Write failing unit tests for CSV export in test/unit/store/csv_export_test.dart: exact header row from FR-064, all weeks included, UTC ISO 8601 timestamps, `session_type` and `status` values, RFC 4180 quoting for titles containing commas and quotes.
- [ ] T098 [P] [US5] Write failing widget test for the bar chart (seven bars, value label format H:MM, stub bars for empty days, today highlighted) in test/widget/report/report_bars_test.dart.
- [ ] T099 [P] [US5] Write failing widget test for the Report stats and empty state in test/widget/report/report_stats_test.dart.

### Implementation for User Story 5

- [ ] T100 [US5] Implement the report queries in lib/core/store/report_queries.dart so T096 passes. Depends on T020, T096.
- [ ] T101 [US5] Implement the CSV export in lib/core/store/csv_export.dart so T097 passes. Depends on T020, T097.
- [ ] T102 [P] [US5] Implement the save-panel channel in macos/Runner/FileExportChannel.swift (`NSSavePanel`, default name, UTF-8 write) and document it in macos/Runner/CHANNELS.md. Register it in macos/Runner/MainFlutterWindow.swift.
- [ ] T103 [P] [US5] Implement lib/core/platform/file_export_channel.dart. Depends on T102.
- [ ] T104 [US5] Implement the bar chart in lib/ui/report/report_bars.dart so T098 passes, with `CustomPainter`, no charting package (Principle IX). Depends on T098, T096.
- [ ] T105 [US5] Implement the stats strip and empty state in lib/ui/report/report_stats.dart so T099 passes. Depends on T099, T100.
- [ ] T106 [US5] Implement the Report window in lib/ui/report/report_screen.dart: week switcher (next disabled on the current week), day section with per-task rows and share bars, and the Export CSV button (FR-057 to FR-063). Depends on T104, T105, T103, T101.
- [ ] T107 [US5] Implement the Report window host in lib/app/secondary_windows.dart: 860 pt wide, min 520 pt, reused if already open (FR-066). Depends on T052, T106.
- [ ] T108 [US5] Wire the popover footer "Report" link to open the Report window in lib/ui/popover/popover_footer.dart. Depends on T073, T107.

**Checkpoint**: The Report works independently. Independent test: quickstart.md section 9.

---

## Phase 8: User Story 6 - Adjust timer lengths and general behaviour in Settings (Priority: P3)

**Goal**: Settings changes persist immediately, apply to the next interval only, and include Launch at login, delete all data, the diagnostic log export, and the Doist disclaimer.

**Independent Test**: Change each timer value and toggle, quit and relaunch, and confirm persistence. Run Delete all data and confirm the empty state (spec User Story 6 scenarios 1–6; FR-079).

### Tests for User Story 6 (write first, must fail)

- [ ] T109 [P] [US6] Write failing widget test for the Settings panels (Todoist, Timer, General) with the defaults and ranges from FR-068 in test/widget/settings/settings_screen_test.dart.
- [ ] T110 [P] [US6] Write failing unit test for "Delete all data" in test/unit/settings/delete_all_data_test.dart: removes sessions, completed tasks, the task cache, timer state, and the Keychain token, and sets the connection to disconnected (FR-079).
- [ ] T111 [P] [US6] Write failing unit test for launch-at-login status reads in test/unit/platform/login_item_channel_test.dart using a fake channel: status is read on open, not trusted from storage (FR-070).

### Implementation for User Story 6

- [ ] T112 [P] [US6] Implement the login-item channel in macos/Runner/LoginItemChannel.swift (`SMAppService.mainApp` register, unregister, status) and document it in macos/Runner/CHANNELS.md.
- [ ] T113 [P] [US6] Implement lib/core/platform/login_item_channel.dart so T111 passes. Depends on T111, T112.
- [ ] T114 [US6] Implement the Settings screen in lib/ui/settings/settings_screen.dart: Todoist panel with the connected name and token suffix (FR-019), Timer panel with number fields clamped on blur or Return, General panel with the three toggles, and the footer with the Doist disclaimer (FR-072). Satisfies T109. Depends on T109, T010f, T010h, T022, T113.
- [ ] T115 [US6] Implement "Delete all data" with a confirmation step in lib/ui/settings/delete_all_data.dart and lib/core/settings/delete_all_data.dart so T110 passes. Depends on T110, T020, T024, T033, T016.
- [ ] T116 [US6] Implement the diagnostic log export in Settings using the save-panel channel (`lib/core/platform/file_export_channel.dart`) and `lib/core/log/diagnostic_log.dart` (FR-018). Depends on T035, T103, T114.
- [ ] T117 [US6] Implement the Settings window host in lib/app/secondary_windows.dart: 580 pt wide, opened from the popover settings button and from ⌘, while the app is active, with `NSApp.activate` first (FR-071). Depends on T052, T114.
- [ ] T118 [US6] Wire the settings changes to the next interval only: the timer controller reads settings at interval creation, and the "+" labels update at once (FR-033). Depends on T068, T114.
- [ ] T119 [US6] Implement the launch-at-login toggle reading real status on open in lib/ui/settings/settings_screen.dart (FR-070). Depends on T113, T114.

**Checkpoint**: Settings works independently. Independent test: quickstart.md section 10.

---

## Phase N: Polish & Cross-Cutting Concerns

**Purpose**: Accessibility, brand compliance, end-to-end validation, and CI.

- [ ] T120 [P] Audit every icon-only control for a semantic label and add a widget test that fails for any unlabelled control in test/widget/accessibility/icon_labels_test.dart (FR-080, SC-011).
- [ ] T121 [P] Add a contrast test in test/unit/theme/contrast_test.dart that checks every text token against its background is at least 4.5:1 (FR-081).
- [ ] T122 [P] Add a brand check test in test/unit/brand/name_test.dart that fails if the display name, bundle name, or user-facing strings contain "Todoist" (FR-082).
- [ ] T123 [P] Add a lint check in test/unit/theme/no_hardcoded_values_test.dart that fails if any file under lib/ui contains a hex colour or a `fontSize` literal outside lib/theme (FR-084, Constitution VI).
- [ ] T124 Add a check in test/unit/providers/isolation_test.dart that no file outside lib/providers/todoist imports it (Principle II).
- [ ] T125 [P] Remove unused scaffold code and placeholders left by T036 in lib/app/ and lib/main.dart.
- [ ] T126 Run quickstart.md end to end on a clean macOS 14 account and record the results in specs/001-focus-timer-todoist/quickstart.md.
- [ ] T127 Confirm CI in .github/workflows/ci.yml (T004) passes on the default branch and runs analyze, unit tests, widget tests, and the build.
- [ ] T128 Add test/unit/privacy/token_leak_test.dart that scans lib/ for token-like values written to preferences, logs, or exports, and confirm the token is absent from `UserDefaults`, the app container, logs, and exports (SC-012, quickstart section 2).

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies. T000 and T001 must close before any feature code.
- **Foundational (Phase 2)**: Depends on Setup. Blocks every user story.
- **User Stories (Phase 3 onward)**: Each depends on Foundational. US1 and US2 are both P1; US2 can start once the timer engine and tray work are in place, and does not need US1's connect flow to be tested.
- **Polish (Phase N)**: Depends on the stories it audits.

### User Story Dependencies

- **US1 (P1)**: Foundational only. Delivers the connect flow and the popover task list.
- **US2 (P1)**: Foundational only. Builds its own popover window (T070) and does not depend on US1's network code. Its popover timer screens are developed alongside the US1 task list, and both are merged into the same primary window at the checkpoint.
- **US3 (P2)**: Depends on US1 (task list, provider) and US2 (timer controller for selection).
- **US4 (P2)**: Depends on US2 (timer controller, popover) and the shared components.
- **US5 (P3)**: Depends on the session store and US2 (sessions being recorded).
- **US6 (P3)**: Depends on the settings store and US1 (token and connection state for Delete all data).

### Within Each Story

- Tests are written first and must fail before implementation.
- Models before services, services before UI.
- Story complete before moving to the next priority.

### Parallel Opportunities

- Phase 2 shared-component tasks T010a to T010h are independent files and can run in parallel.
- Timer tests T010 to T014 are independent and can run in parallel; they must all fail before T015.
- Each story's test tasks marked [P] can run together, and so can its platform-channel tasks (macOS Swift files are separate).
- US5 and US6 can proceed in parallel once US2 is done.

---

## Parallel Example: User Story 2

```bash
# Launch the four failing tests together:
Task: "T059 popover header test in test/widget/popover/popover_header_test.dart"
Task: "T060 popover controls test in test/widget/popover/popover_controls_test.dart"
Task: "T061 menu bar label test in test/unit/app/menu_bar_label_test.dart"
Task: "T062 notification scheduler test in test/unit/platform/notifications_scheduler_test.dart"

# Then the independent platform channels:
Task: "T063 NotificationsChannel.swift"
Task: "T066 SystemEventsChannel.swift"
```

---

## Implementation Strategy

### MVP First (User Stories 1 and 2, both P1)

1. Phase 1: Setup (T000 and T001 first; these are blockers).
2. Phase 2: Foundational. The timer engine tests must fail before the engine exists.
3. Phase 3: US1 (connect and see tasks).
4. Phase 4: US2 (run the timer from the menu bar).
5. **STOP and VALIDATE**: run quickstart sections 1 to 4.
6. Deliver the MVP: a working timer on today's Todoist tasks, with no selection, completion, mini window, Report, or Settings yet.

### Incremental Delivery

1. Setup + Foundational → foundation ready.
2. US1 + US2 → MVP (above).
3. US3 → select and complete tasks.
4. US4 → mini window.
5. US5 → Report and CSV.
6. US6 → Settings, Delete all data, log export.
7. Polish → accessibility, brand, end-to-end run, CI.

---

## Notes

- [P] tasks touch different files and have no dependency on an incomplete task.
- Story labels map each task to its spec user story for traceability.
- The task IDs T000, T001, T004, and T010 to T014 are the IDs referenced in plan.md's Constitution Check and Phases section.
- Shared components use the letter suffixes T010a to T010h to keep the timer test IDs T010 to T014 that plan.md references. These are a deliberate deviation from the plain numbering in the template.
- Avoid: vague tasks, the same file edited in parallel, and cross-story dependencies beyond those listed above.

---

## Task summary

| Phase | Tasks | Count |
| --- | --- | --- |
| Setup | T000–T004 | 5 |
| Foundational | T005–T036 (including T010a–T010h) | 40 |
| US1 (P1, MVP) | T037–T058 | 22 |
| US2 (P1, MVP) | T059–T076 | 18 |
| US3 (P2) | T077–T085 | 9 |
| US4 (P2) | T086–T095 | 10 |
| US5 (P3) | T096–T108 | 13 |
| US6 (P3) | T109–T119 | 11 |
| Polish | T120–T128 | 9 |
| **Total** | | **137** |

Counts are per task line. The MVP scope is Phases 1 to 4 (US1 and US2).
