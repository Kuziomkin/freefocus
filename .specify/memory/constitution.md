# Free Focus Constitution

## Core Principles

### I. Scope Discipline

v1 is exactly five screens: popover, mini window, Report, Settings, and First launch. v1 has
exactly one task provider: Todoist. No feature, setting, screen, or provider MUST be added unless
it appears in an approved spec. Deleting MUST be preferred over adding. Apply YAGNI: build only
what the current approved spec requires.

Rationale: a small, fixed surface keeps a menu bar utility fast to ship, easy to test, and hard to
bloat.

### II. Provider Independence

All task data MUST flow through the `TaskProvider` interface and the normalised `FocusTask` model.
Screens, the timer, and storage MUST NOT reference a specific provider. Each provider MUST live in
its own folder under `lib/providers/`, and nothing outside that folder MAY import it. Adding a
provider MUST NOT require any UI or storage change. A `FakeProvider` MUST be able to drive every
screen in tests.

Rationale: the provider boundary is the only place third-party API shapes are allowed to leak.

### III. Local-First Privacy

The product MUST NOT include a backend, user accounts, analytics, or telemetry. The only network
hosts the app MAY contact are the APIs of connected providers. Provider credentials MUST be stored
only in the macOS Keychain, with one Keychain entry per provider. Credentials MUST NOT appear in
preferences, logs, crash reports, or exports.

Rationale: users' task data and credentials stay on their Mac, and no third party can observe
usage.

### IV. Timer Correctness

All timing MUST be computed from wall-clock end dates. Timers MUST NOT be advanced by counting
ticks. Timer state MUST survive system sleep, App Nap, and app relaunch. The `TimerEngine` MUST be
pure Dart with no UI or provider imports, and it MUST be built test-first: failing tests are
written before the implementation.

Rationale: tick counting drifts whenever the process is suspended, and a Pomodoro timer that drifts
is wrong.

### V. Platform Boundaries

The project MUST use Flutter stable with Dart 3, targeting macOS 14 or later. Everything outside
`macos/` MUST remain platform-neutral. macOS-only behaviour MUST go through small, documented Swift
platform channels. The popover and the mini window MUST share one Flutter window.

Rationale: keeping platform code in one folder keeps the Dart code portable and makes macOS
behaviour easy to audit.

### VI. Design Fidelity

Every colour, font, size, radius, and shadow MUST come from `lib/theme/tokens.dart`. Widgets MUST
NOT contain hard-coded visual values. The fonts Funnel Sans, Literata, and Google Sans Code MUST be
bundled with the app. v1 MUST support light mode only. The Claude Design canvas is the visual
source of truth; deviations MUST be resolved in favour of the canvas.

Rationale: a single token file makes design drift visible in review and keeps the look consistent.

### VII. Testing

Unit tests MUST exist for `TimerEngine`, for every provider adapter (with mocked HTTP), and for the
session store. Widget tests MUST exist for shared components. Pure layout code does not require
tests. CI MUST pass before any change merges.

Rationale: the timer, provider adapters, and session data are the parts whose failures users
notice most and that are most costly to debug.

### VIII. External API Discipline

Every provider endpoint MUST be verified against that provider's official API reference before it
is used. Auth failures MUST open the reconnect flow. Rate-limited requests MUST back off. Network
failures MUST show cached data; a blank screen is never acceptable.

Rationale: third-party APIs change and fail, and the app must degrade gracefully instead of
guessing.

### IX. Dependencies

Dependencies MUST be kept to the minimum needed. Each package MUST be justified in the plan. Every
package version MUST be pinned. Every package MUST be actively maintained, with a release in the
last 12 months. Writing about 50 lines of code MUST be preferred over adding a package.

Rationale: each dependency adds supply-chain, maintenance, and size cost to a small utility.

### X. Accessibility and Brand Compliance

Every icon-only control MUST have a semantic label. Text MUST meet a contrast ratio of at least
4.5:1 against its background. The product name MUST NOT contain a third-party trademark. Each
connected provider's required disclaimer MUST be shown in Settings and on that provider's connect
screen. For Todoist, this is the Doist non-affiliation line.

Rationale: accessibility is a requirement, not an option, and trademark and disclaimer rules are
legal obligations.

## Platform and Technology Constraints

- Flutter stable channel and Dart 3. Deployment target macOS 14 or later.
- Fonts (Funnel Sans, Literata, Google Sans Code) MUST be bundled. Runtime font downloads are
  prohibited.
- Network access is limited to connected provider API hosts (Principle III).
- Credentials are stored only in the macOS Keychain, one entry per provider.
- Light mode only in v1.

## Development Workflow and Quality Gates

- Work follows the Spec Kit sequence: an approved spec precedes the plan, the plan precedes tasks,
  and tasks precede code (Principle I).
- The plan MUST include a Constitution Check that names each principle the work touches. Any
  added dependency MUST be justified in the plan (Principle IX).
- Timer changes MUST start with failing tests (Principle IV).
- CI MUST run unit tests, provider adapter tests with mocked HTTP, and widget tests for shared
  components. All must pass before merge (Principle VII).
- Any new or changed provider endpoint MUST cite the official API reference in the change
  description (Principle VIII).
- Reviewers MUST reject visual values hard-coded outside `lib/theme/tokens.dart` (Principle VI).

## Governance

This constitution supersedes all other practices and guidance in the repository, including specs,
plans, tasks, and templates. Where another artifact conflicts with it, the artifact MUST be
corrected.

Amendments MUST be proposed through `/speckit.constitution`. Each amendment MUST include a
semantic version bump and a one-line rationale, recorded in the commit message and the Sync Impact
Report. Versioning policy:

- MAJOR: a principle is removed or redefined in a backward-incompatible way.
- MINOR: a principle or section is added, or existing guidance is materially expanded.
- PATCH: wording, typo, or non-semantic clarification.

Compliance review: every spec, plan, and pull request MUST state how it complies with the principles
it touches. A plan whose Constitution Check fails MUST NOT proceed to implementation. Reviewers
MUST block merges that violate a MUST principle. Any SHOULD principle that is not followed MUST be
justified in the plan or pull request.

**Version**: 1.0.0 | **Ratified**: 2026-11-20 | **Last Amended**: 2026-11-20
