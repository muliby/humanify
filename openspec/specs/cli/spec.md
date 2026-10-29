# CLI

## Purpose
Everything is reachable from the command line, scriptable with `--json`, so the owner can run and evaluate Humanify without reading code.

## Requirements

### Requirement: Commands
The `humanify` CLI SHALL provide:
- `humanify person add <slug> [--url URL]... [--file PATH]... [--channel C]`
- `humanify person list`
- `humanify person build <slug>` — ingest all sources, then build the profile
- `humanify person show <slug>` — profile summary and calibration
- `humanify rewrite --person <slug> --in <file> --out <file> [--attempts N]`
- `humanify judge --person <slug> --in <file>`
- `humanify bench --person <slug> --in <file> --against <file>` — judge two candidates side by side

#### Scenario: Help works without models
- **WHEN** `humanify --help` runs on a fresh install with no models downloaded
- **THEN** it exits 0 and lists all commands

### Requirement: JSON output
Every command SHALL accept `--json` and then print a single JSON object and nothing else on stdout.

#### Scenario: JSON only
- **WHEN** `humanify judge --json ...` runs
- **THEN** stdout parses as one JSON object containing `verdict`, `scores` and `diagnosis`

### Requirement: Exit codes
Commands SHALL exit 0 on success, 1 on usage error, 2 on a failed judge verdict.

## Later changes (out of scope for v0)
- MCP server exposing rewrite and judge.
- PPTX and DOCX round-tripping.
