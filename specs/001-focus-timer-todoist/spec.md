# Feature Specification: Free Focus v1 — Menu Bar Focus Timer for Todoist

**Feature Branch**: `001-focus-timer-todoist`

**Created**: 2026-11-20

**Status**: Draft

**Input**: User description: "Use @docs/Free Focus — UI & implementation spec.md and @docs/FreeFocus — macOS focus timer for Todoist.html to create a specification of the app"

## Product Summary

Free Focus is a macOS menu bar Pomodoro timer. It shows the user's tasks due today from their existing task app, lets them time one task at a time, and lets them mark tasks done at the source. It has no account system, no backend, and no analytics. The only network traffic is to the connected task provider. Version 1 ships exactly one provider, Todoist, and five screens: menu bar popover, mini window, Report, Settings, and First launch.

The product is built so that other task providers can be added later without changing screens, the timer, or local storage. v1 does not ship them.

## Clarifications

### Session 2026-11-20

- Q: Where should the user find the Quit action, since the app has no Dock icon or main menu? → A: Both a Quit control in the popover footer and a right-click menu on the menu bar item.
- Q: Should users be able to delete all their local focus history and reset the app's stored data, and where should that control live? → A: A "Delete all data" action in Settings with a confirmation step, which also clears the cached tasks and disconnects Todoist.
- Q: When something goes wrong, should the app keep a local diagnostic log that the user can choose to export, or keep no logs at all? → A: Keep a small local log with no tokens or task content, exported manually by the user from Settings.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Connect Todoist and see today's tasks (Priority: P1)

A new user opens Free Focus for the first time. The app asks for their Todoist personal API token. After a valid token is entered, the user sees today's and overdue tasks next to the timer.

**Why this priority**: Nothing else in the app works without a connected task source. It is the entry point for every user.

**Independent Test**: On a clean macOS account, launch the app, paste a valid token, and confirm the popover opens with the user's tasks due today or overdue.

**Acceptance Scenarios**:

1. **Given** the app has never been connected, **When** it launches, **Then** a "Connect Todoist" window appears and the app does nothing else until a connection succeeds.
2. **Given** the Connect window is open, **When** the user pastes a value that is not 40 characters of hexadecimal after trimming whitespace, **Then** the window shows the rejected state with the error "Todoist didn't accept this token. Copy it again and paste the whole thing." and no network request is made.
3. **Given** a well-formed token that Todoist rejects, **When** the user presses Connect, **Then** the rejected state is shown, the Connect button is enabled again, and editing the field clears the error.
4. **Given** a valid token, **When** the user presses Connect, **Then** the button shows "✓ Connected as {name} · Start" for 1.2 seconds, the window closes, and the popover opens with today's tasks.
5. **Given** the connection succeeds, **When** the app finishes connecting, **Then** it requests notification permission. Permission is not requested at launch.

---

### User Story 2 - Run focus and break intervals from the menu bar (Priority: P1)

The user starts a 25-minute focus interval from the menu bar. The menu bar shows the remaining time and the timer moves through focus and break intervals in a fixed cycle, without the user having to reconfigure anything.

**Why this priority**: The timer is the core value of the product. A timer that drifts or loses state fails the product's purpose.

**Independent Test**: Start a focus interval, observe the countdown, pause, resume, and complete intervals until a long break occurs. Confirm the cycle count and mode changes match the rules below.

**Acceptance Scenarios**:

1. **Given** the app is idle with no timer started, **When** the user looks at the menu bar, **Then** it shows a grey dot and "25:00".
2. **Given** an idle timer, **When** the user presses "Start focus", **Then** the countdown starts from the configured focus length, the header reads "FOCUS", and the cycle label reads "1 of 4".
3. **Given** a running interval, **When** the user presses Pause and later Resume, **Then** the remaining time is frozen while paused and continues from the same point when resumed.
4. **Given** a focus interval reaches zero, **When** it ends, **Then** a completed session is recorded, the cycle count advances, and the next break starts automatically.
5. **Given** a break reaches zero, **When** it ends, **Then** the next focus interval waits paused at its full length until the user presses Start.
6. **Given** four completed focus intervals in a cycle, **When** the fourth ends, **Then** the next break is a long break, and after the long break the cycle count resets so the next focus reads "1 of 4".
7. **Given** a running interval, **When** the user presses "+5" (the value set in Settings), **Then** the remaining time and the total length each increase by exactly that many minutes and the progress indicator shrinks proportionally.
8. **Given** the Mac sleeps for 5 minutes during a running focus interval, **When** it wakes, **Then** the remaining time is 5 minutes lower than before sleep, not frozen.
9. **Given** the app is quit during a running interval, **When** it is relaunched before the interval ends, **Then** the timer resumes with the correct remaining time. If the interval ended while the app was closed, it is treated as finished on launch.

---

### User Story 3 - See and complete tasks from the popover (Priority: P2)

The user picks which task they are working on, and closes tasks in Todoist directly from the popover.

**Why this priority**: Task selection and completion turn the timer into a focus tool tied to real work. It is valuable but the timer works without it.

**Independent Test**: Select a task, run an interval, complete a different task by ticking its checkbox, and confirm the task is closed in Todoist.

**Acceptance Scenarios**:

1. **Given** the popover shows tasks, **When** the user clicks a task title, **Then** it becomes the current task and is highlighted as the task being timed.
2. **Given** a focus interval is running, **When** the user switches to another task, **Then** the remaining time continues and the completed session is attributed to the task that was current when the interval started.
3. **Given** an open task, **When** the user clicks its checkbox, **Then** the task is struck through immediately and closed in Todoist.
4. **Given** the closing request fails, **When** the failure is reported, **Then** the strike-through is rolled back and an inline error is shown.
5. **Given** the task being timed is closed, **When** the closure succeeds, **Then** the next open task becomes the current task.
6. **Given** a recurring task is closed, **When** the closure succeeds, **Then** the task reappears with its next due date instead of disappearing.

---

### User Story 4 - Keep the timer visible in a mini window (Priority: P2)

The user wants to see the countdown without opening the popover. A small floating window shows the timer and the main controls, above other apps including full-screen apps.

**Why this priority**: It makes the timer usable during full-screen work, which is a common reason for using a focus timer.

**Independent Test**: Open the mini window, switch to a full-screen app, and confirm the countdown stays visible and controllable.

**Acceptance Scenarios**:

1. **Given** the popover is open, **When** the user presses the mini window button, **Then** the mini window appears and the popover closes.
2. **Given** a timer starts and the "open mini window when a timer starts" behaviour is on, **When** the timer starts, **Then** the mini window opens.
3. **Given** the mini window is open over a full-screen app, **When** the user moves it, quits and relaunches the app, **Then** the window reappears at its last position.
4. **Given** a short break is running, **When** the mini window is shown, **Then** it uses the break colour set and displays "BREAK" with "· next: {task title}".
5. **Given** the mini window is open, **When** the user presses the expand button, **Then** the mini window closes and the popover opens.
6. **Given** the mini window is key, **When** the user presses Escape, **Then** the mini window closes.
7. **Given** an interval ends, **When** the mini window is open, **Then** it stays open and shows "00:00" with the primary button pulsing once.

---

### User Story 5 - Review focus time in a weekly Report (Priority: P3)

The user opens a Report to see how much focus time they logged in a week, which tasks took the time, and can export all session data as a CSV file.

**Why this priority**: Reporting shows progress and supports reflection, but the core timer and tasks do not depend on it.

**Independent Test**: Complete several sessions over two different days, open the Report, and check the totals, bars, and per-task times against the recorded sessions. Export the CSV and check the header row.

**Acceptance Scenarios**:

1. **Given** completed focus sessions this week, **When** the user opens the Report, **Then** the three stats show total focused time, completed focus sessions, and tasks completed this week.
2. **Given** the Report is open, **When** the user clicks a day's bar, **Then** the task list below shows that day's focus time per task, sorted by time, largest first.
3. **Given** the Report is open on the current week, **When** the user tries to move to a later week, **Then** the forward control is disabled.
4. **Given** a week with no sessions, **When** the Report opens, **Then** the stats read "0h 00m", "0", and "0", and the day section reads "No focus sessions on this day."
5. **Given** sessions exist in any week, **When** the user presses Export CSV and saves the file, **Then** the file contains all sessions (not only the visible week), UTF-8 encoded, with the exact header row defined in the Report requirements.

---

### User Story 6 - Adjust timer lengths and general behaviour in Settings (Priority: P3)

The user changes focus and break lengths, the cycle length, the "+" increment, and general options. Changes save as they are made.

**Why this priority**: Defaults work for most users. Settings serve users who want different timings.

**Independent Test**: Change each timer value and toggle, quit and relaunch, and confirm the values persist and apply to the next interval only.

**Acceptance Scenarios**:

1. **Given** a timer value is changed, **When** the user leaves the field or presses Return, **Then** the value is clamped to its allowed range and saved immediately.
2. **Given** a non-digit character is typed into a number field, **When** it is entered, **Then** it is rejected.
3. **Given** a running interval, **When** a timer length is changed, **Then** the running interval is unaffected and the new length applies to the next interval.
4. **Given** the "+" increment is changed, **When** the popover is viewed, **Then** the "+" button label updates immediately.
5. **Given** the "Launch at login" toggle, **When** Settings opens, **Then** the toggle reflects the real login-item status rather than a stored guess.
6. **Given** Settings is open, **When** the user changes the Todoist token via "Change token", **Then** the Connect window opens in replace mode and, on success, Settings shows the new account name.

---

### Edge Cases

- **Network failure while loading tasks**: The last cached task list is shown with a line above it: "Couldn't reach Todoist. Showing tasks from {time}." The app never shows a blank list because of a network failure.
- **No tasks due today**: The list area reads "Nothing due today". The timer works without a selected task and the current task line reads "No task selected".
- **Token revoked or rejected after connection**: The task list is replaced with "Todoist disconnected" and a "Reconnect" button that opens the Connect window.
- **Rate limiting**: Requests that are rate-limited are retried with increasing waits of 2, 4, and 8 seconds, then given up without an error dialog. Cached data stays visible.
- **Timeout**: A request that takes longer than 10 seconds is treated as a network failure.
- **Skip with a short interval**: If an interval is skipped before 60 seconds have run, no session is recorded and the cycle count does not change.
- **Interval ends while the Mac is asleep or the app is closed**: On wake or launch, the interval is finished, its session recorded, and the next interval set up, without a missed-notification gap.
- **Interval ends while notifications are denied**: The timer still advances and the mini window and menu bar still show the state. The user is not blocked.
- **Recurring task closed**: The task reappears with its next due date and the task list refreshes after closing.
- **Very long task title**: Titles are truncated with a tail ellipsis in every place they appear, including the mini window and Report.
- **Time-blocked tasks**: Tasks with a due time and a duration show their time range. Tasks with a time range sort first by start time, then the rest keep the provider's order.
- **Unknown project colour or name**: Projects not in the cached list are fetched again before the colour is shown; if still unknown, a neutral grey is used.
- **Token storage**: The token is never written to preferences, logs, crash reports, or exports.

## Requirements *(mandatory)*

### Functional Requirements

**App presence and platform**

- **FR-001**: The app MUST run only in the macOS menu bar, with no Dock icon and no main menu. The user MUST be able to quit the app from a Quit control in the popover footer and from a right-click menu on the menu bar item.
- **FR-002**: The app MUST target macOS 14.0 or later.
- **FR-003**: The app MUST use light appearance only in every window. Dark mode is out of scope for v1.
- **FR-004**: All user-facing text MUST be in English and kept in a single localisable resource set so that translations can be added later without code changes.
- **FR-005**: The app MUST NOT send analytics, telemetry, or any network request except to the connected provider's API host (for v1, the Todoist API host).

**Task provider**

- **FR-006**: All task data MUST be accessed through a provider-neutral task interface and a normalised task record. No screen, the timer, or the local store may reference a specific provider.
- **FR-007**: v1 MUST ship with exactly one provider, Todoist. Adding a provider later MUST NOT require changes to screens or storage.
- **FR-008**: The app MUST fetch tasks that are due today or overdue, in the provider's order, with time-blocked tasks sorted first by start time.
- **FR-009**: The app MUST show project name and project colour for each task. Project colours MUST come from the provider and map to the provider's published colour set.
- **FR-010**: The app MUST show a task's time range when it has a due time and a duration, formatted in the user's locale with the 24-hour setting respected.
- **FR-011**: Task priority MUST map from the provider's levels to the four priority ring colours shown to the user, with the highest priority shown in red and the lowest in grey.
- **FR-012**: The app MUST cache the last task list and the project list so the popover can show content offline.
- **FR-013**: Projects MUST be cached for 24 hours and refreshed when a task references an unknown project.

**Connection and credentials**

- **FR-014**: On first launch, or when no valid credential is stored, the app MUST show the Connect window and keep the rest of the app inactive until a connection succeeds. The menu bar item MUST remain visible, and its popover MUST offer a "Connect Todoist" button that reopens the Connect window.
- **FR-015**: The Connect window MUST validate the token format locally (40 hexadecimal characters after trimming whitespace) before any network request.
- **FR-016**: On a valid format, the app MUST verify the token with the provider. A success (200) connects; an authorisation failure (401 or 403) shows the rejected state; any other failure or a 10-second timeout shows the network error state "Couldn't reach Todoist. Check your connection."
- **FR-017**: On successful connection, the app MUST store the token only in the macOS Keychain, store the account display name, and request notification permission.
- **FR-018**: The provider's token MUST NOT appear in preferences, logs, crash reports, or exports. The app MUST keep a small local diagnostic log that never contains the token or task titles, and Settings MUST let the user export that log manually. Nothing in the log is sent automatically.
- **FR-019**: Settings MUST show the connected account name and the last four characters of the token, with a "Change token" action that reuses the Connect window in replace mode.
- **FR-020**: Authorisation failures during normal use MUST show the "Todoist disconnected" state with a Reconnect action.
- **FR-021**: Rate-limited requests MUST be retried after 2, 4, and 8 seconds, then abandoned silently.
- **FR-022**: Network failures during normal use MUST keep the cached list visible with the line "Couldn't reach Todoist. Showing tasks from {time}."
- **FR-023**: The app MUST fetch tasks when the popover opens if the last fetch is older than 60 seconds, after a task is closed, and on wake from sleep. There is no background polling and no manual refresh button.

**Timer behaviour**

- **FR-024**: The timer MUST have exactly three modes: focus, short break, and long break. Each mode has its own accent colour set.
- **FR-025**: Remaining time MUST be computed from the wall clock and an end time, never by counting ticks, so that sleep, background suspension, and a busy interface cannot make it drift.
- **FR-026**: Timer state (mode, end time or remaining time, running state, cycle count, and current task) MUST persist across app quit, relaunch, and system sleep.
- **FR-027**: Starting or resuming MUST set the end time to now plus the remaining time. Pausing MUST store the remaining time and clear the end time.
- **FR-028**: "+N" MUST add N minutes (from Settings, default 5) to the remaining time and to the interval's total length, in any mode, and MUST move the running end time by the same amount.
- **FR-029**: When an interval reaches zero, the app MUST record a completed session for focus or break intervals (see Key Entities), advance the cycle count after focus, and select the next mode: a long break after the configured number of focus intervals, otherwise a short break.
- **FR-030**: Breaks MUST start automatically when the preceding focus ends. Focus intervals MUST wait, paused at full length, until the user starts them.
- **FR-031**: Skip MUST end the current interval and move to the next mode. If the interval ran for 60 seconds or more, an interrupted session is recorded. Skip MUST NOT advance the cycle count.
- **FR-032**: The cycle length MUST default to 4 focus intervals. After a long break the cycle count MUST reset to zero.
- **FR-033**: Durations MUST be read from Settings when each interval is created. Changes MUST NOT alter a running interval.
- **FR-034**: The app MUST schedule a system notification for each interval's end time when a timer starts or changes, cancel it on pause, skip, or reset, and fire it even when the app is not frontmost.
- **FR-035**: Notification titles MUST be "Focus done — take 5" (short break), "Focus done — long break, 15 min" (long break, using the configured length), and "Break over — back to {task title}" (returning to focus). The body MUST be the task title.
- **FR-036**: A short sound MUST play with the notification when the "Sound when a timer ends" setting is on, and be silent otherwise.
- **FR-037**: On wake from sleep, the app MUST re-evaluate whether the current interval has ended.

**Popover (primary screen)**

- **FR-038**: The popover MUST be a 360 pt wide window that opens from the menu bar item, closes on a click outside it, and closes on Escape.
- **FR-039**: The menu bar item MUST show a colour dot for the current mode and the remaining time as MM:SS with tabular digits. Idle shows a grey dot and "25:00".
- **FR-040**: The popover MUST show, from top to bottom: header row (mode pill, cycle label, mini window button, settings button), timer, current task with project, progress bar, controls, divider, list header with open-task count, task list, footer with focused time today and a link to the Report. The footer also holds a Quit control (see FR-001).
- **FR-041**: The cycle label MUST read "{n} of {cycle}" during focus and "up next: focus" during breaks.
- **FR-042**: The controls MUST be a primary button ("Start focus", "Pause", or "Resume" depending on state), a "+N" button, and a skip button with an icon only.
- **FR-043**: Clicking a task title MUST select it as the current task at any time.
- **FR-044**: Clicking a task checkbox MUST close the task immediately in the provider, strike it through at once, and roll back with an inline error if the request fails.
- **FR-045**: Each task row MUST show a checkbox coloured by priority, the title, a meta line with time range (if any), and the project name with its colour marker.
- **FR-046**: The popover footer MUST show total focused time today and open the Report when its link is clicked.
- **FR-047**: The keyboard MUST support Space to start or pause, Return on a focused task row to select it, and Escape to close the popover.
- **FR-048**: The task list MUST show five placeholder rows while tasks load.
- **FR-049**: During break modes, the pill, progress bar, primary button, and menu bar dot MUST switch to the break colour set, and the current task line MUST show the task that will be timed next.

**Mini window**

- **FR-050**: The mini window MUST be a small floating card, 288 × 108 pt, that stays above all other windows, including full-screen apps, and is not resizable.
- **FR-051**: The mini window MUST remember its last position. On first appearance it MUST be placed at the top right of the main screen, 24 pt from the right edge and 12 pt below the menu bar.
- **FR-052**: The mini window MUST open from the popover's mini window button, or when a timer starts while "Open mini window when a timer starts" is on (default on, stored but with no visible control in v1).
- **FR-053**: Opening the mini window MUST close the popover.
- **FR-054**: The mini window MUST show the mode name, the current task title, the remaining time, a "+N" button, and a play/pause button, plus a progress bar along its bottom edge.
- **FR-055**: The mini window MUST NOT close when an interval ends. It MUST close when its expand button is pressed (opening the popover instead) or when Escape is pressed while it has focus.
- **FR-056**: The "Keep mini window on top" setting MUST control whether the mini window stays above other windows (default on).

**Report**

- **FR-057**: The Report MUST be a standard resizable window, 860 pt wide by default with a minimum width of 520 pt. Its content column MUST be centred with a maximum width of 660 pt.
- **FR-058**: The Report MUST show one week at a time, Monday to Sunday, following the system's first-day-of-week setting, with previous and next controls. The next control MUST be disabled on the current week.
- **FR-059**: The Report MUST show three stats: total focused time for the week (formatted as "Xh YYm"), count of completed focus sessions, and count of tasks completed from Free Focus that week.
- **FR-060**: The Report MUST show a bar for each day of the week with the day's focused time as a label above the bar, in H:MM format. Days without focus MUST show a dash and a short stub bar.
- **FR-061**: Clicking a day's bar MUST show that day's per-task focus time below the chart. The default selection is today.
- **FR-062**: The per-task list MUST show each task's title, total time, number of completed focus sessions, share of the top task's time as a bar, and project name.
- **FR-063**: The Report MUST show "No focus sessions on this day." when a day has no sessions.
- **FR-064**: Export CSV MUST open a save dialog with the default name "free-focus-sessions-{date}.csv", where {date} is the export date in YYYY-MM-DD format. The export MUST include all sessions across all weeks, UTF-8, comma-separated, with the header row exactly: `id,todoist_task_id,task_title,project_name,session_type,planned_duration_sec,actual_duration_sec,started_at,ended_at,status`.
- **FR-065**: CSV timestamps MUST be ISO 8601 in UTC. `session_type` MUST be `focus`, `short_break`, or `long_break`. `status` MUST be `completed` or `interrupted`.
- **FR-066**: Reopening the Report while it is already open MUST bring the existing window to the front rather than opening a second one.

**Settings**

- **FR-067**: Settings MUST be a fixed-width 580 pt window with three grouped panels in this order: Todoist, Timer, General. Every change MUST save immediately. There is no Save button.
- **FR-068**: The Timer panel MUST provide the following values, each with its default and allowed range:
  - Focus length: 25 minutes (5–120)
  - Short break: 5 minutes (1–30)
  - Long break: 15 minutes (5–60)
  - Long break after: 4 focus intervals (2–8)
  - "+" button adds: 5 minutes (1–30)
- **FR-069**: Number entries MUST be clamped to their range on leaving the field or pressing Return, and MUST reject non-digit characters.
- **FR-070**: The General panel MUST provide three toggles: "Sound when a timer ends" (default on), "Keep mini window on top" (default on), and "Launch at login" (default on, reflecting the real login-item status when Settings opens).
- **FR-071**: Settings MUST open from the popover's settings button and from the standard settings shortcut while the app is active.
- **FR-072**: Settings MUST show a footer with the app name and version, the open-source licence, "no telemetry", and the line "Not created by, affiliated with, or supported by Doist Ltd."

**First launch window**

- **FR-073**: The Connect window MUST be a 420 pt wide, non-resizable window, centred on the main screen, with minimise and zoom disabled.
- **FR-074**: The Connect window MUST show, from top to bottom: app icon, title "Connect Todoist", explanation line, "API token" label, a masked token field, error line when needed, a help line with an "Open" link to the provider's developer settings page, a Connect button, and a privacy line.
- **FR-075**: The token field MUST accept paste, MUST NOT autocorrect or spellcheck, and MUST submit on Return when valid.
- **FR-076**: The Connect window MUST show the privacy line "Stored in your Mac's Keychain · Not affiliated with Doist Ltd."

**Local data**

- **FR-077**: The app MUST record a session for each completed interval and each interrupted focus or break interval, with the fields listed under Key Entities.
- **FR-078**: The app MUST record a completed-task record whenever a task is closed from Free Focus.
- **FR-079**: Session and task data MUST be stored on the Mac only. Nothing MUST be sent to any server other than the connected provider. Settings MUST provide a "Delete all data" action that, after a confirmation step, deletes all focus sessions, completed-task records, the cached task and project lists, and the timer state; removes the provider token from the Keychain; and disconnects the provider, returning the app to the Connect window.

**Accessibility and brand**

- **FR-080**: Every icon-only control MUST have an accessible label that screen readers announce.
- **FR-081**: Text MUST meet a contrast ratio of at least 4.5:1 against its background. Captions and small labels MUST use the minimum-contrast colour defined in the design system.
- **FR-082**: The product name MUST NOT contain the word "Todoist" or any other third-party trademark.
- **FR-083**: The Doist non-affiliation line MUST be shown in Settings and on the Connect window.

**Design fidelity**

- **FR-084**: Every colour, font, size, radius, shadow, and spacing value MUST come from a single design-token source. No screen may hard-code a visual value.
- **FR-085**: Three typefaces MUST be bundled with the app and used as follows: a text face for all UI text, a serif heading face for window titles and the "Today" heading, and a monospace-style numeric face for the timer, stats, times, and number fields. The app MUST NOT download fonts at runtime.
- **FR-086**: Visual design MUST follow the design canvas "Todomato — macOS focus timer for Todoist" (working name). Where this specification and the canvas disagree, this specification governs behaviour and the canvas governs appearance.
- **FR-087**: Icons MUST use the system symbol set at medium weight.

### Design Tokens (summary)

These are the values the design canvas defines. They are listed here so that the design can be checked against them. The full token list is part of the design system.

**Colours**

| Role | Hex | Use |
| --- | --- | --- |
| Primary text | #1F1F1F | Body and headings |
| Timer digits | #1A1A1A | Large timer |
| Secondary label | #2B2B2B | Current task title, secondary button labels |
| Body muted | #6B6B6B | Body copy on light grounds, icon buttons |
| Caption muted | #737373 | Captions and meta lines (minimum for text under 14 pt) |
| Timer colon | #C9C3BB | The colon in the large timer only |
| Focus accent | #D1453B | Focus mode, toggles on, today markers |
| Focus accent, deep | #C93C31 | Primary button fill bottom, links |
| Focus accent, top | #DA4F43 | Primary button fill top |
| Focus text | #A8352A | Text on the focus tint, error text |
| Focus tint | #FDECE8 | Mode pill background, avatar, error glow |
| Selected row | #FDF0EC | Background of the task being timed |
| Short break | #2F7D78 | Short break accent (top #388A84, bottom #2A716C, tint #E3F1EF, text #235F5B) |
| Long break | #246FE0 | Long break accent (top #3079E6, bottom #1F63CC, tint #E6EFFC, text #1C56AD) |
| Priority 2 ring | #EB8909 | Checkbox ring, priority 2 |
| Neutral ring | #A3A3A3 | Checkbox ring, priority 4, completed fill |
| Connected | #2E9E46 | Connected status dot |
| Surface | #FFFFFF | Window and popover background |
| Sunken surface | #F7F6F4 | Settings background, token field fill |
| Number field fill | #F4F3F1 | Number entries in Settings |
| Toggle off | #E3E2E0 | Toggle track when off |
| Hairline | black at 8–10% | Row and section dividers, 0.5 pt |

Priority ring colours (provider priority to ring): highest → #D1453B, high → #EB8909, medium → #246FE0, normal → #A3A3A3.

**Type scale**

| Role | Face | Size / line | Weight |
| --- | --- | --- | --- |
| Large timer | Numeric | 62 / 70 | 500 |
| Mini timer | Numeric | 40 / 46 | 500 |
| Stat number | Numeric | 26 / 32 | 500 |
| Window title | Serif | 30 / 36 | 600 |
| Onboarding title | Serif | 24 / 30 | 600 |
| List title | Serif | 17 | 600 |
| Section title | Serif | 16 | 600 |
| Body | Text | 14 / 19–20 | 400 (500 current task, 600 selected row) |
| Button | Text | 14 | 600 |
| Meta | Text | 12 | 400 |
| Eyebrow | Text, uppercase | 11–12 | 600 |

**Shape and depth**

| Element | Value |
| --- | --- |
| Popover and mini window radius | 14 pt |
| Window radius | 12 pt |
| Grouped panel radius | 10 pt |
| Button and input radius | 9 pt (8 pt in the mini window and small buttons) |
| Hairline border | 0.5 pt, black 14–16% |
| Spacing scale | 4, 6, 8, 10, 12, 14, 16, 18, 20, 24, 28, 36, 40 pt |

### Key Entities

- **Task**: An open item from the provider. Attributes: provider task ID, title, project name, project colour, priority, due date and time, duration, whether it is recurring.
- **Project**: A grouping of tasks. Attributes: provider project ID, name, colour name, last fetched time.
- **Focus Session**: One interval record. Attributes: ID, provider task ID (may be empty), task title, project name, project colour, session type (focus, short break, long break), planned duration (planned plus extensions), actual duration, started time, ended time, status (completed or interrupted).
- **Completed Task Record**: A task closed from Free Focus. Attributes: provider task ID, title, completed time.
- **Timer State**: The current timer. Attributes: mode, running state, end time or remaining time, cycle count, current task, interval start time.
- **Settings**: Timer lengths, cycle length, "+" increment, sound, mini window on top, launch at login, mini window auto-open.
- **Provider Connection**: Provider name, account display name, last four characters of the token, connection state (connected, disconnected, unreachable). The token itself is stored only in the Keychain.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A new user connects a valid task account and sees today's tasks in under 1 minute from first launch.
- **SC-002**: A running focus interval remains accurate to within 1 second after the Mac sleeps for 5 minutes and wakes, in 100% of trials.
- **SC-003**: After quitting and relaunching mid-interval, the remaining time is accurate to within 1 second in 100% of trials.
- **SC-004**: Adding time with "+N" changes the remaining time by exactly N minutes, and the end-of-interval notification fires within 1 second of the new end time.
- **SC-005**: Four completed focus intervals in a row produce a long break every time, and the cycle count returns to 1 after it.
- **SC-006**: A skipped interval of 60 seconds or more is recorded as interrupted, and the cycle count does not change, in 100% of trials.
- **SC-007**: Closing a task from the popover shows the closed state at once, and the task is closed in the provider within 5 seconds in normal network conditions.
- **SC-008**: When the network is unavailable, the popover shows cached tasks and the offline line in 100% of cases, never a blank list.
- **SC-009**: The mini window remains visible above a full-screen app and returns to its last position after relaunch, in 100% of trials.
- **SC-010**: Report totals match the recorded sessions for every week tested, and the exported CSV header matches the specified header exactly.
- **SC-011**: Every icon-only control is announced with a meaningful label by a screen reader, and all text meets the contrast requirement.
- **SC-012**: No provider token appears in preferences, logs, crash reports, or exported files, verified by search after a full connect, use, and export cycle.
- **SC-013**: Users can start a focus interval from the menu bar in under 5 seconds from first opening the popover.

## Assumptions

- **Target users**: Individual Mac users who already keep their tasks in Todoist and want a focus timer tied to those tasks. Users are not required to create an account.
- **Platform**: macOS 14 or later. Support for other operating systems is not in v1 scope, although the product structure must not block a later port.
- **Task provider**: Todoist is the only provider in v1. The provider interface must allow others to be added later without screen or storage changes, but no other provider is built, tested, or shown in v1.
- **Authentication**: Users paste a personal API token from their Todoist developer settings. OAuth is not used in v1.
- **Naming**: The product name is "Free Focus". The design canvas uses the working name "Todomato", and the canvas's working name is not used in user-facing text. Internal storage identifiers may keep the working-name prefix so that existing data is not orphaned.
- **Fonts**: The three bundled typefaces (Funnel Sans, Literata, and Google Sans Code) are licensed under the SIL Open Font License, as stated in the design file. No separate licence clearance is required before bundling. If a licence changes or a face is removed, the system monospace face is the fallback for numbers.
- **Notifications**: The user grants notification permission during Connect. If denied, the timer still works; the user simply gets no notification.
- **Launch at login**: On by default, as specified. Users can turn it off in Settings.
- **Mini window auto-open**: The "open mini window when a timer starts" behaviour is on by default and has no visible control in v1, matching the design.
- **Recurring tasks**: Closing a recurring task advances it to its next occurrence rather than removing it. The provider decides the next date.
- **Task list scope**: Only tasks due today or overdue are shown. Other filters, webhooks, CSV import, global shortcuts, full-screen break overlays, iCloud sync, dark mode, and manual mode tabs are out of scope for v1.
- **Time zones and formatting**: Times are shown in the user's current time zone and locale. CSV timestamps are UTC.
- **Design source**: The design canvas is the visual source of truth. The values in the Design Tokens summary match it. Any difference found during implementation is resolved in the canvas's favour.
- **Report week**: The Report uses the system's first-day-of-week setting, which is Monday in most locales.
- **Sessions before v1 release**: No migration is needed because v1 is the first release.
- **Task provider requirements**: The provider's public API reference is the source for every endpoint and field. Endpoints are verified against the reference before implementation, as required by the project constitution.
