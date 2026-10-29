# Rewrite

## Purpose
Rework a given text so it matches a person's voice while preserving meaning, facts and structure.

## Requirements

### Requirement: Style transfer
Given a source text and a Person with a built profile, the system SHALL produce a rewritten text that preserves meaning and factual content and matches the person's profile.

#### Scenario: Meaning preserved
- **GIVEN** five fixture drafts
- **WHEN** each is rewritten (recorded LLM responses)
- **THEN** bidirectional NLI entailment between source and output scores at least 0.9 for each

### Requirement: No new facts
The output SHALL NOT contain named entities, numbers, dates, URLs or quotations absent from the source.

#### Scenario: Entity diff is empty
- **WHEN** a draft is rewritten
- **THEN** the set of entities and numbers in the output minus those in the source is empty

### Requirement: Structure preservation
Headings, list items, slide boundaries and code blocks SHALL be preserved in count and order unless the style guide explicitly reshapes them. Code blocks SHALL be byte-identical.

#### Scenario: Markdown structure kept
- **GIVEN** a Markdown draft with 4 headings, 2 lists and 1 code block
- **WHEN** it is rewritten
- **THEN** the output has 4 headings in the same order, 2 lists with the same item counts, and the identical code block

### Requirement: Grounded prompting
The rewrite prompt SHALL include the style guide, the k=8 most similar exemplars, the stylometric targets as numbers, and the source text. It SHALL NOT include the full corpus.

#### Scenario: Prompt is bounded
- **WHEN** a 5,000-word corpus person rewrites a 300-word draft
- **THEN** the prompt sent to the LLM is under 6,000 tokens

### Requirement: Chunking
Inputs longer than a configured limit SHALL be rewritten section by section with a running summary passed between chunks for consistency.

#### Scenario: Long input chunked
- **GIVEN** a 6,000-word Markdown draft and a chunk limit of 1,500 words
- **WHEN** it is rewritten
- **THEN** at least four LLM calls are made and the output keeps all headings in order

### Requirement: Iteration with judge feedback
The system SHALL support regeneration with judge feedback injected into the prompt, up to N attempts (default 3), keeping the best-scoring attempt.

#### Scenario: Best attempt kept
- **GIVEN** three attempts with judge scores 0.4, 0.7, 0.6
- **THEN** the output is attempt 2 and the report lists all three scores

## Implementation constraints
- MUST use the Anthropic SDK; model from config.
- MUST NOT add facts, quotes or claims not present in the source.
- Supported inputs in v0: plain text and Markdown. PPTX/DOCX round-trip is a later change.

## Test requirements
- Unit: structure preservation; entity/number diff; prompt size bound; best-attempt selection.
- Integration: end to end on a 500-word fixture with recorded LLM responses.
