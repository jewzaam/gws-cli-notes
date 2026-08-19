# Tasks

## List Task Lists

```bash
gws tasks tasklists list
```

- Response: `items[]` with `id` (long opaque string, e.g. `MTgzODMyMjg3ODk1Njk5NjIwOTM6MDow`) and `title`
- `@default` is accepted anywhere a `tasklist` id is expected and resolves to the account's default list — verified identical output to passing that list's explicit id

## List Tasks

```bash
gws tasks tasks list --params '{"tasklist": "@default", "showCompleted": false, "maxResults": 100}'
```

- `tasklist` is the only required parameter (`parameterOrder: ["tasklist"]`)
- Query parameters: `showCompleted`, `showHidden`, `showDeleted`, `showAssigned`, `maxResults`, `pageToken`, `dueMin`, `dueMax`, `completedMin`, `completedMax`
- **`maxResults` defaults to 20** (max 100) — an unfiltered call silently truncates at 20 tasks. See [Pagination](../CLAUDE.md#pagination)
- Tasks assigned from Docs or Chat Spaces are excluded unless `showAssigned: true`
- Google's limits: 20,000 non-hidden tasks per list, 100,000 total
- Undated tasks come back with no `due` key at all: `jq '.items[] | select(.due == null) | .title'`

## `due` Is Date-Only

The API truncates `due` to midnight UTC regardless of any time of day set on the task in the Google Calendar UI:

```
due=2026-08-18T00:00:00.000Z   test, all day
due=2026-08-18T00:00:00.000Z   test, has end time
due=2026-08-18T00:00:00.000Z   test, no end time
```

Three tasks created deliberately as "all day", "has end time", and "no end time" return byte-identical `due` values. **The Tasks API cannot distinguish a timed task from a date-only one.** The time of day exists only in Calendar's representation of the task, so any workflow keyed on "tasks that have a time" cannot be built on this API.

## Tasks Surfacing as Calendar Events

A task with a due date **and a time** is returned by Calendar `events.list` as an ordinary event:

- `eventType: "focusTime"`, empty `attendees`, `organizer.self: true`
- `description` reads: `Changes made to the title, description, or attachments will not be saved. To make edits, please go to: https://tasks.google.com/task/<id>`

A task with a date but no time never appears in `events.list`.

- `eventType: "focusTime"` alone is **not** a reliable marker — genuine Focus Time blocks share it. The `tasks.google.com/task/` link in the description is the discriminator
- The id in that link (16 characters, e.g. `ZOawPUdoteZ6clm-`) is not the Tasks API task id, which is a long opaque string like the tasklist id. **Unverified** whether the two can be mapped; the Task resource was not inspected for a `webViewLink` field
- Consequence: neither API alone can identify "the tasks cluttering my calendar". Calendar knows which have a time but cannot edit them; Tasks can edit them but cannot tell which have a time

## Pagination

`--page-all` is not supported by every endpoint (see [Pagination](../CLAUDE.md#pagination)). For tasks, loop explicitly: read `nextPageToken` from the response and pass it back as `pageToken` until absent.

## Error Output

On an API or discovery failure `gws` exits non-zero (exit code 4 observed) and splits the two streams:

- **stdout** — Google's error envelope: `{"error": {"code": <n>, "message": "...", "reason": "..."}}`
- **stderr** — one line: `error[<kind>]: <message>`

Programmatic callers should read `error.code` from stdout. The exit code alone does not distinguish a permission failure from a transient one.

## Discovery Dependency

Every `gws tasks ...` invocation resolves `https://www.googleapis.com/discovery/v1/apis/tasks/v1/rest` at runtime — including `gws tasks --help` and `gws schema tasks.tasks.list`. In an environment where `googleapis.com` is unreachable, none of them work, so there is no offline way to check a parameter name. Only the static top-level `gws --help` and `gws auth status` work without discovery.

## Scopes

- Read: `https://www.googleapis.com/auth/tasks.readonly` — verified sufficient for `tasklists list` and `tasks list`
- Write: `https://www.googleapis.com/auth/tasks`
- **Unverified**: whether a read-only token returns 403 on `tasks patch`. Not tested
