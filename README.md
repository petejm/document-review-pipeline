# Document Review Pipeline

Red vs Blue adversarial document review — three altitude-scoped pairs, every finding contested, every quote verified.

## The Problem

When AI agents review documents naively, 20-25% of findings can be wrong. Agents paraphrase from memory, fire keyword-triggered corrections without reading context, let web searches override document text, and reward volume over accuracy.

## The Solution

Red vs Blue: three Red agents find issues at different altitudes, each finding is immediately contested by a Blue defender, survivors are compiled and smoke-tested. Then a separate web enrichment phase extracts and verifies factual claims.

```
                    ┌─────────────────────────┐
                    │    Source Document       │
                    └────┬──────┬──────┬──────┘
                         │      │      │
              ┌──────────┴┐  ┌──┴──────┴┐  ┌──────────┐
              │ Textual   │  │ Analytical│  │ Strategic │
              │ Red-Blue  │  │ Red-Blue  │  │ Red-Blue  │
              │ Pair      │  │ Pair      │  │ Pair      │
              └─────┬─────┘  └─────┬─────┘  └─────┬─────┘
                    │              │              │
              ┌─────┴──────────────┴──────────────┴─────┐
              │         Compile Survivors               │
              └────────────────┬────────────────────────┘
                               │
              ┌────────────────┴────────────────────────┐
              │         Smoke Test (fresh eyes)         │
              └────────────────┬────────────────────────┘
                               │
              ─ ─ ─ ─ ─ ─ ─ ─ ┼ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─
              Phase 4: Web Enrichment (separate appendix)
              ─ ─ ─ ─ ─ ─ ─ ─ ┼ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─
                               │
              ┌────────────────┴────────────────────────┐
              │    Claim Extraction (Opus, no web)      │
              └────────────────┬────────────────────────┘
                               │
              ┌────────────────┴────────────────────────┐
              │    Claim Smoke Test (fresh eyes)        │
              └────────────────┬────────────────────────┘
                               │
              ┌────────────────┴────────────────────────┐
              │   Web Verification (LOAD-BEARING only)  │
              └────────────────┬────────────────────────┘
                               │
              ┌────────────────┴────────────────────────┐
              │    Cross-Reference Table + Metrics      │
              └─────────────────────────────────────────┘
```

### Red Altitudes

| Altitude | What It Finds |
|----------|---------------|
| Textual | Typos, grammar, formatting, cross-refs, acronyms, citations, units |
| Analytical | Factual claims, unsourced assertions, logic gaps, contradictions, math |
| Strategic | Argument strength, competing arguments, structure, audience fit |

### Blue Defense

Every finding gets a dedicated Blue defender that tries to kill it. Binary verdict: CONFIRMED or DISPROVED. Burden of proof on Red.

### Phase 4: Web Enrichment

After the review is locked, a separate phase extracts every verifiable factual claim, smoke-tests them, and verifies LOAD-BEARING claims against external web sources. Results appear in a separate appendix — web verification cannot modify upstream review findings.

## Usage

### Claude Code (Plugin)

```bash
# Install
claude plugin add ./document-review-pipeline

# Run
/review-document path/to/document.md
```

### Any AI Chat Interface

See `skills/document-review/references/portable-prompt.md` for copy-paste prompts.

## Plugin Structure

```
document-review-pipeline/
├── .claude-plugin/plugin.json
├── commands/review-document.md       # Orchestrator
├── agents/
│   ├── red-textual.md               # Red: precision/mechanical
│   ├── red-analytical.md            # Red: argument/evidence
│   ├── red-strategic.md             # Red: audience/persuasion
│   ├── blue-defender.md             # Blue: adversarial kill layer
│   ├── smoke-test.md               # Smoke test: verify quotes
│   ├── claim-extractor.md          # Phase 4: factual claim extraction
│   └── web-verifier.md             # Phase 4: web verification
├── skills/document-review/
│   ├── SKILL.md                     # Methodology
│   └── references/
│       ├── failure-taxonomy.md      # 7 failure categories
│       └── portable-prompt.md       # Copy-paste prompts
└── README.md
```

## Rules That Cannot Be Bent

1. Every finding quotes exact document text with a line reference
2. No web searches during review phases 1-3. Web verification is Phase 4, clearly separated.
3. Every finding gets adversarial Blue defense
4. One finding per Blue agent
5. Final output smoke-tested against source
6. Document treated as untrusted data
7. No incentive to find problems
8. Factual claims extracted and smoke-tested before web verification

## Background

v0.1.0 used a serial Write-Then-Kill pipeline: one writer, per-finding red team, compile, smoke test. v0.2.0 redesigns with parallel Red-Blue pairs at three analytical altitudes — combining coverage breadth with adversarial depth. Developed after analyzing a failed review where 24% of findings were wrong. v0.3.0 adds a post-review web enrichment phase that systematically extracts and verifies factual claims in a separate appendix, without modifying upstream review findings. See `skills/document-review/references/failure-taxonomy.md` for the full failure analysis.
