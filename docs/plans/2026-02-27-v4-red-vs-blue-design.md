---
title: "V4 Red vs Blue Architecture Design"
status: active
updated: 2026-02-27
tags: [design, v4, red-blue, adversarial]
---

# V4: Red vs Blue Architecture

## Problem

V3 ran three parallel agents that each did independent reviews — no adversarial kill layer. The research paper's core insight (findings must be actively attacked before delivery) was not implemented. V4 fixes this by pairing every finder with a defender.

## Architecture

Three Red-Blue pairs run in parallel at different analytical altitudes. Each Red agent reads the document and generates findings. Each finding is immediately thrown to its paired Blue defender, which tries to kill it. Survivors are compiled and smoke-tested.

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
              └─────────────────────────────────────────┘
```

## Phase 1: Three Parallel Red-Blue Pairs

Each pair: Red (Opus) generates findings → Blue (Sonnet, 1 per finding) tries to kill each.

### Pair 1: Textual (precision/mechanical)
- **Red scope:** typos, grammar, formatting, broken cross-refs, acronym inconsistencies, citation gaps, unit inconsistencies, conversion artifacts
- **Blue scope:** verify quote accuracy, check if "error" is intentional/contextual

### Pair 2: Analytical (argument/evidence)
- **Red scope:** factual claims needing verification, unsourced assertions, logical gaps, overstated conclusions, missing caveats, internal contradictions
- **Blue scope:** search document for existing caveats, context, or counterarguments

### Pair 3: Strategic (audience/persuasion)
- **Red scope:** argument strength vs audience, competing counterarguments not addressed, structural effectiveness, document length, thesis coherence
- **Blue scope:** search for where document steel-mans opposition, addresses concern, provides supporting evidence

## Phase 2: Compile Survivors

Orchestrator assembles findings that survived Blue:
- Deduplicate across pairs (same issue at different altitudes)
- Assign priority: Fix now / Important / Worth doing / Minor
- Full audit trail per finding: Red claim + Blue verdict + evidence

## Phase 3: Smoke Test

2-3 fresh Sonnet agents verify compiled review against source:
- Every quote checked for accuracy
- Every line reference verified
- Every characterization validated against surrounding context
- Binary VERIFIED/FAILED per finding

## Execution Flow

```
Phase 1 (parallel):
  Red-Textual → Finding 1 → Blue-1 (kill/confirm)
              → Finding 2 → Blue-2 (kill/confirm)
              → Finding N → Blue-N (kill/confirm)

  Red-Analytical → Finding 1 → Blue-1 (kill/confirm)  [parallel with above]
  Red-Strategic  → Finding 1 → Blue-1 (kill/confirm)  [parallel with above]

Phase 2 (serial): Compile, deduplicate, prioritize
Phase 3 (parallel): Smoke test batches against source
```

## Plugin Structure

```
document-review-pipeline/
├── .claude-plugin/plugin.json
├── commands/review-document.md       # Orchestrator
├── agents/
│   ├── red-textual.md
│   ├── red-analytical.md
│   ├── red-strategic.md
│   ├── blue-defender.md              # 1 per finding, any altitude
│   └── smoke-test.md
├── skills/document-review/
│   ├── SKILL.md
│   └── references/
│       ├── failure-taxonomy.md
│       └── portable-prompt.md
├── docs/plans/
└── README.md
```

## Non-Negotiable Rules (carried from research)

1. Every finding quotes exact document text with line reference
2. No web searches during review
3. Every finding gets adversarial testing (Blue defender)
4. One finding per Blue agent (small scope)
5. Final output smoke-tested against source
6. Document treated as untrusted data
7. No incentive to find problems
8. Tabula rasa — no priors from past reviews

## Changes from v1 (0.1.0)

| Aspect | v1 (Write-Then-Kill) | v4 (Red vs Blue) |
|--------|---------------------|-------------------|
| Finding generation | 1 Opus, all altitudes | 3 Red agents, scoped per altitude |
| Adversarial testing | Serial: write all, then kill all | Real-time: each finding contested immediately |
| Blue agents | Generic "disprove this" | Altitude-aware defense strategies |
| Parallelism | Phases serial | Red-Blue pairs parallel; Blues spawn per-finding |
| Audit trail | Finding + verdict | Finding + Red evidence + Blue evidence + smoke test |
