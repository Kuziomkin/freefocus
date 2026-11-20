# Free Focus — UI & implementation spec

Nov 20, 2026 · @Dzianis

## Overview

Free Focus v1 is a macOS menu bar Pomodoro timer that shows today's tasks from the user's task app, lets the user time one task at a time and tick tasks off at the source. Build it in Flutter (stable channel, Dart 3) for macOS 14.0+, with no backend and no account system. Keep everything outside `macos/` platform-neutral so a Windows build can follow; macOS-only behaviour goes through small Swift platform channels in `macos/Runner`. The popover and mini window share one Flutter window; Report, Settings and First launch are secondary windows (`desktop_multi_window`) that read shared data from the local database and preferences.

The visual source of truth is the Claude Design canvas "Todomato — macOS focus timer for Todoist" (Todomato was the working name; the product ships as Free Focus". Where this spec and the canvas disagree, this spec wins on behaviour and the canvas wins on look.

**Task sources are pluggable.** v1 ships one provider, Todoist, but every screen, the timer and the local store work only with the `TaskProvider` interface (see Task providers). Todoist-specific code lives in `lib/providers/todoist/` and nothing outside that folder may import it. Adding a provider later (TickTick, Things, Asana, Linear, Notion) must mean adding one folder and one registry line, with no UI or storage changes.

**Screens in v1**

| # | Screen | Window type | Size |
| --- | --- | --- | --- |
| 1 | Menu bar item + popover | Tray icon + frameless window (`tray_manager`, `window_manager`) | 360 × \~640 pt |
| 2 | Mini window (focus / break) | Same window as the popover, resized and set always on top | 288 × 108 pt |
| 3 | Report | Standard window | 860 pt wide, content column 660 pt |
| 4 | Settings | Secondary window (`desktop_multi_window`) | 580 pt wide |
| 5 | First launch (connect a task source) | Standard window, not resizable | 420 pt wide |

**App rules**

- `LSUIElement = YES` in `macos/Runner/Info.plist`: no Dock icon, no main menu. The app lives only in the menu bar.
- Light appearance only in v1. Dark mode is out of scope; set `themeMode: ThemeMode.light` on every app root, in every window.
- English only in v1, but keep all user-facing strings in ARB files (`flutter_localizations` + `gen-l10n`).
- No analytics, no telemetry, no network calls except to the connected provider's API (v1: `api.todoist.com`).

**Out of scope for v1:** task providers other than Todoist (the architecture is ready, only Todoist ships), webhooks, CSV import, task filters other than Today & overdue, global shortcuts, full-screen break overlay, iCloud sync, dark mode, manual mode tabs (the timer advances on its own; Skip moves to the next interval).

## Design tokens

Put every token below in one `lib/theme/tokens.dart` file (colours as `static const Color`, text styles as `TextStyle` constants, radii and shadows as `BorderRadius` / `BoxShadow` constants) and never hard-code a value in a widget.

### Colours

| Token | Hex | Used for |
| --- | --- | --- |
| `ink` | #1F1F1F | Primary text |
| `inkStrong` | #1A1A1A | Timer digits |
| `ink2` | #2B2B2B | Secondary buttons' labels, current task title |
| `muted` | #6B6B6B | Body copy on light grounds, icon buttons |
| `muted2` | #737373 | Captions, meta lines, small labels (minimum allowed for text under 14 pt) |
| `colon` | #C9C3BB | The ":" in the big timer only |
| `focus` | #D1453B | Focus mode accent, toggles on, today markers |
| `focusDeep` | #C93C31 | Bottom of primary button fill, links |
| `focusTop` | #DA4F43 | Top of primary button fill |
| `focusText` | #A8352A | Text on `focusTint`, error text |
| `focusTint` | #FDECE8 | Mode pill background, avatar, error ring glow |
| `rowSelected` | #FDF0EC | Background of the task being timed |
| `shortBreak` | #2F7D78 | Short break accent (fill top #388A84, bottom #2A716C, tint #E3F1EF, text #235F5B) |
| `longBreak` | #246FE0 | Long break accent (fill top #3079E6, bottom #1F63CC, tint #E6EFFC, text #1C56AD) |
| `priority2` | #EB8909 | Checkbox ring for Todoist priority 2 |
| `checkboxRing` | #A3A3A3 | Checkbox ring for priority 4, completed fill |
| `projectGreen` | #299438 | Example only: project colours come from Todoist |
| `success` | #2E9E46 | Connected dot |
| `surface` | #FFFFFF | Window and popover background |
| `surfaceSunk` | #F7F6F4 | Settings window background, token field fill |
| `fieldFill` | #F4F3F1 | Number inputs in Settings |
| `toggleOff` | #E3E2E0 | Toggle track when off |
| `hairline` | black at 8–10% | Row and section dividers, drawn 0.5 pt |

Todoist project colours: map Todoist's colour names (`green`, `blue`, `charcoal`, …) to Todoist's published hex values in a `TodoistColor` enum. The `#` glyph after a project name uses that colour.

### Typography

Three bundled Google Fonts. Register them with `ATSApplicationFontsPath` in Info.plist.

| Role | Family | Weights | Where |
| --- | --- | --- | --- |
| Text | Funnel Sans | 400, 500, 600 | All UI text by default |
| Headings | Literata (variable, opsz axis) | 600 | Window titles, "Today", Report date heading |
| Numbers | Google Sans Code | 500, 600 | Timer, menu bar time, task times, stats, number fields, token field |

| Style | Font | Size / line | Weight | Tracking |
| --- | --- | --- | --- | --- |
| `timerLarge` | Google Sans Code | 62 / 70 | 500 | −0.05 em |
| `timerMini` | Google Sans Code | 40 / 46 | 500 | −0.05 em |
| `statNumber` | Google Sans Code | 26 / 32 | 500 | −0.04 em |
| `windowTitle` | Literata | 30 / 36 | 600 | −0.015 em |
| `onboardingTitle` | Literata | 24 / 30 | 600 | −0.012 em |
| `listTitle` | Literata | 17 | 600 | −0.01 em |
| `sectionTitle` | Literata | 16 | 600 | −0.005 em |
| `body` | Funnel Sans | 14 / 19–20 | 400 (500 for current task, 600 for selected row) | −0.005 em |
| `button` | Funnel Sans | 14 | 600 | −0.01 em |
| `meta` | Funnel Sans | 12 | 400 | 0 |
| `eyebrow` | Funnel Sans | 11–12, uppercase | 600 | +0.06 em |

Licences: Funnel Sans and Literata are under the SIL Open Font License. Confirm the Google Sans Code licence before shipping; if it can't be bundled, fall back to SF Mono for numbers.

### Shape and depth

| Token | Value |
| --- | --- |
| Radius: popover, mini window | 14 pt |
| Radius: windows | 12 pt |
| Radius: grouped panels | 10 pt |
| Radius: buttons, inputs | 9 pt (8 pt in mini window and small buttons) |
| Radius: selected task row | 9 pt |
| Hairline border | 0.5 pt, black 14–16% |
| Popover / window shadow | three layers: 0.5 pt ring black 14%; y 2 blur 6 black 5%; y 28 blur 56 spread −16 black 30% |
| Popover material | white at 94% over `.ultraThinMaterial`-style blur |
| Spacing scale | 4, 6, 8, 10, 12, 14, 16, 18, 20, 24, 28, 36, 40 pt |

## Shared components

Build these as reusable SwiftUI views first; every screen is composed from them.

| Component | Spec | States |
| --- | --- | --- |
| `PrimaryButton` | Height 38 pt (32 in mini), radius 9, vertical gradient `modeTop` → `modeBottom`, white `button` text, optional 12 pt SF Symbol to the left (`play.fill` / `pause.fill`). Shadow: inner top highlight white 22% 1 pt; 0.5 pt ring black 12%; y 1 blur 2 black 12%; y 4 blur 10 spread −4 in the mode colour at 55% | Normal; pressed = brightness −6%; disabled = 45% opacity, no shadow. Colour follows the timer mode (focus, short, long). |
| `SecondaryButton` | Height 38 pt (32 in mini, 28–30 small), radius 9, white fill, 0.5 pt ring black 16%, y 1 blur 2 black 6%, label `ink2` 13 pt 600 | Normal; pressed = fill #F4F3F1 |
| `IconButton` | 28 × 28 pt hit area, radius 7, 16 pt glyph in `muted`, no fill | Hover = fill black 5% |
| `ModePill` | Height 22, horizontal padding 8/9, radius 11, fill `modeTint`, 6 pt dot in mode colour, `eyebrow` text in `modeText` | Focus / Short break / Long break |
| `TaskCheckbox` | 18 pt circle, 1.5 pt ring in the Todoist priority colour (p1 #D1453B, p2 #EB8909, p3 #246FE0, p4 #A3A3A3) | Done = filled #A3A3A3 with white 10 pt check; title struck through in #8C8C8C |
| `TaskRow` | Checkbox + title (`body`) + meta line 4 pt below: red calendar glyph + time range in Google Sans Code if the task has a time, project name + coloured `#` right-aligned. Padding 9/8/10/8, 0.5 pt bottom hairline (none on last row) | Selected (being timed) = fill `rowSelected`, radius 9, title weight 600, no hairline. Clicking the title selects; clicking the checkbox completes. |
| `Toggle` | 38 × 22 pt track, radius 11; 18 pt white knob with 0.5 pt ring + y 1 blur 3 black 22% | On = `focus` track, knob right; off = `toggleOff`, knob left. Animate 150 ms ease-out. |
| `GroupedPanel` | White fill, radius 10, 0.5 pt ring black 10%, y 1 blur 2 black 4%, rows 46 pt tall, 14 pt side padding, 0.5 pt hairline between rows | — |
| `Eyebrow` | `eyebrow` style in `muted2`, 8 pt above its panel, inset 12 pt | — |
| `ProgressLine` | 4 pt (popover) or 3 pt (mini) track black 6%, fill in mode colour, radius 2 | Width = elapsed ÷ total, animated linearly each second |

**Window chrome:** use standard macOS title bars with the traffic lights; hide the title text (`.windowStyle(.hiddenTitleBar)`) so the Literata heading inside the content is the only title. Content starts 10–18 pt below the title bar.

**Icons:** SF Symbols only, weight `.medium`. Mapping: mini window `pip`, settings `slider.horizontal.3`, skip `forward.end.fill`, expand `arrow.up.left.and.arrow.down.right`, calendar `calendar`, clock `clock`, lock `lock`, export `square.and.arrow.down`, chevrons `chevron.left` / `chevron.right`.

## Screen 1 — Menu bar item and popover

The popover is the app: timer on top, today's Todoist tasks below, one line of totals at the bottom. It opens when the user clicks the menu bar item and closes on click outside.

### Menu bar item

- Content: a 7 pt dot in the current mode colour + remaining time `MM:SS` in Google Sans Code 13 pt 600, tabular digits.
- Idle with no timer started: show the dot in `checkboxRing` grey and `25:00`.
- Setting "Show time in menu bar" is not in v1; always show the time.
- Implementation: `MenuBarExtra { PopoverView() } label: { MenuBarLabel() }` with `.menuBarExtraStyle(.window)`. Update the label once per second from the timer engine.

### Popover layout (top to bottom)

Width 360 pt. Background white 94% over blur, radius 14, shadow per tokens.

1. **Header row** (height 28, padding 16 top / 18 right / 20 left)
   - `ModePill` ("FOCUS", "SHORT BREAK", "LONG BREAK").
   - 8 pt gap, then cycle label in `meta` `muted2`: `3 of 4` during focus (number of the current pomodoro in the cycle), `up next: focus` during breaks.
   - Right side: `IconButton` mini window (opens Screen 2), `IconButton` settings (opens Screen 4).
2. **Timer** — `timerLarge`, 8 pt below header. Render minutes, colon, seconds as three `Text`s; the colon uses `colon` colour.
3. **Current task** — 2 pt below: task title (`body`, weight 500, `ink2`, single line, tail truncation) + project name and `#` in `meta` style.
4. **ProgressLine** — 16 pt below, 4 pt tall.
5. **Controls** — 16 pt below, 8 pt gaps:
   - `PrimaryButton` (fills remaining width): label "Start focus" (idle at full duration), "Pause" (running), "Resume" (paused mid-interval). Icon `play.fill` / `pause.fill`.
   - `SecondaryButton` 54 pt wide: `+5` (number = Settings › "+ button adds"). Adds that many minutes to the current interval.
   - `SecondaryButton` 42 pt wide, icon-only skip. Ends the current interval without counting it and moves to the next.
6. **Divider** — 0.5 pt hairline, full width.
7. **List header** — padding 14/20/6: "Today" in `listTitle` + `N tasks` in `meta` `muted2` (N = open tasks).
8. **Task list** — fixed height 318 pt, scrolls; horizontal padding 12. One `TaskRow` per task, in Todoist order: time-blocked tasks sorted by start time first, then the rest in Todoist's order.
9. **Footer** — 0.5 pt top hairline, fill black 1.8%, padding 11/20. Clock glyph + "**2h 30m** focused today" (number in Google Sans Code 600) + right-aligned link "Report ›" in `focusDeep` 600 that opens Screen 3.

### States

| State | What changes |
| --- | --- |
| Loading tasks | Task list shows 5 skeleton rows (grey bars, no shimmer) |
| No tasks today | List area shows "Nothing due today" in `muted` centred, and the timer still works without a task (current task line reads "No task selected") |
| Todoist error (network) | One line above the list in `focusText`: "Couldn't reach Todoist. Showing tasks from 18:05." Keep the cached list |
| Token rejected (401) | Replace the list with "Todoist disconnected" + `SecondaryButton` "Reconnect" that opens Screen 5 |
| Break modes | Pill, progress, primary button and menu bar dot switch to the break colour set. Current task line shows the task that will be timed next |

### Interactions

- Click a task title: it becomes the current task. Allowed at any time; if a focus interval is running, the remaining time continues and the session is attributed to the task that was current when the interval started (switching mid-interval splits nothing in v1).
- Click a checkbox: close the task in Todoist immediately (optimistic: strike through at once, roll back with an inline error if the API call fails). If the closed task was current, the next open task becomes current.
- Refresh tasks when the popover opens if the last fetch is older than 60 s. No manual refresh button.
- Keyboard: Space = start/pause, Return on a focused row = select, Esc closes the popover.

## Screen 2 — Mini window

The mini window is a small floating timer that stays above all apps, including full-screen ones, so the user sees the countdown without opening the popover.

### Window behaviour

- `NSPanel` subclass, style `[.nonactivatingPanel, .borderless]`, `level = .floating`, `collectionBehavior = [.canJoinAllSpaces, .fullScreenAuxiliary]`, `isMovableByWindowBackground = true`, `hidesOnDeactivate = false`, transparent background with the SwiftUI view drawing its own rounded card and shadow.
- Size 288 × 108 pt (the canvas artboard adds 6 pt of shadow margin around it). Not resizable.
- Position: remember the last frame in `UserDefaults`; first appearance = top-right of the main screen, 24 pt from the right and 12 pt below the menu bar.
- Opens when: the user clicks the mini window button in the popover, or a timer starts while "Open mini window when a timer starts" is on (default on). Opening it closes the popover.
- "Keep mini window on top" off (Settings) = `level = .normal`.
- Closes when: the expand button is clicked (opens the popover instead), or the user presses Esc while it is key. It does not close when an interval ends.

### Layout

Card: white 95% over blur, radius 14, 0.5 pt ring black 16%, shadows y 2 blur 4 black 5% and y 10 blur 24 spread −8 black 28%. Padding 11 top, 12 right, 14 left.

1. **Top row** (20 pt): 6 pt dot + `eyebrow` mode name in `modeText` ("FOCUS", "BREAK", "LONG BREAK") + "· {task title}" in 12 pt `muted`, truncating; right: 22 pt `IconButton` expand.
2. **Main row**: timer in `timerMini` with muted colon, filling the left; right: `SecondaryButton` 40 × 32 "+5" and `PrimaryButton` 40 × 32 icon-only play/pause, 6 pt gap.
3. **ProgressLine** 3 pt at the very bottom edge, full width, no side padding, right end rounded 2 pt.

### Break variant

Same layout with the short-break (teal) or long-break (blue) colour set. The task line reads "· next: {task title}". The +5 button extends the break.

### States

| State | Display |
| --- | --- |
| Running | Pause icon; time counts down |
| Paused | Play icon; time frozen |
| Interval finished, waiting for the user (auto-start off) | Time shows `00:00`, the primary button pulses once (scale 1 → 1.06 → 1, 300 ms), play icon starts the next interval |

## Screen 3 — Report

The Report window shows one week of focus time and, below it, today's time per task. All data comes from the local session store; nothing is fetched from Todoist here except task names already cached.

### Window

Standard window, default 860 × 760 pt, min width 520, resizable. Content column max 660 pt, centred, padding 18 top / 24 sides / 44 bottom. Opened from the popover footer link; reuse the window if already open.

### Layout (top to bottom)

1. **Header row**
   - Left: eyebrow "THIS WEEK" (changes to "LAST WEEK" or the week's dates when navigating back), then "Report" in `windowTitle`.
   - Right: week switcher — a 30 pt tall segmented control with white fill and hairline ring: `chevron.left` button, the range label `16 – 22 Nov` (Google Sans Code 13 pt 600), `chevron.right` button. Next is disabled on the current week. Weeks start on Monday (follow `Calendar.current.firstWeekday`).
2. **Stats strip** — 28 pt below, hairlines above and below, three equal cells separated by vertical hairlines. Each cell: number in `statNumber`, label in `meta` `muted2` 3 pt below.
   - Focused: sum of completed focus session durations, `10h 25m`.
   - Pomodoros: count of completed focus sessions.
   - Tasks completed: count of tasks closed from Todomato that week.
3. **Bar chart** — 32 pt below. Swift Charts `BarMark`, 7 bars (Mon–Sun), plot height 110 pt, bar max width 34 pt, corner radius 6 top / 2 bottom, 14 pt gaps, one baseline hairline, no gridlines, no y-axis.
   - Value label above each bar in Google Sans Code 11 pt 600: `2:05` (H:MM); days with no focus show `–` and a 4 pt stub bar.
   - Colours: today `focus`; other days with data #F4CFC9; empty days #F0EFED.
   - X labels below: weekday 11 pt `muted2` + date number 12 pt 600 in a 22 × 20 pt badge; today's badge is filled `focus` with white text.
4. **Day section** — 36 pt below: heading `20 Nov · Today · Friday` in `sectionTitle`, day total right-aligned in `meta`; 0.5 pt hairline under it. Shows today by default; clicking a bar shows that day instead.
5. **Task rows** — one per task with focus time that day, sorted by time descending. Row: title (`body`) + time right-aligned (Google Sans Code 14 pt 600, `1h 15m` / `50m`); 6 pt below, a 4 pt share bar (width = task time ÷ the top task's time, in the project colour) + `3 pomodoros` + project name and `#`. 12 pt vertical padding, hairline between rows.
6. **Export** — right-aligned `SecondaryButton` "Export CSV" with the download glyph, 20 pt below the list.

### Empty state

A week with no sessions: stats show `0h 00m`, `0`, `0`; bars all stubs; the day section reads "No focus sessions on this day." in `muted`.

### CSV export

`NSSavePanel`, default name `todomato-sessions-2026-11-20.csv`. Exports all sessions (not just the visible week), UTF-8, comma-separated, header row exactly:

```csv
id,todoist_task_id,task_title,project_name,session_type,planned_duration_sec,actual_duration_sec,started_at,ended_at,status
```

Timestamps in ISO 8601 UTC; `session_type` is `focus`, `short_break` or `long_break`; `status` is `completed` or `interrupted`.

## Screen 4 — Settings

Settings is one scrolling column with three grouped panels: Todoist, Timer, General. Every change saves immediately; there is no Save button.

### Window

SwiftUI `Settings` scene, fixed width 580 pt, height fits content. Background `surfaceSunk`. Padding 10 top / 36 sides / 32 bottom. "Settings" in `windowTitle`. Opened from the popover's settings button or ⌘, while the popover is open.

### Todoist panel

Eyebrow "TODOIST", then one `GroupedPanel` row (height fits, padding 12/14):

- 32 pt avatar circle, `focusTint` fill, initial of the Todoist name in `focusText` 13 pt 700.
- Name (14 pt 500) and below it a 6 pt green dot + "Connected · token ••••3f9a in Keychain" in `meta` `muted2` (last four characters of the token).
- Right: small `SecondaryButton` "Change token" (28 pt tall) → opens Screen 5 in replace mode. On success the old token is overwritten in Keychain.

### Timer panel

Eyebrow "TIMER", then a `GroupedPanel` with five 46 pt rows. Each row: label (14 pt) left, a 52 × 28 number field (fill `fieldFill`, 0.5 pt inner ring, radius 7, Google Sans Code 14 pt 500, right-aligned), unit label 13 pt `muted2` in a 70 pt column.

| Label | Key | Default | Range | Unit |
| --- | --- | --- | --- | --- |
| Focus | `focusMinutes` | 25 | 5–120 | min |
| Short break | `shortBreakMinutes` | 5 | 1–30 | min |
| Long break | `longBreakMinutes` | 15 | 5–60 | min |
| Long break after | `pomodorosPerCycle` | 4 | 2–8 | pomodoros |
| "+" button adds | `extendMinutes` | 5 | 1–30 | min |

Validation: clamp to the range on blur or Return; reject non-digits. Changes apply to the next interval, never to the running one. The `+` button labels in the popover update at once.

### General panel

Eyebrow "GENERAL", a `GroupedPanel` with three `Toggle` rows:

| Label | Key | Default | Effect |
| --- | --- | --- | --- |
| Sound when a timer ends | `playSound` | on | Play a short bundled chime with the notification |
| Keep mini window on top | `miniOnTop` | on | Mini window level `.floating` vs `.normal` |
| Launch at login | `launchAtLogin` | on | `SMAppService.mainApp.register()` / `unregister()`; read the real status on open |

"Open mini window when a timer starts" (`autoOpenMini`, default on) is stored but has no row in v1. Open the Settings scene from the popover with `SettingsLink` (macOS 14) and call `NSApp.activate(ignoringOtherApps: true)` first, because an agent app is never frontmost.

### Footer

28 pt below the last panel, inset 12: the 28 pt app icon + two lines in `meta` `muted2`: "**Todomato 1.0** · open source (MIT) · no telemetry" and "Not created by, affiliated with, or supported by Doist Ltd." (required by Todoist's brand rules).

### Persistence

All settings in `UserDefaults` via `@AppStorage`, keys as in the tables. The token is never stored there (see Todoist integration).

## Screen 5 — First launch (connect Todoist)

On first launch, or whenever no valid token is in Keychain, Todomato opens a single window asking for the Todoist API token. Nothing else in the app works until it succeeds; the menu bar item is visible but its popover shows a "Connect Todoist" button that reopens this window.

### Window

420 pt wide, height fits content (\~470 pt), not resizable, centred on the main screen, traffic lights with minimise and zoom disabled. Activate the app when showing it. Padding 8 top / 40 sides / 30 bottom.

### Layout (top to bottom)

1. App icon 64 × 64, centred, soft shadow y 6 blur 12 in `focusDeep` at 28%.
2. "Connect Todoist" in `onboardingTitle`, centred, 18 pt below.
3. "Paste your Todoist API token to see today's tasks next to the timer." 14/20 `muted`, centred, 6 pt below.
4. Label "API token" 13 pt 600, 26 pt below.
5. Token field 7 pt below: `SecureField`, height 40, radius 9, fill `surfaceSunk`, 0.5 pt inner ring black 14%, Google Sans Code 14 pt, placeholder "40 characters". Autocorrect and spellcheck off. Paste from clipboard must work.
6. Error line (only in error state), 8 pt below: 13 pt exclamation glyph + "Todoist didn't accept this token. Copy it again and paste the whole thing." 12 pt `focusText`.
7. Help line 9 pt below: "Find it in Todoist › Settings › Integrations › Developer." 12/18 `muted2` + link "Open ↗" in `focusDeep` 600 → opens `https://app.todoist.com/app/settings/integrations/developer` in the default browser.
8. `PrimaryButton` full width, 26 pt below, focus colours: "Connect".
9. Privacy line 16 pt below, centred: lock glyph + "Stored in your Mac's Keychain · Not affiliated with Doist Ltd." 11 pt `muted2`.

### States

| State | Field | Button | Other |
| --- | --- | --- | --- |
| Empty | Placeholder | "Connect", disabled (45% opacity) | — |
| Typed | Value masked | "Connect", enabled | Return submits |
| Checking | Read-only | "Checking…" with a small `ProgressView`, disabled | — |
| Rejected | Ring 1.5 pt `focus` + 3 pt outer glow `focus` at 14% | "Connect", enabled | Error line shown; editing the field clears the error |
| Network error | Normal ring | "Try again" | Error line text: "Couldn't reach Todoist. Check your connection." |
| Success | — | Turns into "✓ Connected as {name} · Start" for 1.2 s | Then the window closes and the popover opens |

### Validation logic

1. Trim whitespace. If the trimmed value is not 40 hex characters, show the Rejected state without a network call.
2. Otherwise call the Todoist API with the token (see Todoist integration: fetch user / projects). 200 = success; 401 or 403 = Rejected; anything else or a timeout after 10 s = Network error.
3. Only on success write the token to Keychain, store the user's display name and request notification permission (`UNUserNotificationCenter.requestAuthorization(options: [.alert, .sound])`).
4. In "Change token" mode (from Settings) the window is identical; on success it closes and Settings shows the new name.

## Timer engine

One `@MainActor @Observable final class TimerEngine` owns all timer state; the popover, mini window and menu bar label only read it. Time is computed from the wall clock, never by counting ticks, so sleep, App Nap and a busy main thread cannot make it drift.

&#91;embedded content: timer state machine · 4 states\]

The cycle length (4) comes from Settings › "Long break after". Idle exists only before the first Start after launch.

### State

```swift
enum Mode { case focus, shortBreak, longBreak }

@Observable final class TimerEngine {
  var mode: Mode = .focus
  var isRunning = false
  var endDate: Date?            // set while running
  var remaining: TimeInterval   // authoritative while paused
  var plannedDuration: TimeInterval
  var extraDuration: TimeInterval = 0   // sum of +N presses this interval
  var completedInCycle = 0      // 0...pomodorosPerCycle
  var currentTaskID: String?
  var intervalStartedAt: Date?
}
```

### Rules

1. **Remaining time** while running = `max(0, endDate.timeIntervalSinceNow)`. Drive the UI with a `Timer.publish(every: 0.25)` or `TimelineView(.periodic)`; round up to whole seconds for display.
2. **Start / Resume**: `endDate = now + remaining`, `isRunning = true`; on the first start of an interval also set `intervalStartedAt = now`.
3. **Pause**: `remaining = endDate − now`, `endDate = nil`, `isRunning = false`.
4. **+N**: `remaining += N×60`, `extraDuration += N×60`, and if running `endDate += N×60`. Progress total = `plannedDuration + extraDuration`. Allowed in every mode.
5. **Interval ends** (remaining hits 0): save a `FocusSession` with status `completed`; for focus increment `completedInCycle`; choose next mode (long break when `completedInCycle == pomodorosPerCycle`, then reset to 0 after the long break). Next interval is paused at full duration unless auto-start applies (v1: breaks auto-start, focus waits for the user).
6. **Skip**: save the session with status `interrupted` (only if it ran ≥ 60 s), do not increment the cycle, move to the next mode as in rule 5.
7. **Durations** are read from Settings when an interval is created, so changes affect the next interval only.
8. **Persistence across relaunch**: store `mode`, `endDate`/`remaining`, `isRunning`, `completedInCycle`, `currentTaskID` in `UserDefaults` on every change; on launch, restore and, if `endDate` is already past, finish that interval (rule 5) immediately.
9. **Sleep / wake**: no special handling needed because time is wall-clock based; on `NSWorkspace.didWakeNotification` re-evaluate rule 5.

### Notifications

- Schedule a `UNNotificationRequest` with a `UNTimeIntervalNotificationTrigger` for `endDate` whenever the timer starts or +N changes it; cancel it on pause, skip and reset. This fires even if the app is suspended.
- Titles: "Focus done — take 5" (short break), "Focus done — long break, 15 min", "Break over — back to {task title}". Body: the task title.
- Sound: bundled `chime.caf` when `playSound` is on, otherwise silent.
- Request permission after a successful onboarding connect, never on launch.

## Todoist integration and local data

Todomato talks only to Todoist's unified API v1 at `https://api.todoist.com/api/v1`, authenticated with the user's personal token as `Authorization: Bearer <token>`. Before coding, the agent must check every endpoint below against the official reference at [developer.todoist.com/api/v1](https://developer.todoist.com/api/v1/); third-party write-ups disagree on the filter endpoint.

### Endpoints

| Purpose | Method + path | Notes |
| --- | --- | --- |
| Validate token, get name | `GET /user` | Use `full_name`; fall back to `GET /projects` returning 200 if `/user` is not available |
| Today & overdue tasks | `GET /tasks/filter?query=today%20%7C%20overdue` | If the reference shows filtering on `GET /tasks?filter=…` instead, use that |
| Project names and colours | `GET /projects` | Cache for 24 h; refresh when a task references an unknown project |
| Complete a task | `POST /tasks/{id}/close` | Recurring tasks advance to their next date instead of disappearing; refetch after closing |

All list endpoints are cursor-paginated: loop while `next_cursor` is non-null, passing it as `cursor`. Decode with `Codable`, `snake_case` key strategy.

**Priority mapping:** the API uses 1 = normal … 4 = urgent, the reverse of the p1–p4 the user sees. API 4 → p1 ring (#D1453B), 3 → p2 (#EB8909), 2 → p3 (#246FE0), 1 → p4 (#A3A3A3).

**Time-blocked tasks:** a task shows a time range when its `due` has a time component and a `duration` (minutes). Display `HH:mm–HH:mm` in the user's locale, 24 h if the system uses it.

### Client rules

- One `TodoistClient` actor with `async throws` methods: `fetchUser()`, `fetchTodayTasks()`, `fetchProjects()`, `closeTask(id:)`.
- Errors: 401/403 → `.unauthorized` (app shows the reconnect state); 429 → retry with exponential backoff 2, 4, 8 s, then give up silently; other failures → `.network`.
- Timeout 10 s per request. No background polling in v1: fetch on popover open if older than 60 s, after closing a task, and on wake from sleep.

### Token storage

Keychain only: `kSecClassGenericPassword`, service `app.todomato.todoist`, account `personal-token`, accessibility `kSecAttrAccessibleAfterFirstUnlock`. Never write the token to `UserDefaults`, logs, crash reports or the CSV.

### Local data

SwiftData model, stored in the app's Application Support container:

```swift
@Model final class FocusSession {
  @Attribute(.unique) var id: UUID
  var todoistTaskID: String?
  var taskTitle: String
  var projectName: String?
  var projectColor: String?
  var sessionType: String      // focus | short_break | long_break
  var plannedDurationSec: Int  // planned + extensions
  var actualDurationSec: Int
  var startedAt: Date
  var endedAt: Date
  var status: String           // completed | interrupted
}
```

Also keep a small `CompletedTask` record (Todoist id, title, completedAt) whenever a task is closed from Todomato; the Report's "Tasks completed" counts these. Cache the last task list as JSON in Application Support so the popover can show it offline.

## Acceptance checklist

The build is done when every item below passes on a clean macOS 14 user account. Build in this order: tokens and components, timer engine, popover, onboarding, Todoist client, mini window, Settings, Report.

- [ ] Fresh install opens the Connect Todoist window; a bad token shows the error state without a network call; a valid token closes it and opens the popover with today's tasks.
- [ ] The token is in Keychain under `app.todomato.todoist` and nowhere else (search `UserDefaults` and the container).
- [ ] Start focus, put the Mac to sleep for 5 minutes, wake it: remaining time is 5 minutes lower, not frozen.
- [ ] Quit and relaunch mid-interval: the timer resumes with the correct remaining time.
- [ ] +5 adds exactly the Settings value; the progress bar shrinks proportionally; the notification fires at the new end time.
- [ ] Four completed focus intervals produce a long break; the cycle label reads 1 of 4 → 4 of 4, then resets.
- [ ] Skip records an interrupted session (if ≥ 60 s) and does not advance the cycle count.
- [ ] Ticking a task closes it in Todoist (visible in the Todoist app within seconds); a recurring task reappears with its next date.
- [ ] Pulling the network shows the cached list and the "Couldn't reach Todoist" line; revoking the token in Todoist shows "Todoist disconnected" + Reconnect.
- [ ] The mini window floats over a full-screen app, remembers its position, and switches to teal on a short break.
- [ ] Report numbers match the session store; Export CSV produces the exact header row in the spec.
- [ ] Settings changes persist after relaunch; Launch at login reflects `SMAppService` status.
- [ ] Every screen uses only `Theme.swift` tokens; Funnel Sans, Literata and Google Sans Code render (check with Font Book disabled system fonts).
- [ ] The Doist disclaimer appears in Settings and onboarding; the app name never includes "Todoist".
- [ ] VoiceOver reads every icon-only button (mini window, settings, skip, expand, play/pause, checkboxes) with a label.
