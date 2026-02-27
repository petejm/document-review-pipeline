---
title: "Web Enrichment Phase Design Notes — Working Draft"
status: active
updated: 2026-02-27
tags: [design, brainstorming, web-enrichment, v0.3.0]
---

# Web Enrichment Phase — Design Notes

## Design Decisions (confirmed with Peter)

1. **Primary goal:** Verify factual claims the document makes
2. **Enrichment power:** Annotate only — web results never change finding priority, verdict, or text
3. **Scope:** Only findings/claims with verifiable factual assertions (distances, dates, titles, costs, org facts)
4. **Presentation:** Separate appendix ("External Verification Appendix") — not inline
5. **Extraction step required:** Don't just enrich Red's confirmed findings — systematically extract ALL factual claims from the document first, then verify them ALL
6. **Pipeline calibration:** Cross-reference Red findings vs web results to measure Red catch rate

## Revised Pipeline Architecture (v0.3.0)

```
Phase 1     → 3 Red agents (Opus, parallel) — textual/analytical/strategic
Phase 1b    → N Blue defenders (Sonnet, parallel, 1 per finding) — adversarial kill
Phase 2     → Orchestrator compiles survivors, deduplicates, prioritizes
[User gate] → Review presented for inspection
Phase 3     → 2-3 Smoke-test agents (Sonnet, parallel) — quote verification
Phase 4a    → Factual Claim Extractor (Opus) — systematic extraction, NO web
Phase 4b    → Web Verification agents (Sonnet, parallel) — targeted searches
Phase 4c    → Orchestrator cross-references Red findings vs web results
```

## New Agents

### claim-extractor (Phase 4a)
- **Model:** Opus (deep reading required for systematic extraction)
- **Color:** green (neutral — not red/blue)
- **Tools:** Read, Grep (NO WebSearch)
- **Input:** Source document path
- **Output:** Structured list of all verifiable factual claims with line references
- **Claim categories:** distances, budget/cost figures, dates/timelines, titles/names, organizational facts, historical claims, statistical claims, geographic/scientific facts
- **Rules:** Document-only. No web. No evaluation of whether claims are correct — just extraction.

### web-verifier (Phase 4b)
- **Model:** Sonnet (mechanical fact-checking)
- **Color:** green
- **Tools:** Read, WebSearch
- **Input:** Individual factual claim (or batch) from Phase 4a output
- **Output per claim:** CONFIRMED / CONTRADICTED / UNVERIFIABLE + source URL + evidence quote
- **Rules:** Cannot modify upstream findings. Cannot generate new review findings. Output is annotation only.

## Phase 4c: Cross-Reference (Orchestrator)

Produces a comparison table:

| Claim | Document Says | Web Says | Verdict | Red Caught? | Finding # |
|---|---|---|---|---|---|
| Guam-Taiwan distance | 1,700 NM | ~1,500 NM | CONTRADICTED | Yes | T-13 |
| ABMS FY2024 funding | $500M | $487M (enacted) | CONTRADICTED | No | — |
| Sec of Defense title | Secretary of War | Secretary of Defense | CONTRADICTED | Yes | T-5 |

Metrics produced:
- **Red catch rate:** % of web-contradicted claims that Red already flagged
- **Web-only discoveries:** Claims the pipeline would have missed without Phase 4
- **False positive rate:** Web results that contradict claims but are themselves wrong/outdated

## Definition: Factual Claim

A statement in the document that:
1. Asserts something about the external world (not about the document's own argument)
2. Can be checked against an authoritative external source
3. Has a truth value independent of the document's rhetorical purpose

**IS a factual claim:** distances, budget figures, dates, titles, organizational facts, historical events, statistics
**NOT a factual claim:** recommendations, strategic arguments, value judgments, rhetorical framing

## Key Safety Constraints

1. Phase 4a has NO web access — claims come FROM the document only
2. Phase 4b output is a SEPARATE appendix — never merged into review findings
3. Web results cannot alter, kill, or elevate existing findings
4. The worst case (bad web result → false CONTRADICTED) is in a labeled appendix with source URL — reader can verify
5. Phase 4 is strictly additive to the v0.2.0 pipeline — Phases 1-3 unchanged

## Files to Modify for v0.3.0

| File | Change |
|---|---|
| `.claude-plugin/plugin.json` | Bump to 0.3.0 |
| `commands/review-document.md` | Add Phase 4 section |
| `agents/claim-extractor.md` | New file |
| `agents/web-verifier.md` | New file |
| `skills/document-review/SKILL.md` | Add Phase 4; amend Rule 2 |
| `skills/document-review/references/failure-taxonomy.md` | Amend Category 3 |
| `skills/document-review/references/portable-prompt.md` | Add Step 4 |
| `README.md` | Update pipeline diagram |

## Open Questions

1. Should claim-extractor be Opus or Sonnet? Opus is better at deep reading but more expensive. Could Sonnet handle systematic extraction on an 87-page doc?
2. Batch size for web-verifier: one per claim (cleanest) or batches of 5-10 (cheaper)?
3. Should Phase 4 be optional (flag on /review-document) or always-on?
4. How to handle UNVERIFIABLE claims — some things just can't be web-checked (internal ANG data, classified info). Do we report these or skip them?

## v1 Failure Mode Comparison

- **v1 (failed):** Web search → generate findings → compare to document memory → wrong findings
- **v5 (proposed):** Document read → extract claims → web search → annotate discrepancies → separate appendix

Critical difference: the claim list comes FROM the document, not from the web.
