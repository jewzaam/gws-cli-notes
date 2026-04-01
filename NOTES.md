# GWS CLI Notes

## Authentication

```bash
gws auth logout
gws auth login --readonly -s docs,drive,calendar,sheets,slides,meet
```

- Scopes were added incrementally: `docs,drive,calendar` -> `+sheets` -> `+slides` -> `+meet`
- After adding a new scope, must logout and re-login for the token to include it
- API enablement errors include URLs to enable the API on the GCP project (one-time action per API)
- The `--format` flag is available on all commands: json (default), table, yaml, csv

## Sheets Usage

Reading spreadsheets requires `spreadsheets.readonly` scope (added via `-s sheets`).

Use the `+read` helper for simple reads:
```bash
gws sheets +read --spreadsheet <SPREADSHEET_ID> --range "Sheet Name" --format table
```

To list all tabs in a spreadsheet (find tab names for a given gid):
```bash
gws sheets spreadsheets get --params '{"spreadsheetId": "<ID>", "fields": "sheets.properties"}'
```

## Docs Usage

### Read a Google Doc

```bash
gws docs documents get --params '{"documentId":"DOC_ID","includeTabsContent":true}' 2>/dev/null > output.json
```
- `includeTabsContent:true` is required to get all tabs; without it only first tab is in `doc.body.content`
- With it, content is in `doc.tabs[N].documentTab.body.content`
- Document ID is the string between `/d/` and `/edit` in the Google Docs URL
- `-o` flag is for binary responses only; use shell redirect for JSON

### Suggestions

Add `suggestionsViewMode` query param:
- `SUGGESTIONS_INLINE` (default) - raw doc with `suggestedInsertionIds`/`suggestedDeletionIds` on textRun elements
- `PREVIEW_SUGGESTIONS_ACCEPTED` - doc as if all suggestions accepted
- `PREVIEW_WITHOUT_SUGGESTIONS` - doc with all suggestions rejected (original text)

Example:
```bash
gws docs documents get --params '{"documentId":"DOC_ID","includeTabsContent":true,"suggestionsViewMode":"SUGGESTIONS_INLINE"}' 2>/dev/null > output.json
```

Finding suggestions in the JSON: look for `suggestedInsertionIds` (new text) and `suggestedDeletionIds` (removed text) on `textRun` elements. Same suggestion ID on an insertion and deletion = a replacement.

## Slides Usage

### Read a Presentation

```bash
gws slides presentations get --params '{"presentationId": "<PRESENTATION_ID>"}' --format json
```
- Presentation ID is the string between `/d/` and `/edit` in the Google Slides URL
- Response includes: title, presentationId, locale, pageSize, slides (with all pageElements, shapes, text, images, tables), layouts, masters
- Each slide has an `objectId` and array of `pageElements`
- Text content is nested: `pageElement.shape.text.textElements[].textRun.content`

### Export Presentation as PDF

```bash
gws drive files export --params '{"fileId": "<PRESENTATION_ID>", "mimeType": "application/pdf"}' -o deck.pdf
```
- Google Workspace files (Slides, Docs, Sheets) cannot be downloaded with `files get --alt media` — use `files export` instead
- `files get` on a Workspace file returns metadata JSON, not binary content
- The `-o` flag saves the exported bytes to a local file
- Response JSON includes `bytes`, `mimeType`, `saved_file`, `status`
- Claude Code's Read tool can then read the PDF directly (with `pages` param for >10 pages) — useful for extracting text, tables, and visual content from slides without JSON parsing
- For programmatic text extraction, the Slides API JSON (`presentations get`) gives structured access to text elements, but requires traversing nested `pageElements > shape > text > textElements > textRun > content` and `table > tableRows > tableCells > text` paths — the PDF export + Read is simpler for one-off content extraction

### Get a Single Slide Page

```bash
gws slides presentations pages get \
  --params '{"presentationId": "<ID>", "pageObjectId": "<SLIDE_OBJECT_ID>"}'
```

## Drive Usage

### Comments (Docs, Slides, and any Google file)

Comments are accessed via the Drive API, not the Docs or Slides APIs:

```bash
gws drive comments list \
  --params '{"fileId": "<FILE_ID>", "fields": "comments(id,content,author,createdTime,modifiedTime,resolved,replies,anchor,quotedFileContent)"}'
```
- The `fields` parameter is **required** — use `comments(*)` for all fields, or specify individual fields
- `fileId` is the same as `documentId` or `presentationId`
- `anchor` — JSON string identifying the location (for Slides: contains `page`, `uid`, `targets`)
- `quotedFileContent.value` — the text that was highlighted when the comment was made
- `replies` — thread replies with author, content, and action (e.g., `resolve`)

```bash
# Get a single comment
gws drive comments get --params '{"fileId": "<ID>", "commentId": "<COMMENT_ID>", "fields": "*"}'
```

### List Files in a Folder

```bash
gws drive files list --params '{"q": "\"<FOLDER_ID>\" in parents", "fields": "files(id,name,mimeType,createdTime,viewedByMeTime)", "orderBy": "createdTime desc", "pageSize": 30}'
```
- `q` uses Drive search query syntax — `"<ID>" in parents` lists a folder's contents
- `fields` controls which file properties are returned (reduces payload)
- `viewedByMeTime` is null/absent if the authenticated user has never opened the file — useful for tracking unwatched demos
- `orderBy` supports: `createdTime`, `modifiedTime`, `name`, `folder` (append `desc` for descending)
- `pageSize` max is 1000; use `nextPageToken` or `--page-all` for pagination

### Export Google Workspace Files as Text

```bash
gws drive files export --params '{"fileId": "<FILE_ID>", "mimeType": "text/plain"}' -o output.txt
```
- Works for Docs, Slides, Sheets — converts to plain text
- For Slides, use `application/pdf` instead (text export loses structure)
- `-o` flag writes directly to file — do NOT pipe (breaks allowlist matching)
- See also: PDF export example under Slides Usage

### File Metadata and Downloads

```bash
# Get metadata (size, duration, resolution, permissions)
gws drive files get --params '{"fileId": "<FILE_ID>", "fields": "id,name,mimeType,size,webViewLink,webContentLink,createdTime,videoMediaMetadata,capabilities"}' --format json

# Download (if capabilities.canDownload is true)
gws drive files get --params '{"fileId": "<FILE_ID>", "alt": "media"}' --output file.ext
```
- `--output` path must be relative to the current working directory (absolute paths are rejected)
- Check `capabilities.canDownload` first — org admins or file owners can restrict downloads
- `videoMediaMetadata` provides `durationMillis`, `height`, `width` for video files
- When `canDownload: true`, pipe to ffplay: `gws drive files get --params '{"fileId": "...", "alt": "media"}' 2>/dev/null | ffplay -`

### Download via `files download`

```bash
gws drive files download --params '{"fileId": "<FILE_ID>"}' --output recording.webm
```
- Alternative to `files get --params '{"alt": "media"}'` — simpler syntax for binary downloads
- `--output` path must be relative (same restriction as `files get`)
- Used by meet-enrich for downloading meeting recordings

### Download Restrictions (org policy)

When `viewersCanCopyContent: false` / `canDownload: false`:
- The Drive API refuses to serve bytes regardless of endpoint (`files.get?alt=media` returns 403, `files.download` returns 500)
- Browser streaming still works because Google's embedded player uses internal infrastructure
- No public API provides an alternative streaming URL
- Only the file owner or a Drive admin can change this

## Calendar Usage

### Show Today's Agenda (Quick)

```bash
gws calendar +agenda --today --format table
```
- Shows events across **all** calendars (PTO, team calendars, personal, etc.)
- Use `--calendar <NAME>` to filter to one calendar
- Other options: `--tomorrow`, `--week`, `--days <N>`

### List Events on Primary Calendar (Detailed)

```bash
gws calendar events list --params '{"calendarId": "primary", "timeMin": "2026-03-20T00:00:00-04:00", "timeMax": "2026-03-21T00:00:00-04:00", "singleEvents": true, "orderBy": "startTime"}' --format json
```
- `singleEvents: true` expands recurring events into individual instances
- `orderBy: startTime` requires `singleEvents: true`
- Response includes `attendees[]` with `self: true` and `responseStatus` (accepted/declined/tentative/needsAction)

### Response Status (Accepted / Declined / Tentative)

- **No server-side filter** — the Google Calendar API does not support filtering by response status
- Must query events and check `attendees[].responseStatus` where `attendees[].self == true` in post-processing
- Events with no `attendees` array are self-owned (effectively accepted)
- Possible values: `accepted`, `declined`, `tentative`, `needsAction`

### Meeting Artifacts from Calendar Events

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

### Working Location

- Shows as an event with `eventType: workingLocation`
- Properties in `workingLocationProperties.type`: `homeOffice`, `officeLocation`, `customLocation`
- All-day event, transparent (doesn't block calendar)

### Working Hours

- **Not available via Calendar API** — working hours are a UI-only setting in Google Calendar
- Not exposed in `settings list`, `calendarList get`, or any other endpoint

### Calendar Settings

```bash
gws calendar settings list --params '{}' --format json
```
- Returns: timezone, locale, weekStart, format24HourTime, defaultEventLength, showDeclinedEvents, hideWeekends

## Meet Usage

Requires `-s meet` in auth login and the Meet API enabled on the GCP project (one-time: <https://console.developers.google.com/apis/api/meet.googleapis.com/overview>).

### Find a Conference Record

```bash
gws meet conferenceRecords list --params '{"filter": "space.meeting_code=\"<CODE>\""}' --format json
```
- Meeting code is from the Google Meet URL (e.g., `unp-byke-sps` from `meet.google.com/unp-byke-sps`)
- Also available in calendar event under `conferenceData.entryPoints[].label` or `conferenceData.conferenceId`
- Response includes `name` (e.g., `conferenceRecords/<ID>`), start/end times, expiry (30 days)

### List Recordings

```bash
gws meet conferenceRecords recordings list --params '{"parent": "conferenceRecords/<ID>"}' --format json
```
- Returns `driveDestination.file` (Drive file ID) and `driveDestination.exportUri` (view URL)
- `state`: `FILE_GENERATED` when ready
- Still subject to Drive download restrictions (`canDownload`, `viewersCanCopyContent`)

### Transcripts

```bash
gws meet conferenceRecords transcripts list --params '{"parent": "conferenceRecords/<ID>"}' --format json
```
- Returns `docsDestination.document` (Google Doc ID) and `docsDestination.exportUri`
- **Transcription is separate from Gemini note-taking** — transcription must be explicitly enabled during the meeting. A meeting can have Gemini notes (summary, action items) but no transcript.
- If no transcript was enabled, this returns empty `{}`
- There is no hidden closed-caption or ASR data available after the fact — if transcription wasn't on, no transcript exists anywhere in the API or Drive

### Structured Transcript Entries

```bash
gws meet conferenceRecords transcripts entries list \
  --params '{"parent": "conferenceRecords/<ID>/transcripts/<TRANSCRIPT_ID>", "pageSize": 100}' --format json
```
- Returns individual utterances with `text`, `participant`, `startTime`, `endTime`, `languageCode`
- Participants are referenced by ID; resolve via `conferenceRecords/<ID>/participants/<PID>`
- Paginated — use `nextPageToken` or `--page-all` to get all entries
- Entries may differ from the Google Docs transcript

### Other Meet Resources

```bash
# List participants (with display names and join/leave times)
gws meet conferenceRecords participants list --params '{"parent": "conferenceRecords/<ID>"}' --format json

# Participant sessions (individual join/leave events per participant)
gws meet conferenceRecords participants participantSessions list \
  --params '{"parent": "conferenceRecords/<ID>/participants/<PID>"}' --format json
```

### Listing All Meetings a User Attended (Date Range Query)

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

### Meeting Lifecycle Detection

- **Active meeting**: conference record has `startTime` but no `endTime` field
- **Concluded meeting**: conference record has both `startTime` and `endTime`
- Check: `"endTime" in record` → concluded, `"endTime" not in record` → still active
- Use case: avoid summarizing in-progress meetings in automation

### Artifact Discovery via Meet API

The Meet API provides an alternative to calendar event attachments for discovering recordings and transcripts:
- `recordings list` returns Drive file IDs and viewable URLs (`driveDestination.exportUri`)
- `transcripts list` returns Google Doc IDs and edit URLs (`docsDestination.exportUri`)
- These work for ad-hoc meetings with no calendar event — the only way to find artifacts for meetings not on the calendar
- Empty `{}` response means no recording/transcript was enabled for that meeting
- Recordings and transcripts are still subject to Drive download restrictions (`canDownload`)
- The viewable/edit URLs are useful even when download is blocked — store them in metadata for manual access

### Resolve Space Name to Meeting Code

```bash
gws meet spaces get --params '{"name": "spaces/<SPACE_ID>"}'
```
- Returns `meetingCode` (the 3-3-3 code like `abc-defg-hij`)
- Useful when a conference record references a space but not a meeting code directly
- meet-enrich caches these mappings in its database to avoid repeated lookups

### smartNotes Limitation

The `smartNotes` endpoint returns 403 even with Owner role on the GCP project. The API schema lists `"scopes": []` (no OAuth scopes), indicating it's not available via standard OAuth flows. Likely requires service account with domain-wide delegation or is gated behind Workspace admin APIs. The Gemini notes content is still accessible via the Drive/Docs API using the doc ID from the calendar event attachment.

## People API Usage

### Get Person Info

```bash
gws people people get --params '{"resourceName": "people/<NUMERIC_ID>", "personFields": "names,emailAddresses"}'
```
- `resourceName`: `people/me` for authenticated user, `people/<numeric_id>` for a specific person
- `personFields`: comma-separated list of fields to return (e.g., `names`, `emailAddresses`, `photos`)
- Response: `names[0].displayName`, `emailAddresses[0].value`, `resourceName`
- The Meet API uses `users/<id>` prefix while People API uses `people/<id>` — the numeric ID is the same
- meet-enrich uses this to resolve Meet participant IDs to email addresses and display names

### Get Authenticated User

```bash
gws people people get --params '{"resourceName": "people/me", "personFields": "names,emailAddresses"}'
```
- Returns the authenticated user's `resourceName` (e.g., `people/12345`)
- Used to identify the current user in Meet participant lists (convert `people/` → `users/` prefix)

## Auth Status

### Check Token and Scopes

```bash
gws auth status
```
- Returns JSON with `user` (email), `scopes`, `expired`, `token_type`
- Useful for programmatic checks — meet-enrich uses this to resolve the authenticated user's email
- No `--params` or `--format` needed

## Docs API Content Structure (for Markdown Conversion)

Analyzed a Gemini-generated meeting notes doc (two tabs: Notes, Transcript).

### Tab Structure

- With `includeTabsContent:true`, tabs are in `doc.tabs[]`
- Each tab: `tabProperties.title`, `tabProperties.tabId`, `documentTab.body.content[]`
- Without `includeTabsContent`, only the first tab appears in `doc.body.content`

### Paragraph Styles (namedStyleType)

Only three styles observed in Gemini meeting docs:
- `HEADING_2` — tab title, section headers
- `HEADING_3` — subsection headers (e.g., "Summary", "Details", timestamps in transcript)
- `NORMAL_TEXT` — everything else (body content, attendee lists, attachments)
- No `HEADING_1` — the document title is metadata, not a structural element
- No list styles via `namedStyleType`; bullets appear as Unicode characters (e.g., `\ue907`) in text content

### Element Types (within paragraph.elements[])

- `textRun` — primary content. Has `content` (string) and `textStyle` (bold, strikethrough, link, etc.)
- `person` — inline @mention. Has `personProperties.name` and `personProperties.email`. Can have `textStyle.strikethrough`
- `richLink` — smart chip / embedded link. Has `richLinkProperties.title` and `richLinkProperties.uri`
- `dateElement` — inline date chip (present but content structure not fully explored)
- `sectionBreak` — top-level structural element (not inside paragraphs), marks section boundaries
- `horizontalRule` — not observed in this doc but documented in the API
- `table` — not observed in this doc but documented in the API
- `inlineObjectElement` — images, not observed here

### Special Characters in Text Content

- `\x0b` (vertical tab, `U+000B`) — soft line break within a paragraph. Used heavily in transcript tab where multiple speaker turns are in one paragraph. Map to `\n` in markdown.
- `\xa0` (non-breaking space, `U+00A0`) — appears at start of transcript paragraphs. Strip or normalize to regular space.
- `\ue907` (private use area) — Gemini's custom bullet icon in summary/detail paragraphs. Map to a markdown list bullet (`-`) or strip.
- `\n` — paragraph-ending newline (always the last char in `textRun.content`)

### Structural Patterns in Gemini Meeting Docs

**Notes tab:**
1. H2: meeting title
2. NORMAL_TEXT: "Invited" + person elements (attendee list). Strikethrough persons = removed from invite.
3. NORMAL_TEXT: "Attachments" + richLink elements (links to slides, backlog, etc.)
4. NORMAL_TEXT: "Meeting records" + richLink elements (transcript doc, recording link)
5. H3: "Summary"
6. NORMAL_TEXT: `\ue907` prefix + bold topic headers + description text (Gemini-generated summary bullets)
7. H3: "Details"
8. NORMAL_TEXT: bold topic header + `:` + detailed text with inline timestamp references (e.g., `00:03:52`)

**Transcript tab:**
1. H2: meeting title + "- Transcript"
2. Repeating pattern: H3 timestamp (e.g., `00:05:15`) followed by NORMAL_TEXT with `\x0b`-separated speaker turns
3. Each speaker turn: bold speaker name + `:` + spoken text
4. Multiple speakers in a single paragraph, separated by `\x0b`

### Text Style Properties (textRun.textStyle)

- `bold` — used for topic headers (Notes) and speaker names (Transcript)
- `strikethrough` — used on removed attendees (person elements)
- `link.url` — hyperlinks (check `textStyle.link` for linked text)
- Other possible styles (not observed here): `italic`, `underline`, `fontSize`, `foregroundColor`, `weightedFontFamily`

### Markdown Conversion Mapping

| Docs API | Markdown |
|----------|----------|
| `HEADING_1` | `#` |
| `HEADING_2` | `##` |
| `HEADING_3` | `###` |
| `NORMAL_TEXT` | plain paragraph |
| `textRun` with bold | `**text**` |
| `textRun` with strikethrough | `~~text~~` |
| `textRun` with link | `[text](url)` |
| `person` | `@Name` or `~~@Name~~` if strikethrough |
| `richLink` | `[title](uri)` |
| `\x0b` in content | line break (`\n`) |
| `\ue907` prefix | `-` (list item) |
| `sectionBreak` | blank line or `---` |
| `horizontalRule` | `---` |

## Gmail Usage

### Search Messages

```bash
gws gmail users messages list --params '{"userId": "me", "q": "is:unread newer_than:7d (ANSTRAT OR nexus)", "maxResults": 20}'
```
- `q` uses Gmail search syntax — same operators as the Gmail search bar (`is:unread`, `from:`, `subject:`, `newer_than:`, `OR`, parentheses for grouping)
- Returns message IDs only — fetch headers or full content separately
- `maxResults` max is 500; use `nextPageToken` for pagination

### Get Message Headers (Metadata Only)

```bash
gws gmail users messages get --params '{"userId": "me", "id": "<MESSAGE_ID>", "format": "metadata", "metadataHeaders": ["Subject", "Date", "From", "To", "Cc"]}'
```
- `format: "metadata"` returns only headers specified in `metadataHeaders` — lightweight
- Headers are in `payload.headers[]` as `{name, value}` pairs

### Get Full Message Content

```bash
gws gmail users messages get --params '{"userId": "me", "id": "<MESSAGE_ID>", "format": "full"}'
```
- Returns full MIME structure including body content
- Body is base64url-encoded in `payload.body.data` or nested in `payload.parts[].body.data`
- `format` options: `minimal` (IDs only), `metadata` (headers), `full` (everything), `raw` (RFC 2822)

### Read Helper (Shortcut)

```bash
gws gmail +read --id <MESSAGE_ID> --headers --format json
```
- Convenience wrapper — `--headers` returns just headers without the full MIME payload

## Scope Requirements

| Service | Operation | Scope URL | Login flag |
|---------|-----------|-----------|------------|
| Calendar | Read events, settings | `https://www.googleapis.com/auth/calendar.readonly` | `-s calendar` |
| Calendar | RSVP / patch events (PA dashboard) | `https://www.googleapis.com/auth/calendar.events` | requires `--scopes` (see below) |
| Docs | Read content, suggestions | `https://www.googleapis.com/auth/documents.readonly` | `-s docs` |
| Drive | Comments, file metadata, downloads | `https://www.googleapis.com/auth/drive.readonly` | `-s drive` |
| Slides | Read presentations | `https://www.googleapis.com/auth/presentations.readonly` | `-s slides` |
| Sheets | Read spreadsheets | `https://www.googleapis.com/auth/spreadsheets.readonly` | `-s sheets` |
| Meet | Conference records, recordings, transcripts, participants | `https://www.googleapis.com/auth/meetings.space.readonly` | `-s meet` |
| Gmail | Read messages | `https://www.googleapis.com/auth/gmail.readonly` | `-s gmail` |
| People | Get self (`people/me`) | `https://www.googleapis.com/auth/userinfo.profile` | auto-included |
| People | Resolve other users (`people/<id>`) | `https://www.googleapis.com/auth/directory.readonly` | requires `--scopes` |
| Meet | Smart notes | unavailable via OAuth | n/a |

### Login with all scopes (PA dashboard + meet-enrich)

The PA dashboard requires `calendar.events` (write) for RSVP. The `-s calendar` flag only grants `calendar.readonly`. To get both read and write, use explicit `--scopes`:

```bash
gws auth logout
gws auth login --scopes https://www.googleapis.com/auth/calendar.events,https://www.googleapis.com/auth/calendar.readonly,https://www.googleapis.com/auth/documents.readonly,https://www.googleapis.com/auth/drive.readonly,https://www.googleapis.com/auth/gmail.readonly,https://www.googleapis.com/auth/meetings.space.readonly,https://www.googleapis.com/auth/presentations.readonly,https://www.googleapis.com/auth/spreadsheets.readonly,https://www.googleapis.com/auth/directory.readonly
```

The `--readonly` + `-s` shorthand flags don't support write scopes. Use `--scopes` with full URLs when any write scope is needed.

Must logout first — cached token won't pick up new scopes without a fresh login.
