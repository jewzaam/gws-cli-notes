# Calendar

## Show Today's Agenda (Quick)

```bash
gws calendar +agenda --today --format table
```

- Shows events across **all** calendars (PTO, team calendars, personal, etc.)
- Use `--calendar <NAME>` to filter to one calendar
- Other options: `--tomorrow`, `--week`, `--days <N>`

## List Events on Primary Calendar (Detailed)

```bash
gws calendar events list --params '{"calendarId": "primary", "timeMin": "2026-03-20T00:00:00-04:00", "timeMax": "2026-03-21T00:00:00-04:00", "singleEvents": true, "orderBy": "startTime"}' --format json
```

- `singleEvents: true` expands recurring events into individual instances
- `orderBy: startTime` requires `singleEvents: true`
- Response includes `attendees[]` with `self: true` and `responseStatus` (accepted/declined/tentative/needsAction)
- **Paginated** — defaults to 250 results max. A busy shared calendar over 30 days will hit this limit. Must follow `nextPageToken` to get all events (see [Pagination](../CLAUDE.md#pagination))

## Response Status (Accepted / Declined / Tentative)

- **No server-side filter** — the Google Calendar API does not support filtering by response status
- Must query events and check `attendees[].responseStatus` where `attendees[].self == true` in post-processing
- Events with no `attendees` array are self-owned (effectively accepted)
- Possible values: `accepted`, `declined`, `tentative`, `needsAction`

## Meeting Artifacts from Calendar Events

Calendar events from Google Meet include attachments with meeting artifacts:

- Attachments are in `event.attachments[]` with `fileId`, `fileUrl`, `mimeType`, `title`
- Common attachment types:
  - `video/mp4` — Meeting recording
  - `text/plain` — Chat log
  - `application/vnd.google-apps.document` — Notes (human or Gemini-generated), Transcript
- Gemini notes docs contain: summary, detailed notes with timestamps, suggested next steps, and (if transcription was enabled) **full transcript**
- Human meeting notes are a separate doc (titled "Notes - \<meeting name\>") with running notes across meetings in tabs
- To fetch doc content: `gws docs documents get --params '{"documentId": "<fileId>", "includeTabsContent": true}'`
- Recording/chat files may return 404 if you don't have access (e.g., declined the meeting)
- Recordings are owned by the meeting organizer, not attendees

## Working Location

- Shows as an event with `eventType: workingLocation`
- Properties in `workingLocationProperties.type`: `homeOffice`, `officeLocation`, `customLocation`
- All-day event, transparent (doesn't block calendar)

## Working Hours

- **Not available via Calendar API** — working hours are a UI-only setting in Google Calendar
- Not exposed in `settings list`, `calendarList get`, or any other endpoint

## Create Events

```bash
gws calendar events insert --params '{"calendarId": "<CALENDAR_ID>"}' --json '{"summary": "Event Title", "start": {"dateTime": "2026-04-09T15:00:00-04:00"}, "end": {"dateTime": "2026-04-09T17:00:00-04:00"}}'
```

- `--params` carries URL/query parameters (`calendarId`)
- `--json` carries the request body (event object: `summary`, `start`, `end`, etc.)
- **Do NOT put the event body inside `requestBody` within `--params`** — that results in `Missing end time` errors. The body goes in `--json`, not `--params`
- `calendarId` defaults to `primary` if omitted
- `dateTime` uses RFC3339 with timezone offset (e.g., `-04:00` for EDT)
- Requires `calendar.events` scope (not just `calendar.readonly`)

## Settings

```bash
gws calendar settings list --params '{}' --format json
```

- Returns: timezone, locale, weekStart, format24HourTime, defaultEventLength, showDeclinedEvents, hideWeekends
