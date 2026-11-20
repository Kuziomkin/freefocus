# Quickstart: Free Focus v1 validation

**Spec**: [spec.md](spec.md) | **Plan**: [plan.md](plan.md)

This guide proves v1 works end to end. Run it after the build passes CI. Each step names the acceptance scenario it checks.

## Prerequisites

1. macOS 14.0 or later, on a clean user account for the first-launch checks.
2. Flutter stable 3.50 or later (Dart 3.14), Xcode with command-line tools.
3. A Todoist account with at least three tasks due today, one of them time-blocked and one recurring, and one overdue task.
4. A Todoist personal API token from Settings › Integrations › Developer.

## Setup

```bash
flutter pub get
flutter gen-l10n
flutter analyze
flutter test
flutter run -d macos
```

Expected: analyze reports no issues; all tests pass; the app starts with no Dock icon and a menu bar item.

## Validation scenarios

### 1. First launch and connect (US1)

1. Launch with no stored token. Expect the Connect window only.
2. Paste `abc` and press Connect. Expect the rejected message and no network call (check the diagnostic log has no request line).
3. Paste a 40-character value that is not a real token. Expect the rejected state.
4. Paste the real token and press Connect. Expect "✓ Connected as {name} · Start", the window closes, and the popover lists today's tasks.

### 2. Token location (FR-018, SC-012)

1. Search the app container and `UserDefaults` for the token string. Expect no matches.
2. Confirm the Keychain item `app.todomato.todoist` / `personal-token` exists.

### 3. Timer, sleep, and relaunch (US2, SC-002, SC-003)

1. Start focus. Note the remaining time.
2. Put the Mac to sleep for 5 minutes, then wake it. Expect remaining time 5 minutes lower.
3. Start focus, quit with the popover Quit control, relaunch. Expect the same remaining time within 1 second.

### 4. Plus, notifications, and cycle (US2, SC-004, SC-005)

1. Press "+5". Expect the total to grow by exactly 5 minutes and the progress bar to shrink.
2. Let an interval end with the app in the background. Expect the notification at the stored end time.
3. Complete four focus intervals. Expect a long break, then the cycle label returns to "1 of 4".

### 5. Skip (SC-006)

1. Skip after 30 seconds. Expect no interrupted session and no cycle change.
2. Skip after 60 seconds. Expect one interrupted session in the Report export and no cycle change.

### 6. Task completion (US3, SC-007)

1. Click the checkbox of the recurring task. Expect it to strike through at once and reappear with its next date after a refetch.
2. Click another task's checkbox with the network disabled. Expect the strike-through to roll back with an inline error.

### 7. Offline (SC-008)

1. Disable the network and open the popover. Expect the cached list and "Couldn't reach Todoist. Showing tasks from {time}."
2. Revoke the token in Todoist and refresh. Expect "Todoist disconnected" with Reconnect.

### 8. Mini window (US4, SC-009)

1. Open the mini window. Switch to a full-screen app. Expect the window to stay visible.
2. Move it, quit, relaunch. Expect it at the same position.
3. Start a short break. Expect the teal colour set and "· next: {task title}".

### 9. Report and CSV (US5, SC-010)

1. Open the Report. Compare the stats with the session data in the database.
2. Export CSV. Expect the header row exactly as in FR-064.

### 10. Settings and delete (US6, FR-079)

1. Change focus length to 1 (clamped to 5). Expect the value to persist after relaunch.
2. Toggle "Launch at login". Expect the toggle to match the login item status after reopening Settings.
3. Choose Delete all data and confirm. Expect no sessions, no cached tasks, no token, and the Connect window.

### 11. Accessibility and brand (SC-011, FR-082, FR-083)

1. Turn on VoiceOver and move through the mini window, Settings, and popover. Expect every icon-only button to be announced with a label.
2. Check the Settings and Connect windows for the Doist disclaimer line.
3. Confirm the app name and bundle display name contain no "Todoist".
