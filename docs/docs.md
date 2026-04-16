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

- `DEFAULT_FOR_CURRENT_ACCESS` (default) - resolves to `SUGGESTIONS_INLINE` for edit access, may differ for view-only
- `SUGGESTIONS_INLINE` - raw doc with `suggestedInsertionIds`/`suggestedDeletionIds` on textRun elements
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

## Insert Text at End of Doc

```bash
# Get current end index
END=$(gws docs documents get --params '{"documentId":"DOC_ID"}' | python3 -c "
import sys, json
doc = json.loads(sys.stdin.read())
content = doc.get('body', {}).get('content', [])
print(content[-1].get('endIndex', 1) - 1 if content else 1)
")

# Insert text at end
gws docs documents batchUpdate --params '{"documentId":"DOC_ID"}' --json "{\"requests\":[{\"insertText\":{\"location\":{\"index\":${END}},\"text\":\"Your text here\"}}]}"
```

## Markdown Auto-Detection Does NOT Apply to API Inserts

Google Docs has a "Enable Markdown" preference (Tools → Preferences). This auto-formats markdown when **typed or pasted in the UI**. However, text inserted via the `batchUpdate` API is treated as plain text — markdown syntax (`# heading`, `**bold**`) is NOT converted to formatting, even with the preference enabled. To format text via the API, you must use explicit `updateParagraphStyle` and `updateTextStyle` requests.

## Upload HTML as a Formatted Google Doc (via Drive API)

Requires `https://www.googleapis.com/auth/drive` scope (not `drive.readonly`).

```bash
gws drive files create \
  --json '{"name":"Document Title","mimeType":"application/vnd.google-apps.document"}' \
  --upload path/to/file.html \
  --upload-content-type text/html
```

- `mimeType: application/vnd.google-apps.document` in `--json` tells Drive to convert to a Google Doc
- `--upload-content-type text/html` tells Drive the source format
- Creates a **new document** each time — cannot update an existing doc's content via HTML upload
- HTML formatting (headings, bold, tables, links) is preserved in the conversion
- **Mermaid diagrams are NOT rendered** — they appear as raw code blocks
- Useful for one-time formatted doc creation, not for iterative updates to a stable URL
