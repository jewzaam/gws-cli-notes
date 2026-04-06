# Docs

## Read a Google Doc

```bash
gws docs documents get --params '{"documentId":"DOC_ID","includeTabsContent":true}' 2>/dev/null > output.json
```

- `includeTabsContent:true` is required to get all tabs; without it only first tab is in `doc.body.content`
- With it, content is in `doc.tabs[N].documentTab.body.content`
- Document ID is the string between `/d/` and `/edit` in the Google Docs URL
- `-o` flag is for binary responses only; use shell redirect for JSON

## Suggestions

Add `suggestionsViewMode` query param:

- `SUGGESTIONS_INLINE` (default) - raw doc with `suggestedInsertionIds`/`suggestedDeletionIds` on textRun elements
- `PREVIEW_SUGGESTIONS_ACCEPTED` - doc as if all suggestions accepted
- `PREVIEW_WITHOUT_SUGGESTIONS` - doc with all suggestions rejected (original text)

Example:

```bash
gws docs documents get --params '{"documentId":"DOC_ID","includeTabsContent":true,"suggestionsViewMode":"SUGGESTIONS_INLINE"}' 2>/dev/null > output.json
```

Finding suggestions in the JSON: look for `suggestedInsertionIds` (new text) and `suggestedDeletionIds` (removed text) on `textRun` elements. Same suggestion ID on an insertion and deletion = a replacement.

## Write to a Google Doc (batchUpdate)

Requires `https://www.googleapis.com/auth/documents` scope (not `documents.readonly`).

```bash
gws docs documents batchUpdate --params '{"documentId":"DOC_ID"}' --json '{"requests":[...]}'
```
- `--params` carries URL/path parameters (`documentId`)
- `--json` carries the request body (`requests` array) — do NOT use `--body` (not supported)
- Scopes: `documents`, `drive`, or `drive.file`
- Each request in the array is validated before any are applied; if one fails, none are applied

## Add a Tab

```bash
gws docs documents batchUpdate --params '{"documentId":"DOC_ID"}' --json '{"requests":[{"addDocumentTab":{"tabProperties":{"title":"My Tab"}}}]}'
```
- Response includes `replies[0].addDocumentTab.tabProperties.tabId` — the new tab's ID
- For child tabs, add `"parentTabId":"PARENT_TAB_ID"` to `tabProperties`
- `tabProperties.index` controls position within the parent (zero-based)
- **Tab title max length is 50 characters** — API returns 400 if exceeded

## Insert Text into a Tab

```bash
gws docs documents batchUpdate --params '{"documentId":"DOC_ID"}' --json '{"requests":[{"insertText":{"endOfSegmentLocation":{"tabId":"TAB_ID"},"text":"Hello world"}}]}'
```
- `endOfSegmentLocation.tabId` targets a specific tab; omit `tabId` for the first tab
- `insertText` inserts **plain text only** — markdown syntax (`# heading`, `**bold**`) is inserted as literal characters, not converted to formatting
- To apply formatting, use separate `updateParagraphStyle` and `updateTextStyle` requests with index ranges targeting the same `tabId`
- New tabs start with a section break (index 0–1) and a trailing newline; inserted text starts at index 1

## Delete a Tab

```bash
gws docs documents batchUpdate --params '{"documentId":"DOC_ID"}' --json '{"requests":[{"deleteTab":{"tabId":"TAB_ID"}}]}'
```
