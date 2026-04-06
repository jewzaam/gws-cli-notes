# Drive

## Comments (Docs, Slides, and any Google file)

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

## List Files in a Folder

```bash
gws drive files list --params '{"q": "\"<FOLDER_ID>\" in parents", "fields": "files(id,name,mimeType,createdTime,viewedByMeTime)", "orderBy": "createdTime desc", "pageSize": 30}'
```

- `q` uses Drive search query syntax — `"<ID>" in parents` lists a folder's contents
- `fields` controls which file properties are returned (reduces payload)
- `viewedByMeTime` is null/absent if the authenticated user has never opened the file — useful for tracking unwatched demos
- `orderBy` supports: `createdTime`, `modifiedTime`, `name`, `folder` (append `desc` for descending)
- `pageSize` max is 1000; use `nextPageToken` or `--page-all` for pagination

## Export Google Workspace Files as Text

```bash
gws drive files export --params '{"fileId": "<FILE_ID>", "mimeType": "text/plain"}' -o output.txt
```

- Works for Docs, Slides, Sheets — converts to plain text
- For Slides, use `application/pdf` instead (text export loses structure)
- `-o` flag writes directly to file — do NOT pipe (breaks allowlist matching)
- See also: PDF export example under [Slides](slides.md#export-presentation-as-pdf)

## File Metadata and Downloads

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

## Download via `files download`

```bash
gws drive files download --params '{"fileId": "<FILE_ID>"}' --output recording.webm
```

- Alternative to `files get --params '{"alt": "media"}'` — simpler syntax for binary downloads
- `--output` path must be relative (same restriction as `files get`)
- Used by meet-enrich for downloading meeting recordings

## Download Restrictions (org policy)

When `viewersCanCopyContent: false` / `canDownload: false`:

- The Drive API refuses to serve bytes regardless of endpoint (`files.get?alt=media` returns 403, `files.download` returns 500)
- Browser streaming still works because Google's embedded player uses internal infrastructure
- No public API provides an alternative streaming URL
- Only the file owner or a Drive admin can change this
