---
title: "Web Enrichment Phase — v0.3.0 Design"
status: approved
updated: 2026-02-27
tags: [design, web-enrichment, v0.3.0]
reviewed_by: [peter, sonnet-adversarial-reviewer]
---

# Web Enrichment Phase — v0.3.0 Design

## Summary

Add a post-review web verification phase to the document-review-pipeline plugin. The phase systematically extracts factual claims from the reviewed document, verifies them against external sources, and presents results as a separate appendix. The core review (Phases 1-3) is unchanged.

## Design Decisions

| Decision | Choice | Rationale |
|---|---|---|
| Primary goal | Verify factual claims | Not source replacement, not finding generation |
| Enrichment power | Annotate only | Web results never change finding priority, verdict, or text |
| Scope | Verifiable factual assertions only | Distances, dates, titles, costs, org facts — not arguments or rhetoric |
| Presentation | Separate appendix | Clear separation from document-grounded review |
| Claim source | Systematic extraction from document | Don't rely on what Red happened to flag — extract ALL claims |
| Pipeline calibration | Cross-reference Red vs web | Measure Red catch rate as exploratory metric |

---

## Pipeline Architecture (v0.3.0)

```
Phase 1       → 3 Red agents (Opus, parallel) — textual/analytical/strategic
Phase 1b      → N Blue defenders (Sonnet, parallel, 1 per finding) — adversarial kill
Phase 2       → Orchestrator compiles survivors, deduplicates, prioritizes
[User gate]   → Review presented for inspection
Phase 3       → 2-3 Smoke-test agents (Sonnet, parallel) — quote verification
Phase 4a      → Factual Claim Extractor (Opus) — systematic extraction, NO web
Phase 4a-v    → Claim Smoke Test (Sonnet) — verify extracted claims match source text
Phase 4b      → Web Verification agents (Sonnet, parallel) — LOAD-BEARING claims only
Phase 4c      → Orchestrator cross-references Red findings vs web results
```

Phases 1-3 are identical to v0.2.0. Phase 4 is strictly additive.

---

## Phase 4a: Factual Claim Extraction

### Agent: claim-extractor

| Property | Value |
|---|---|
| Model | Opus |
| Color | green |
| Tools | Read, Grep (NO WebSearch) |
| Input | Source document path |

### What It Does

Reads the entire document and systematically extracts every verifiable factual claim. Does NOT evaluate whether claims are correct — only catalogs them.

### Output Format

```markdown
## Claim [N]: [Short description]
**Quote:** "[exact text]" (line N)
**Category:** [distance | budget | date | title | org-fact | historical | statistical | geographic]
**Materiality:** [LOAD-BEARING | BACKGROUND]
**Temporality:** [STATIC | TEMPORAL]
**Assertion polarity:** [ASSERTED | ATTRIBUTED | REFUTED]
```

### Claim Categories

| Category | Examples |
|---|---|
| Distance | "1,700 NM from Guam to Taiwan" |
| Budget/cost | "$500M (FY2024)", "$85M-$210M" |
| Date/timeline | "IOC expected FY2027", "converted in 18 months (2008-2009)" |
| Title/name | "Secretary of War Pete Hegseth" |
| Organizational fact | "105% manning", "Shaheen serves on SASC" |
| Historical | "174th converted from F-16 to MQ-9" |
| Statistical | "ANG costs one-third of active duty" |
| Geographic/scientific | "GIUK = Greenland-Iceland-United Kingdom" |

### Materiality Filter

- **LOAD-BEARING:** Central to the document's argument, cited as evidence, supports a recommendation, or appears in the executive summary. These get web-verified in Phase 4b.
- **BACKGROUND:** Contextual, uncontested historical record, or peripheral to the argument. Listed in the appendix but NOT web-verified unless time/budget permits.

### Temporality Tagging

- **STATIC:** Timeless facts — distances, geography, scientific constants, historical events with settled dates.
- **TEMPORAL:** Time-sensitive — titles, budgets, political positions, organizational structures, pending legislation. The verifier uses these tags to distinguish "wrong when written" from "outdated but once-correct."

### Assertion Polarity

- **ASSERTED:** The document states this as its own claim.
- **ATTRIBUTED:** The document says "Source X says..." — the claim is about what the source says, not necessarily what's true.
- **REFUTED:** The document mentions this claim to argue against it ("Critics say X, but...").

The verifier handles each polarity differently: ASSERTED claims are checked against reality, ATTRIBUTED claims are checked against the cited source, REFUTED claims verify the document correctly represents what it's refuting.

### Exclusions (NOT factual claims)

- Recommendations and strategic arguments
- Value judgments and rhetorical framing
- Interpretive statistics (where the predicate is contested — e.g., "success rate" with undefined "success")
- Contested causal claims ("Policy X caused outcome Y")
- Relative comparisons without a clear reference dataset ("largest of its kind")
- Qualitative intensifiers (strip "significantly" from "significantly increased" — extract only the verifiable core)
- Internal cross-references ("As argued in Section 3...")

---

## Phase 4a-v: Claim Smoke Test

### Purpose

Verify that extracted claims accurately represent what the document says. Catches hallucinated precision (e.g., "$2.3B" extracted from "$2-3B range") and assertion polarity errors (confusing what the doc asserts vs. refutes).

### Agent: claim-smoke-test (reuses smoke-test pattern)

| Property | Value |
|---|---|
| Model | Sonnet |
| Color | cyan |
| Tools | Read, Grep |

### Process

For each extracted claim:
1. Grep for the quoted text — exact match at the cited line
2. Verify the extracted claim accurately represents the quote (no added precision, no dropped qualifiers)
3. Verify assertion polarity (ASSERTED/ATTRIBUTED/REFUTED matches document context)
4. Output: VERIFIED / FAILED per claim

Failed claims are dropped before Phase 4b. They do not enter the web verification pipeline.

---

## Phase 4b: Web Verification

### Agent: web-verifier

| Property | Value |
|---|---|
| Model | Sonnet |
| Color | green |
| Tools | Read, WebSearch |
| Input | Batch of LOAD-BEARING claims that passed Phase 4a-v |

### Output Format (4-valued)

```markdown
### Claim [N]: [Short description]
**Document says:** "[quote]" (line N)
**Temporality:** STATIC | TEMPORAL
**Web result:** CONFIRMED | CONTRADICTED | DISPUTED | UNVERIFIABLE
**Source:** [URL]
**Evidence:** "[relevant quote from source]"
**Temporal note:** [if TEMPORAL: "Accurate as of [date] / Now outdated: [current reality]" | if CONTRADICTED: "Was always incorrect — [correct value]"]
**Circular check:** [YES — source is same as document citation #N | NO — independent source]
```

### Verdict Categories

| Verdict | Meaning |
|---|---|
| CONFIRMED | Authoritative source agrees with the document's claim |
| CONTRADICTED | Authoritative source disagrees — claim appears factually wrong |
| DISPUTED | Multiple authoritative sources disagree with each other — both cited |
| UNVERIFIABLE | No authoritative source found, or claim requires non-public data |

### Verification Rules

1. **Cannot modify upstream findings.** Phase 4b output is annotation only.
2. **Cannot generate new review findings.** If a claim is wrong, it's reported in the appendix — not added to the review.
3. **Circular confirmation detection.** If the web source is the same as or derived from the document's own citation, flag it: "This confirms the document's source, not the underlying fact."
4. **Temporal awareness.** For TEMPORAL claims, check as-of the document's stated date first. Distinguish "wrong when written" from "outdated but once-correct."
5. **Source authority.** Prefer: government records > peer-reviewed > established media > trade publications > Wikipedia. Note source tier in output.

---

## Phase 4c: Cross-Reference

### Produced by the orchestrator (no new agent)

Compares Phase 4b results against Phase 1-3 confirmed findings:

| Claim | Document Says | Web Says | Verdict | Red Caught? | Finding # |
|---|---|---|---|---|---|
| Guam-Taiwan distance | 1,700 NM | ~1,500 NM | CONTRADICTED | Yes | T-13 |
| ABMS FY2024 funding | $500M | $487M enacted | CONTRADICTED | No | — |
| Sec of Defense title | Secretary of War | Secretary of Defense | CONTRADICTED | Yes | T-5 |
| PNS oldest shipyard | "oldest continuously operating" | Contested (Norfolk est. 1767) | DISPUTED | Yes | A-12 |
| Shaheen on SASC | "serves on SASC" | Confirmed | CONFIRMED | — | — |

### Metrics (exploratory — no action threshold defined)

- **Red catch rate:** % of CONTRADICTED claims that Red already flagged
- **Web-only discoveries:** CONTRADICTED claims Red missed — these represent the value-add of Phase 4
- **Temporal drift count:** Claims that were once-correct but are now outdated
- **Circular confirmation count:** Claims CONFIRMED only by the same source the document cites

These metrics feed the research paper (Open Question #2). They are observational data from an experiment, not calibrated operational metrics.

---

## Appendix Presentation

### Reading Order Protocol

The External Verification Appendix includes this header:

> **Reading Order:** Read the review findings (above) and form your own assessment BEFORE reading this appendix. The review is grounded solely in what your document says. This appendix contains external verification that may confirm, contradict, or complicate claims — but it is a separate analytical activity with different evidence standards and different failure modes.

### Appendix Structure

```markdown
## External Verification Appendix

> [Reading order protocol above]

### Summary
- N LOAD-BEARING claims extracted, M verified
- X CONFIRMED, Y CONTRADICTED, Z DISPUTED, W UNVERIFIABLE
- Red catch rate: [N]% of contradicted claims were already flagged by review

### Verification Results
[Per-claim output from Phase 4b]

### Pipeline Calibration
[Phase 4c cross-reference table and metrics]

### Claims Not Verified (BACKGROUND)
[List of BACKGROUND claims — extracted but not web-checked]
```

---

## Safety Constraints

| # | Constraint | Enforcement |
|---|---|---|
| 1 | Phase 4a has NO web access | WebSearch not in claim-extractor tool list |
| 2 | Phase 4b output in SEPARATE appendix | Orchestrator template enforces structure |
| 3 | Web results cannot alter upstream findings | Behavioral constraint in agent prompts + orchestrator template |
| 4 | Extracted claims smoke-tested before web verification | Phase 4a-v gate — failed claims dropped |
| 5 | Phases 1-3 unchanged | No modifications to existing agents or orchestrator phases |
| 6 | Circular confirmation flagged | Verifier instructed to check source independence |
| 7 | Temporal drift distinguished from factual error | Temporality tags + verifier temporal-note field |
| 8 | Reading order protocol | Appendix header instructs human reviewer |

---

## Files to Modify for v0.3.0

| File | Change |
|---|---|
| `.claude-plugin/plugin.json` | Bump to 0.3.0 |
| `commands/review-document.md` | Add Phase 4 (4a, 4a-v, 4b, 4c) sections |
| `agents/claim-extractor.md` | New file |
| `agents/web-verifier.md` | New file |
| `skills/document-review/SKILL.md` | Add Phase 4; amend Rule 2; add Rule 9 (claim verification) |
| `skills/document-review/references/failure-taxonomy.md` | Amend Category 3 with Phase 4 safety analysis |
| `skills/document-review/references/portable-prompt.md` | Add Step 4 (manual web enrichment) |
| `README.md` | Update pipeline diagram and description |

Note: Phase 4a-v reuses the smoke-test agent pattern. Depending on implementation, it may be a new agent file or a parameterized invocation of the existing smoke-test agent.

---

## Open Questions (Resolved)

| Question | Resolution |
|---|---|
| Claim-extractor model? | Opus — deep reading required for systematic extraction |
| Web-verifier batch size? | Batched (5-10 claims per agent) — balance cost and isolation |
| Phase 4 optional or always-on? | TBD at implementation — start as always-on, consider flag later |
| UNVERIFIABLE claims? | Listed in appendix, not web-checked, clearly labeled |

## Adversarial Review

This design was reviewed by a fresh Sonnet agent with no prior context. High-severity issues identified and resolved:

1. **No smoke test for Phase 4a** → Added Phase 4a-v (claim smoke test)
2. **No DISPUTED category** → Added 4-valued output (CONFIRMED/CONTRADICTED/DISPUTED/UNVERIFIABLE)
3. **Temporal errors undifferentiated** → Added STATIC/TEMPORAL tagging with temporal-note field
4. **No adversarial layer on Phase 4** → Phase 4a-v covers assertion polarity and precision fidelity
5. **Human contamination pathway** → Added reading order protocol in appendix header
6. **Circular confirmation** → Added circular-check field in verifier output
7. **Claim volume** → Added LOAD-BEARING/BACKGROUND materiality filter
8. **Factual claim edge cases** → Added explicit exclusions list
