# gws-cli-notes

[![Markdown Lint](https://github.com/jewzaam/gws-cli-notes/actions/workflows/markdown-lint.yml/badge.svg)](https://github.com/jewzaam/gws-cli-notes/actions/workflows/markdown-lint.yml) [![Links](https://github.com/jewzaam/gws-cli-notes/actions/workflows/links.yml/badge.svg)](https://github.com/jewzaam/gws-cli-notes/actions/workflows/links.yml)

Personal reference notes for using the `gws` CLI to interact with Google Workspace APIs.

## Structure

- [CLAUDE.md](CLAUDE.md) — comprehensive index with general topics (auth, scopes, pagination) and links to per-service docs
- [docs/](docs/) — per-service usage notes:
  - [Calendar](docs/calendar.md) — agenda, events, RSVP, meeting artifacts
  - [Docs](docs/docs.md) — read documents, suggestions
  - [Docs API Content Structure](docs/docs-api-content-structure.md) — JSON structure, markdown conversion
  - [Drive](docs/drive.md) — comments, file listing, export, downloads
  - [Gmail](docs/gmail.md) — search, message headers, full content, triage (bulk metadata), format comparison
  - [Meet](docs/meet.md) — conference records, recordings, transcripts, participants
  - [People API](docs/people.md) — resolve user IDs
  - [Sheets](docs/sheets.md) — read spreadsheets, cell formatting
  - [Slides](docs/slides.md) — read presentations, PDF export

## Usage

Structured for AI assistant consumption (e.g., referenced from CLAUDE.md) so that tools like Claude Code can use `gws` commands effectively without rediscovering API quirks, scope requirements, and response structure.
