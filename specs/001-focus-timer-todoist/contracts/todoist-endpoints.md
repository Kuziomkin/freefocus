# Contract: Todoist API v1 (consumed endpoints)

**Source of truth**: `https://developer.todoist.com/openapi.json` and `https://developer.todoist.com/api/v1/`. Verified 2026-11-20 (Principle VIII). Any change to these endpoints MUST cite the reference in its change description.

**Base URL**: `https://api.todoist.com/api/v1`
**Auth header**: `Authorization: Bearer <personal token>`
**Timeout**: 10 s per request (FR-016, FR-022 context).

## Endpoints used

| Purpose | Method and path | Query | Response | Notes |
| --- | --- | --- | --- | --- |
| Validate token and read the account | `GET /user` | none | `UserJSON`; use `full_name` | 401 means rejected. |
| Tasks due today or overdue | `GET /tasks/filter` | `query=today%20%7C%20overdue`, `limit`, `cursor` | Paginated: `results`, `next_cursor` | Loop while `next_cursor` is non-null. |
| Project names and colours | `GET /projects` | `limit`, `cursor` | Paginated: `results`, `next_cursor` | Colour is a name string, e.g. `charcoal`. |
| Close (complete) a task | `POST /tasks/{task_id}/close` | none | `null` | Recurring tasks advance to the next occurrence. Refetch after closing. |

**Note on the spec**: the spec's endpoint table said the filter list is under `items`. The reference shows the key is `results`. The spec's paths otherwise match.

## Response fields used

### Task (`ItemSyncView`)

| Field | Type | Used for |
| --- | --- | --- |
| `id` | string | `FocusTask.id` |
| `content` | string | `FocusTask.title` |
| `project_id` | string | Project lookup |
| `section_id` | string or null | Not used in v1 |
| `parent_id` | string or null | Subtasks are listed as tasks in v1 |
| `priority` | integer 1 to 4 | Priority ring (1 normal, 4 urgent) |
| `due` | object or null | `date` (date or date-time), `timezone`, `is_recurring` |
| `duration` | object or null | `amount` (positive int) and `unit` (`minute` or `day`) |
| `checked` | boolean | Completion state |
| `order_key` | string | Provider order |
| `is_deleted` | boolean | Ignore deleted tasks |

### Project (`PersonalProjectSyncView`, `WorkspaceProjectSyncView`)

| Field | Type | Used for |
| --- | --- | --- |
| `id` | string | Project ID |
| `name` | string | Project name |
| `color` | string | Colour name (e.g. `charcoal`) |
| `is_archived` | boolean | Excluded from the cache |
| `is_deleted` | boolean | Excluded from the cache |

### User (`UserJSON`)

| Field | Type | Used for |
| --- | --- | --- |
| `full_name` | string | Account name in Settings and the Connect success line |
| `id` | string | Not stored |

## Error mapping (client rules)

| HTTP status | Client result | App behaviour |
| --- | --- | --- |
| 200 | Success | Decode `results`, follow `next_cursor`. |
| 401 | `unauthorized` (error code 477 in body) | Show "Todoist disconnected" with Reconnect (FR-020). Connect window rejects the token (FR-016). |
| 403 | `unauthorized` | Same as 401 during connect. |
| 429 | `rateLimited` | Retry after 2, 4, and 8 s, then give up silently (FR-021). |
| 5xx, network, timeout (10 s) | `network` | Keep the cached list and show the offline line (FR-022). |
| Other 4xx | `network`-class error, logged | Show the cached list; log category only (FR-018). |

**Open check (T001)**: the 429 backoff values and any `Retry-After` header are taken from the spec. Confirm against the "Request limits" page before the client is merged. A 401 response carries `Retry-After` and `retry_after` metadata; these are informational because a rejected token does not recover by waiting.

## Authentication notes

- The personal token is used directly as a Bearer token. Legacy personal tokens do not expire.
- OAuth is not used in v1 (spec Assumptions).
- Never log the `Authorization` header or the token (FR-018).
