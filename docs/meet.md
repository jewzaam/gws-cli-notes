# Meet

Requires `-s meet` in auth login and the Meet API enabled on the GCP project (one-time: <https://console.developers.google.com/apis/api/meet.googleapis.com/overview>).

## Find a Conference Record

```bash
gws meet conferenceRecords list --params '{"filter": "space.meeting_code=\"<CODE>\""}' --format json
```

- Meeting code is from the Google Meet URL (e.g., `unp-byke-sps` from `meet.google.com/unp-byke-sps`)
- Also available in calendar event under `conferenceData.entryPoints[].label` or `conferenceData.conferenceId`
- Response includes `name` (e.g., `conferenceRecords/<ID>`), start/end times, expiry (30 days)

## List Recordings

```bash
gws meet conferenceRecords recordings list --params '{"parent": "conferenceRecords/<ID>"}' --format json
```

- Returns `driveDestination.file` (Drive file ID) and `driveDestination.exportUri` (view URL)
- `state`: `FILE_GENERATED` when ready
- Still subject to Drive download restrictions (`canDownload`, `viewersCanCopyContent`)

## Transcripts

```bash
gws meet conferenceRecords transcripts list --params '{"parent": "conferenceRecords/<ID>"}' --format json
```

- Returns `docsDestination.document` (Google Doc ID) and `docsDestination.exportUri`
- **Transcription is separate from Gemini note-taking** — transcription must be explicitly enabled during the meeting. A meeting can have Gemini notes (summary, action items) but no transcript.
- If no transcript was enabled, this returns empty `{}`
- There is no hidden closed-caption or ASR data available after the fact — if transcription wasn't on, no transcript exists anywhere in the API or Drive

## Structured Transcript Entries

```bash
gws meet conferenceRecords transcripts entries list \
  --params '{"parent": "conferenceRecords/<ID>/transcripts/<TRANSCRIPT_ID>", "pageSize": 100}' --format json
```

- Returns individual utterances with `text`, `participant`, `startTime`, `endTime`, `languageCode`
- Participants are referenced by ID; resolve via `conferenceRecords/<ID>/participants/<PID>`
- Paginated — use `nextPageToken` or `--page-all` to get all entries
- Entries may differ from the Google Docs transcript

## Other Meet Resources

```bash
# List participants (with display names and join/leave times)
gws meet conferenceRecords participants list --params '{"parent": "conferenceRecords/<ID>"}' --format json

# Participant sessions (individual join/leave events per participant)
gws meet conferenceRecords participants participantSessions list \
  --params '{"parent": "conferenceRecords/<ID>/participants/<PID>"}' --format json
```

## Listing All Meetings a User Attended (Date Range Query)

```bash
gws meet conferenceRecords list --params '{"filter":"start_time>=\"2026-03-27T00:00:00Z\" AND start_time<=\"2026-03-28T00:00:00Z\"","pageSize":100}' --format json
```

- Returns all conference records the authenticated user has access to within the date range
- **No participant filter exists** — the API only filters on `space.meeting_code`, `space.name`, `start_time`, `end_time`
- To find which meetings the user actually attended: iterate records and call `participants list` for each, checking for the user's ID
- With no filter at all (`--params '{}'`), returns all records ordered by start time descending (default page size 25)
- This is the only way to discover ad-hoc meetings not on the calendar — meetings the user joined via link or phone without a calendar event
- `end_time IS NULL` filter can find active/in-progress meetings
- Conference records expire after 30 days (`expireTime` field)

## Meeting Lifecycle Detection

- **Active meeting**: conference record has `startTime` but no `endTime` field
- **Concluded meeting**: conference record has both `startTime` and `endTime`
- Check: `"endTime" in record` → concluded, `"endTime" not in record` → still active
- Use case: avoid summarizing in-progress meetings in automation

## Artifact Discovery via Meet API

The Meet API provides an alternative to calendar event attachments for discovering recordings and transcripts:

- `recordings list` returns Drive file IDs and viewable URLs (`driveDestination.exportUri`)
- `transcripts list` returns Google Doc IDs and edit URLs (`docsDestination.exportUri`)
- These work for ad-hoc meetings with no calendar event — the only way to find artifacts for meetings not on the calendar
- Empty `{}` response means no recording/transcript was enabled for that meeting
- Recordings and transcripts are still subject to Drive download restrictions (`canDownload`)
- The viewable/edit URLs are useful even when download is blocked — store them in metadata for manual access

## Resolve Space Name to Meeting Code

```bash
gws meet spaces get --params '{"name": "spaces/<SPACE_ID>"}'
```

- Returns `meetingCode` (the 3-3-3 code like `abc-defg-hij`)
- Useful when a conference record references a space but not a meeting code directly
- meet-enrich caches these mappings in its database to avoid repeated lookups

## Recurring Meetings and Conference Records

Recurring calendar events reuse the same Google Meet meeting code across all instances. A daily standup at `meet.google.com/abc-defg-hij` creates a new conference record each day, but querying `space.meeting_code="abc-defg-hij"` returns ALL of them — potentially hundreds.

**Pitfall: attendance check against wrong instance.** If you query conference records by meeting code and check participants to determine whether the user attended, you must filter records by time to match the specific calendar event instance. Without time filtering, attendance from yesterday's standup will match today's query.

```python
# WRONG — matches attendance from any past instance
records = list_conference_records(meeting_code)
for rec in records:
    if user_in_participants(rec):
        return True  # false positive: user attended yesterday, not today

# CORRECT — filter to records that started during this event's window
event_start = parse(calendar_event["start"])
for rec in records:
    conf_start = parse(rec.get("startTime", ""))
    if conf_start >= event_start and user_in_participants(rec):
        return True
```

The conference record `startTime` is when participants first joined, not when the calendar event was scheduled. For a 9:15 AM meeting, the conference might start at 9:14 or 9:16. Filter using `>=` against the calendar event start — the conference for today's instance will always start at or after the event's scheduled time.

## smartNotes Limitation

The `smartNotes` endpoint returns 403 even with Owner role on the GCP project. The API schema lists `"scopes": []` (no OAuth scopes), indicating it's not available via standard OAuth flows. Likely requires service account with domain-wide delegation or is gated behind Workspace admin APIs. The Gemini notes content is still accessible via the Drive/Docs API using the doc ID from the calendar event attachment.
