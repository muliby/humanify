# Judge

## Purpose
Measure whether a candidate text reads as the person's own writing, and explain how it falls short, so "hard to tell" is a number.

## Definitions
- **AV score**: same-author probability between the candidate and the person's held-out Documents, from an authorship-verification model.
- **Human-likeness**: perplexity under a reference language model and an AI-detector probability.
- **Calibration**: the distribution of AV scores obtained by scoring the person's own held-out Documents against the rest of the held-out set.

## Requirements

### Requirement: Authorship verification
The system SHALL compute an AV score for a candidate against the person's held-out set.

#### Scenario: Own writing scores high
- **GIVEN** the fixture person's held-out Documents
- **WHEN** each is scored against the others
- **THEN** at least 80% pass the person's threshold

#### Scenario: Generic draft scores low
- **GIVEN** a generic LLM-written draft on the same topic
- **WHEN** it is scored
- **THEN** it fails the person's threshold

### Requirement: Human-likeness
The system SHALL compute perplexity under a local reference LM and an AI-detector probability.

#### Scenario: Scores reported
- **WHEN** any candidate is judged
- **THEN** the result includes a perplexity value and an AI-detector probability in [0,1]

### Requirement: Calibrated threshold
The threshold SHALL be derived per person from calibration (default: the 20th percentile of own held-out AV scores), not a global constant.

#### Scenario: Threshold is per person
- **GIVEN** two fixture persons
- **WHEN** both are calibrated
- **THEN** their thresholds differ and each is reported with the calibration distribution

### Requirement: Verdict and feedback
The system SHALL return pass/fail, all scores, and a short natural-language diagnosis derived from feature deltas (e.g. "sentences 40% shorter than target; no parentheticals; three em-dashes where the person uses none") suitable for injection into a regeneration prompt.

#### Scenario: Feedback names the largest deltas
- **GIVEN** a candidate whose sentence length is 40% below target and comma rate 50% above
- **WHEN** it is judged
- **THEN** the diagnosis mentions both sentence length and comma rate, largest first

## Implementation constraints
- AV model: start from the open code and data of "Catch Me If You Can? Not Yet" (EMNLP 2025 Findings); document the chosen model in openspec/specs/refs/authorship-verification.md.
- The judge MUST run offline after model download.
- Scores MUST be persisted per run for the benchmark.

## Test requirements
- Unit: threshold derivation; feedback ordering; score persistence.
- Integration: calibration on the fixture corpus; generic draft fails.

## Acceptance
- humanify's output for the owner's corpus passes AV at a rate at least equal to the owner's real held-out Documents (founding doc §6).
