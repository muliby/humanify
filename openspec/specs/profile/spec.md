# Voice profile

## Purpose
Turn a corpus into a reusable description of how the person writes: numbers (stylometry), words (style guide) and examples (exemplar index).

## Definitions
- **Stylometric profile**: deterministic numeric features over the corpus, overall and per channel.
- **Style guide**: a natural-language description of the voice, at most 400 words, written by the LLM from the stylometric profile plus sampled exemplars.
- **Exemplar index**: an embedding index over corpus paragraphs used to retrieve the most similar passages to a query.
- **Held-out set**: 20% of Documents, chosen deterministically per person, never used for exemplars or the style guide; reserved for the judge.

## Requirements

### Requirement: Stylometric features
The system SHALL compute, overall and per channel: sentence length mean and standard deviation; paragraph length distribution; function-word frequencies (a fixed list of at least 100 English function words); punctuation rates per 100 words for comma, semicolon, colon, dash, parenthesis, question mark, exclamation mark; reading grade (Flesch–Kincaid); first-sentence pattern counts (question, imperative, first-person, quotation); type–token ratio on 500-word windows.

#### Scenario: Deterministic features
- **GIVEN** a fixture text with hand-computed values
- **WHEN** features are computed twice
- **THEN** every feature equals the hand-computed value within 1% and both runs are identical

#### Scenario: Two authors differ
- **GIVEN** fixture corpora for two different authors
- **WHEN** profiles are built for both
- **THEN** at least 5 features differ by more than one standard deviation

### Requirement: Style guide
The system SHALL generate a style guide of at most 400 words covering: tone, typical structure, openers and closers, recurring phrases, sentence rhythm, and things the person never does.

#### Scenario: Style guide length
- **WHEN** a style guide is generated from the fixture corpus (recorded LLM response)
- **THEN** it has at most 400 words and non-empty sections for each of the six topics

### Requirement: Exemplar retrieval
Given a query passage, the system SHALL return the k most similar corpus paragraphs (default k=8) from non-held-out Documents, with their source and channel.

#### Scenario: Retrieval excludes held-out
- **WHEN** retrieval runs with k=8
- **THEN** no returned paragraph comes from a held-out Document

### Requirement: Held-out split
The system SHALL split each person's corpus deterministically into 80% working and 20% held-out Documents and persist the assignment.

#### Scenario: Stable split
- **WHEN** the profile is rebuilt without corpus changes
- **THEN** the held-out assignment is unchanged

### Requirement: Persistence
Profiles SHALL be stored per person and reused across runs; `person build` rebuilds them.

#### Scenario: Profile reused
- **WHEN** `humanify rewrite` runs after `person build`
- **THEN** no profile is rebuilt and the stored profile is used

## Implementation constraints
- Features MUST be computed with deterministic code, never by asking the LLM.
- Embedding model MUST be local (sentence-transformers) by default; the model name is config.
- The style guide prompt MUST include the numeric profile so the LLM describes the real voice, not a guess.

## Test requirements
- Unit: every feature on fixtures with hand-computed values; split determinism; retrieval exclusion.
- Integration: build a profile for the fixture corpus end to end with recorded LLM responses.
