# Ingest

## Purpose
Collect a named person's own writing from the web and local files into a deduplicated, channel-tagged corpus.

## Definitions
- **Person**: a named individual identified by a slug (e.g. `mulix`) with a list of source URLs and/or local files.
- **Document**: one piece of the person's writing: text, source (URL or path), channel (`blog` | `paper` | `post` | `talk` | `other`), date if known.
- **Corpus**: all Documents for a Person.

## Requirements

### Requirement: Register a person
The system SHALL let the user create a Person with a slug and add sources (URLs, local files, pasted text) to it.

#### Scenario: Add URL sources
- **WHEN** the user adds two URLs to person `mulix`
- **THEN** person `mulix` lists both URLs as pending sources

### Requirement: Collect from URLs
The system SHALL fetch each URL, extract the main authored text, and store one Document per page or post. For index pages it SHALL follow links to individual posts on the same host up to a configured depth (default 1).

#### Scenario: Blog index
- **GIVEN** a URL that lists posts
- **WHEN** ingest runs with depth 1
- **THEN** each linked post on the same host becomes its own Document with channel `blog`

#### Scenario: Boilerplate removed
- **GIVEN** a page with navigation, footer and comments around an article
- **WHEN** it is ingested
- **THEN** the Document text contains the article body and none of the navigation, footer or comment text

### Requirement: Collect from local files and pasted text
The system SHALL accept `.md`, `.txt`, `.pdf` and `.pptx` files and raw pasted text as sources.

#### Scenario: PPTX source
- **WHEN** a `.pptx` file is added and ingested
- **THEN** one Document is created whose text is the slide text in slide order, channel `talk`

### Requirement: Deduplicate
The system SHALL discard a Document whose normalized text is at least 95% similar to an existing Document of the same Person.

#### Scenario: Near-duplicate dropped
- **GIVEN** a Document already in the corpus
- **WHEN** a second source yields text that differs only in whitespace and a trailing line
- **THEN** no second Document is created and the ingest report counts one duplicate

### Requirement: Channel attribution
Every Document SHALL carry a channel so the profile can model per-channel differences. Channel is inferred from source type and URL pattern and MAY be overridden by the user.

#### Scenario: Channel override
- **WHEN** the user adds a URL with `--channel paper`
- **THEN** every Document from that URL has channel `paper`

### Requirement: Ingest report
Ingest SHALL print a report: documents added, duplicates dropped, failures with reasons, total words per channel.

#### Scenario: Report counts
- **WHEN** ingest finishes
- **THEN** the report's added + duplicates + failures equals the number of pages and files attempted

## Implementation constraints
- MUST respect robots.txt and rate-limit to at most 1 request/second per host.
- MUST store only the person's authored text; strip comments, navigation and boilerplate.
- SHOULD extract PDF and PPTX text with existing libraries (pypdf, python-pptx), not custom parsers.
- MUST work fully offline for local files and in tests (fixture HTML served from disk).

## Test requirements
- Unit: dedupe threshold; channel inference and override; boilerplate stripping on three fixture pages.
- Integration: ingest a local fixture site (no network) and assert the report counts.

## Acceptance
- Ingesting the fixture copy of a personal site yields the expected number of Documents, zero duplicates, each with a channel.
