# humanify — Founding Document

*Status: draft v0.1 — 2026-10-29*

## 1. What humanify is

humanify takes two inputs:

1. **AI-generated content** — a blog post, an article, a slide deck, an email, any text.
2. **A person** — a real human, ideally with a public online footprint (blog, papers, talks, LinkedIn/X posts, newsletters).

It reworks the content so that it reads as if that person wrote it. The bar is not "sounds less robotic"; it is **indistinguishability**: a reader familiar with the person, and an authorship-verification model, should have a hard time telling humanify's output from the person's own writing.

It is *not*:

- a generic "AI humanizer" / detector-evasion tool (RewriteAI, Undetectable, etc.);
- a content generator — it only reworks what it is given;
- a brand-voice feature (Jasper, Writer) — the target is a specific individual.

## 2. Why build it — the gap

Survey of existing tools (2026-10-29):

| Tool | What it does | Gap vs. humanify |
|---|---|---|
| **Spiral** (Every) — *state of the art, benchmark* | Learns your voice from samples/channels; `personalize` rewrites in your voice; `humanize` strips AI tells; MCP + CLI. $15/mo personal. | First-person only (you upload your own samples); short-form leaning; no automatic corpus collection from a named person's public footprint; no measured "hard to tell" score. |
| ContentIn, LinkedGrow, Postessia, Apify LinkedIn Voice Trainer | Train a voice profile from LinkedIn posts, generate new posts | LinkedIn-only; generation, not reworking of long/structured content |
| Jasper / Writer / Copy.ai brand voice, HyperWrite style rewriter | Tone presets, famous-author imitation | Not a specific individual; shallow |
| RewriteAI, Undetectable-style humanizers | Generic human-sounding, detector evasion | No target person at all |

humanify's three differentiators:

1. **Automatic voice-profile construction** from a person's whole public footprint, not pasted samples.
2. **Reworking long, structured content** (decks, articles, docs) while preserving structure and meaning.
3. **A measured judge loop** — authorship verification + AI-detection scores — so "hard to tell" is a number, not a vibe.

Research to lean on: STRAP, STYLL, low-resource authorship style transfer with in-context learning, authorship style transfer with policy optimization (authorship-embedding similarity as reward); *Catch Me If You Can? Not Yet* (EMNLP 2025 Findings, open code + data) for evaluation; *How Well Do LLMs Imitate Human Writing Style?* (2025) for the perplexity gap finding (human ≈ 29.5 vs LLM ≈ 15.2).

## 3. The experiment: natural-language programming

humanify is also an experiment: **build it without ever reading the generated code, in any language.**

### Process stack

| Layer | Choice | Why |
|---|---|---|
| Agent | **Claude Code** | Every SDD framework targets it natively; runs subagents, tests, git itself |
| Scaffold | **OpenSpec** (MIT) | Healthiest open SDD project; `openspec/specs/` is the permanent source of truth, every change is a delta proposal |
| Spec structure | Borrowed from **Plain** (Codeplain) | Definitions → functional requirements → implementation constraints → test requirements → acceptance tests |
| Build environment | Borrowed from **SpecSoloist** | One `arrangement.yaml`-style file (language, commands, constraints); *reference specs* for libraries the LLM might hallucinate |
| Eyes | **Tests + CI** | The only feedback loop allowed. A failing test is a spec bug first. |
| Language | **Python** | Stated once in the arrangement, never again. Richest NLP / authorship-verification ecosystem → fewer hallucinations |

### Rules of the experiment

1. Muli only ever opens files under `openspec/` and `tests/` (and reads CI output). The implementation directory is never listed or opened.
2. Every change goes through an OpenSpec change proposal, never a direct instruction like "fix line 42".
3. "Done" is pinned to acceptance tests in the spec, never to inspection.
4. When output is wrong, the first move is to ask: what did the spec fail to say?
5. Keep a `JOURNAL.md` of where the process broke — that is the experiment's data.

## 4. Product architecture (what the specs will ask for)

```
   person URLs ──► ingest ──► corpus ──► profile ──► rewrite ◄── AI draft
                                            │            │
                                            └── judge ◄──┘
                                                   │
                                           score ≥ τ ? ──► output
                                                   │
                                            no ──► regenerate (≤ N) / report
```

1. **Ingest** — given a person's name + URLs, collect text (blog, papers, talk transcripts, social posts); dedupe; keep per-source metadata (people write differently per channel).
2. **Voice profile** — two parts:
   - *Stylometric profile*: sentence-length distribution, function-word ratios, punctuation habits, reading grade, opener patterns, paragraph rhythm.
   - *LLM-written style guide* + an exemplar index (embeddings) for retrieval.
3. **Rewrite** — Claude via API, prompted with the style guide + the 5–10 most similar exemplars (retrieved, not the whole corpus) + the source text. Structure-preserving for decks and docs.
4. **Judge** — authorship-verification embedding similarity (same-author probability vs. the person's held-out texts) + perplexity / AI-detector score. Regenerate until threshold τ or stop and report.
5. **Interface** — CLI first (`humanify --person mulix --in deck.pptx --out deck.humanified.pptx`), MCP server second so agents (Claude Code, Spiral-style tools) can call it.

### Open questions (decide before the first spec)

- Consent: must the target person be the user? v0: yes — user supplies their own URLs. Third-party use is a later, explicit decision.
- Corpus size: how much text is needed before the judge is meaningful? Hypothesis: ≥ 20 pieces / ≥ 10k words.
- "Hard to tell" for whom: v0 targets the verification model; blind human reads are a manual check.
- Decks: v0 handles plain text and Markdown; .pptx round-tripping is a later change.

## 5. Benchmark

**Spiral is the state of the art to compare against.** Protocol:

1. Same AI-written draft.
2. Same corpus of Muli's public writing as samples (for Spiral: uploaded as Writing Style samples; for humanify: ingested from URLs).
3. Run Spiral `personalize` and humanify.
4. Score both with the same authorship-verification model against held-out posts; plus a blind read of 3 samples each.

## 6. First acceptance test (the spec's teeth)

> Given an AI-written draft and the corpus from mulix.org, the authorship-verification model must classify humanify's output as *same author* at ≥ the rate it classifies Muli's actual held-out posts; and a blind reader presented with 3 humanify outputs mixed with 3 real posts must score ≤ chance.

---

# Appendix A — OpenSpec skeleton

Directory layout after `openspec init`:

```
openspec/
  project.md            # context the agent reads every time (this is the arrangement)
  specs/                # source of truth, one file per capability
    ingest.md
    profile.md
    rewrite.md
    judge.md
    cli.md
  changes/              # one folder per proposed change (delta specs + tasks)
tests/                  # acceptance + unit tests (agent-written, Muli-reviewed)
JOURNAL.md              # the experiment's log
```

## A.1 `openspec/project.md` (the "arrangement")

```markdown
# humanify — project context

## Purpose
Rework AI-generated text so it reads as if a specific named person wrote it.

## Tech stack (do not change without a change proposal)
- Python 3.12, managed with uv
- CLI: typer
- LLM: Anthropic SDK (claude-sonnet-4-5 default; model pinned in config)
- Embeddings / authorship verification: sentence-transformers; model named in judge spec
- Storage: SQLite (corpus, profiles, scores)
- Tests: pytest; lint: ruff
- Sandbox: run tests in Docker when network access is needed

## Conventions
- The implementation lives under src/humanify/. The user never reads it.
- Every public function has a type-hinted signature and a docstring.
- No file in src/ may be touched except through a change in openspec/changes/.
- Every capability spec has a "Test requirements" section; tests are generated from it.

## Reference specs
For any library whose API the agent is unsure of, write a `reference` spec in
openspec/specs/refs/<lib>.md documenting only the subset used. No code is generated
from reference specs; they are context.

## Commands
- test: uv run pytest -q
- lint: uv run ruff check src tests
- run: uv run humanify --help
```

## A.2 Capability spec template (Plain-style sections)

Each file under `openspec/specs/` uses this shape:

```markdown
# <Capability>

## Definitions
- *Term*: meaning used throughout this spec.

## Requirements
### Requirement: <name>
The system SHALL ...
#### Scenario: <name>
- GIVEN ...
- WHEN ...
- THEN ...

## Implementation constraints
- MUST / MUST NOT statements about libraries, limits, formats.

## Test requirements
- Unit: ...
- Integration: ...
- Fixtures needed: ...

## Acceptance tests
- Concrete, runnable, with numbers.
```

## A.3 `openspec/specs/ingest.md` (draft)

```markdown
# Ingest

## Definitions
- *Person*: a named individual identified by a slug (e.g. `mulix`) and a list of source URLs.
- *Document*: one piece of the person's writing with text, source URL, channel
  (blog | paper | post | talk | other), date if known.
- *Corpus*: all documents for a person.

## Requirements
### Requirement: Collect from URLs
The system SHALL fetch each configured URL, extract main text, and store one Document
per page/post. It SHALL follow blog index pages to individual posts up to a configured depth.
#### Scenario: blog index
- GIVEN a URL that lists posts
- WHEN ingest runs with depth 1
- THEN each linked post becomes its own Document with channel=blog

### Requirement: Deduplicate
The system SHALL drop documents whose normalized text is ≥ 95% similar to an existing one.

### Requirement: Attribute by channel
Each Document SHALL carry a channel so the profile can model per-channel differences.

### Requirement: Local and pasted sources
The system SHALL also accept local files (.md, .txt, .pdf, .pptx) and raw pasted text.

## Implementation constraints
- MUST respect robots.txt and rate-limit to 1 request/s per host.
- MUST NOT store anything other than the person's own authored text (strip comments, navigation, boilerplate).
- SHOULD extract text from PDFs and PPTX with existing libraries, not custom parsers.

## Test requirements
- Unit: dedupe threshold; channel assignment; boilerplate stripping on 3 fixture pages.
- Integration: ingest a local fixture site (no network) and count documents.

## Acceptance tests
- Ingesting the fixture copy of mulix.org yields ≥ N documents, zero duplicates, each with a channel.
```

## A.4 `openspec/specs/profile.md` (draft)

```markdown
# Voice profile

## Definitions
- *Stylometric profile*: numeric features over the corpus.
- *Style guide*: a short LLM-written natural-language description of the voice.
- *Exemplar index*: embedding index over corpus paragraphs for retrieval.

## Requirements
### Requirement: Stylometric features
The system SHALL compute per person (and per channel): sentence-length mean/sd,
paragraph-length distribution, function-word frequencies, punctuation rates
(comma, semicolon, dash, parenthesis, question, exclamation), reading grade,
first-sentence patterns, vocabulary richness (type–token ratio on fixed windows).

### Requirement: Style guide
The system SHALL generate a ≤ 400-word style guide from the stylometric profile plus
sampled exemplars, covering tone, structure, openers/closers, recurring phrases,
things the person never does.

### Requirement: Exemplar retrieval
Given a passage, the system SHALL return the k most similar corpus paragraphs by embedding.

### Requirement: Persistence
Profiles SHALL be stored and reusable across runs; rebuilt on demand.

## Implementation constraints
- Features MUST be computed with deterministic code, not by asking the LLM.
- Embedding model MUST be local (sentence-transformers) by default.

## Test requirements
- Unit: each feature on a known text yields the expected value (fixtures with hand-computed numbers).
- Integration: build a profile for the fixture corpus; style guide length ≤ 400 words.

## Acceptance tests
- Profiles for two different fixture authors differ on ≥ 5 of the stylometric features by > 1 sd.
```

## A.5 `openspec/specs/rewrite.md` (draft)

```markdown
# Rewrite

## Requirements
### Requirement: Style transfer
Given a source text and a person, the system SHALL produce a rewritten text that
preserves meaning and factual content and matches the person's profile.

### Requirement: Structure preservation
Headings, lists, slide boundaries and code blocks SHALL be preserved in position
and count unless the style guide explicitly reshapes them.

### Requirement: Grounded prompting
The rewrite prompt SHALL include: the style guide, the k=8 most similar exemplars,
the stylometric targets (as numbers), and the source text. It SHALL NOT include the full corpus.

### Requirement: Iteration
The system SHALL support regeneration with judge feedback injected (see judge spec),
up to N attempts (default 3).

## Implementation constraints
- MUST use the Anthropic SDK; model from config.
- MUST NOT add facts, quotes or claims not present in the source.
- Chunk long inputs by section; keep a running summary for cross-chunk consistency.

## Test requirements
- Unit: structure-preservation on fixture docs (heading/list/slide counts equal).
- Unit: no-new-facts check via entity/number diff against the source.
- Integration (recorded LLM responses): end-to-end on a 500-word fixture.

## Acceptance tests
- On 5 fixture drafts, meaning-preservation score (NLI entailment both ways) ≥ 0.9.
```

## A.6 `openspec/specs/judge.md` (draft)

```markdown
# Judge

## Definitions
- *AV score*: probability that output and the person's held-out texts share an author,
  from an authorship-verification model.
- *Human-likeness score*: perplexity under a reference LM and an AI-detector probability.

## Requirements
### Requirement: Authorship verification
The system SHALL compute an AV score for a candidate output against the person's
held-out set (20% of corpus, never used for exemplars).

### Requirement: Human-likeness
The system SHALL compute perplexity and an AI-detector probability.

### Requirement: Verdict and feedback
The system SHALL return pass/fail against configurable thresholds and a short
natural-language diagnosis (e.g. "sentences 40% shorter than target; no parentheticals")
usable as regeneration feedback.

### Requirement: Calibration
The system SHALL report the AV score distribution of the person's own held-out texts
so thresholds are relative to the person, not absolute.

## Implementation constraints
- AV model: start with the open code/data from "Catch Me If You Can? Not Yet" (EMNLP 2025);
  document it in a reference spec.
- Judge MUST be runnable offline after model download.

## Test requirements
- Unit: thresholds; feedback text generation from feature deltas.
- Integration: the person's own held-out texts pass at ≥ 80%; a generic LLM draft fails.

## Acceptance tests
- See founding doc §6: humanify output passes AV at ≥ the rate of the person's real held-out posts.
```

## A.7 `openspec/specs/cli.md` (draft)

```markdown
# CLI

## Requirements
- `humanify person add <slug> --url ... [--file ...]`
- `humanify person build <slug>`      # ingest + profile
- `humanify rewrite --person <slug> --in <file> --out <file> [--attempts N]`
- `humanify judge --person <slug> --in <file>`
- `humanify bench --person <slug> --in <file> --against spiral.txt`  # compare two outputs
- All commands SHALL have `--json` output.

## Later changes
- MCP server exposing rewrite/judge.
- .pptx and .docx round-tripping.
```

---

# Appendix B — First week

1. `openspec init`; commit `project.md` and the five specs above.
2. Change 1: ingest + profile, with the fixture corpus (a frozen copy of mulix.org).
3. Change 2: judge, calibrated on the fixture corpus (own held-out texts must pass).
4. Change 3: rewrite + iteration loop.
5. Run the §6 acceptance test. Run the Spiral benchmark (§5).
6. Write up what the process taught in `JOURNAL.md`.
