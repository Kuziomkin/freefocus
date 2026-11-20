# Data Model: Free Focus v1

**Date**: 2026-11-20 | **Spec**: [spec.md](spec.md) | **Research**: [research.md](research.md)

This model is derived from the spec's Key Entities and FR-077 to FR-079. Storage is SQLite (sessions and completed tasks, cached tasks and projects) plus `SharedPreferencesAsync` (settings and timer state). The token is never stored in either.

## Entity: FocusTask (provider-neutral)

Normalised task. Screens, the timer, and storage use only this type (Principle II).

| Field | Type | Notes |
| --- | --- | --- |
| `id` | String | Provider task ID. Opaque to the app. |
| `title` | String | Single line; truncated with a tail ellipsis in the UI. |
| `projectId` | String? | Provider project ID. Null if no project. |
| `projectName` | String? | Cached from the project list. |
| `projectColor` | ColorName? | Provider colour name, mapped to hex by the provider layer. |
| `priority` | Priority | Enum `p1`..`p4` (p1 = highest). Mapping in research §13. |
| `due` | DateTime? | Local date-time for time-blocked tasks; date-only otherwise. |
| `isTimeBlocked` | bool | True when `due` includes a time component. |
| `duration` | Duration? | Present for time-blocked tasks with a duration in minutes. |
| `isRecurring` | bool | From `due.is_recurring`. |
| `order` | int | Provider order, used for the stable sort after time-blocked tasks. |

**Validation**: `id` non-empty. `title` trimmed; empty titles are shown as "Untitled task". `duration` > 0 minutes.

**Sort rule (FR-008)**: time-blocked tasks by `due` ascending, then all others by `order`.

## Entity: Project

| Field | Type | Notes |
| --- | --- | --- |
| `id` | String | |
| `name` | String | |
| `colorName` | ColorName | Unknown names map to a neutral grey (spec edge case). |
| `fetchedAt` | DateTime | Cache expiry is 24 h (FR-013). |

## Entity: FocusSession

Matches the spec's Key Entities and the CSV columns in FR-064.

| Field | Type | Constraints |
| --- | --- | --- |
| `id` | String (UUID) | Primary key. |
| `todoistTaskId` | String? | Null when no task was selected. |
| `taskTitle` | String | Snapshot at the time of the session. |
| `projectName` | String? | Snapshot. |
| `projectColor` | String? | Snapshot colour name. |
| `sessionType` | enum | `focus`, `short_break`, `long_break`. |
| `plannedDurationSec` | int | Planned length plus extensions (FR-028). |
| `actualDurationSec` | int | Time actually run. |
| `startedAt` | DateTime (UTC) | |
| `endedAt` | DateTime (UTC) | `endedAt >= startedAt`. |
| `status` | enum | `completed` or `interrupted`. Interrupted only if the interval ran 60 s or more (FR-031). |

**Indexes**: `startedAt` (Report range queries and the CSV export order); `todoistTaskId` (per-task Report totals).

**Rules**:
- A completed session is written when an interval reaches zero (FR-029).
- An interrupted session is written on Skip only when `actualDurationSec >= 60` (FR-031).
- The Report's "Focused" total sums `actualDurationSec` of `status = completed` and `sessionType = focus`.
- The "Pomodoros" count is the number of `completed` focus sessions.

## Entity: CompletedTask

| Field | Type | Constraints |
| --- | --- | --- |
| `todoistTaskId` | String | Primary key. |
| `title` | String | Snapshot. |
| `completedAt` | DateTime (UTC) | Written when a task is closed from Free Focus (FR-078). |

The Report's "Tasks completed" count is the number of rows whose `completedAt` falls in the selected week.

## Entity: TimerState

Persisted in `SharedPreferencesAsync` on every change (FR-026).

| Field | Type | Notes |
| --- | --- | --- |
| `mode` | enum | `focus`, `shortBreak`, `longBreak`. |
| `isRunning` | bool | |
| `endDate` | DateTime? | Set while running. |
| `remaining` | Duration | Authoritative while paused. |
| `plannedDuration` | Duration | Set when the interval is created. |
| `extraDuration` | Duration | Sum of "+N" presses for this interval. |
| `completedInCycle` | int | 0 to `pomodorosPerCycle`. |
| `currentTaskId` | String? | |
| `intervalStartedAt` | DateTime? | First start of this interval. |

**State transitions** (Timer engine, spec rules 1 to 9):

```text
focus (paused, full) --start--> focus (running) --pause--> focus (paused)
focus (running) --reaches 0--> break (session saved; cycle +1; auto-start)
break (running) --reaches 0--> focus (paused, full; waits for user)
any --skip (>=60 s)--> next mode (interrupted session saved; cycle unchanged)
focus when completedInCycle == pomodorosPerCycle --> longBreak; cycle resets after the long break
```

## Entity: Settings

Stored in `SharedPreferencesAsync`. Defaults and ranges from FR-068 and FR-070.

| Key | Type | Default | Range |
| --- | --- | --- | --- |
| `focusMinutes` | int | 25 | 5–120 |
| `shortBreakMinutes` | int | 5 | 1–30 |
| `longBreakMinutes` | int | 15 | 5–60 |
| `pomodorosPerCycle` | int | 4 | 2–8 |
| `extendMinutes` | int | 5 | 1–30 |
| `playSound` | bool | true | |
| `miniOnTop` | bool | true | |
| `launchAtLogin` | bool | true | Read from the login-item status on open, not trusted from storage. |
| `autoOpenMini` | bool | true | No visible control in v1 (FR-052). |
| `miniFrame` | Rect? | null | Last mini window position (FR-051). |

Changes apply to the next interval only (FR-033).

## Entity: ProviderConnection

| Field | Type | Notes |
| --- | --- | --- |
| `providerId` | String | `todoist` in v1. |
| `accountName` | String | From `GET /user` `full_name`. |
| `tokenSuffix` | String | Last four characters, for display only. |
| `state` | enum | `connected`, `disconnected`, `unreachable`. |

The token lives only in the Keychain under `app.todomato.todoist` / `personal-token` (research §9). `ProviderConnection` stores no token.

## Local cache: task list

- Stored as JSON in Application Support (`tasks-cache.json`), with the fetch time, so the popover can render offline (FR-012, FR-022).
- Replaced after each successful fetch. Never written with the token.

## Delete all data (FR-079)

Removes, in one transaction: all `FocusSession` and `CompletedTask` rows; the `tasks-cache.json` file; the `projects` cache; the `TimerState` keys; and the Keychain token. Then sets `state = disconnected` and returns to the Connect window.

## Diagnostic log (FR-018)

- A rotating local text file in Application Support, capped at 512 KB, with two files kept.
- Lines contain timestamps, event names, HTTP status codes, and error categories only. Never the token, task titles, or project names.
- Exported by the user from Settings through a save panel (no automatic upload).

## Schema version

`PRAGMA user_version = 1`. Migrations are numbered; v1 is the first release, so no migration exists yet.
