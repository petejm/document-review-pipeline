---
name: review-document
description: Run a Red vs Blue adversarial document review. Three altitude-scoped pairs contest every finding. Every quote verified.
argument: file_path - Path to the document to review
---

# Document Review Pipeline — Red vs Blue (v4)

Reviewing: `$ARGUMENTS`

!`test -f "$ARGUMENTS" && echo "File found: $(wc -l < "$ARGUMENTS") lines" || echo "ERROR: File not found: $ARGUMENTS"`

---

## Phase 1: Three Parallel Red-Blue Pairs

Launch all three Red agents in parallel using the Task tool. Each reads the full document and generates findings at its assigned altitude.

**Launch these three agents simultaneously in a single message with three Task tool calls:**

Red-Textual:

    subagent_type: "document-review-pipeline:red-textual"
    prompt: |
      Review this document for textual precision and mechanical accuracy: $ARGUMENTS
      Read the entire document. Find typos, grammar errors, formatting issues, broken
      cross-references, acronym inconsistencies, citation gaps, unit inconsistencies,
      and conversion artifacts. Every finding must quote exact text with line numbers.

Red-Analytical:

    subagent_type: "document-review-pipeline:red-analytical"
    prompt: |
      Review this document for argument quality and evidence: $ARGUMENTS
      Read the entire document. Find factual claims needing verification, unsourced
      assertions, logical gaps, overstated conclusions, internal contradictions, missing
      caveats, and math errors. Every finding must quote exact text with line numbers.

Red-Strategic:

    subagent_type: "document-review-pipeline:red-strategic"
    prompt: |
      Review this document for strategic effectiveness: $ARGUMENTS
      Read the entire document. Evaluate argument strength vs audience, identify competing
      counterarguments not addressed, assess structural effectiveness and thesis coherence.
      Ground every finding in specific document text.

Wait for all three Red agents to complete. Collect all findings.

---

## Phase 1b: Blue Defense — Contest Every Finding

For EACH finding from ALL three Red agents, launch a separate Blue defender agent. Run ALL Blue defenders in parallel.

For each finding, use the Task tool:

    subagent_type: "document-review-pipeline:blue-defender"
    prompt: |
      Source document: $ARGUMENTS

      Your job is to DISPROVE this finding. Search the source document for evidence
      that contradicts it. Check if the quote is accurate. Check if the document
      addresses this concern elsewhere. Binary verdict: CONFIRMED or DISPROVED.

      FINDING:
      [paste the individual finding here, including its altitude tag]

Launch ALL blue-defender agents in a single message to maximize parallelism.

Collect verdicts. Kill any finding that received a DISPROVED verdict.

---

## Phase 2: Compile Survivors

Assemble findings that survived Blue defense:

1. **Tag each finding with its source altitude** (Textual / Analytical / Strategic).

2. **Deduplicate.** If the same underlying issue was caught by multiple Red agents at different altitudes, merge into a single finding. Note which altitudes flagged it (higher cross-altitude agreement = higher confidence).

3. **Assign priority:**
   - **Fix now** — factual error or serious credibility risk
   - **Important** — meaningful issue that should be addressed
   - **Worth doing** — genuine improvement, lower urgency
   - **Minor** — small polish item

4. **Include the full audit trail per finding:**
   - Red agent's original finding (with quote and line reference)
   - Blue defender's CONFIRMED verdict (with evidence of what it searched and couldn't disprove)

5. **Summary line:** "X findings generated across 3 Red agents → Y survived Blue defense → Z unique findings after deduplication"

Present the compiled review to the user for inspection before proceeding to Phase 3.

---

## Phase 3: Smoke Test — Verify Every Quote

Launch 2-3 smoke-test agents in parallel. Split the surviving findings into batches. Each agent verifies its batch against the source document.

For each batch, use the Task tool:

    subagent_type: "document-review-pipeline:smoke-test"
    prompt: |
      Source document: $ARGUMENTS

      Verify that every quote and line reference in these findings matches the source
      document exactly. For each finding: VERIFIED or FAILED.

      FINDINGS TO VERIFY:
      [paste the batch of findings here]

Launch smoke-test agents in parallel.

---

## Gate Check

After Phase 3 completes:

- If ALL findings are VERIFIED: deliver the final review with confidence.
- If ANY finding is FAILED: flag the specific failures prominently. Include the smoke test results so the user can see what didn't match. Correct or remove failed findings before final delivery.

---

---

## Phase 4: External Verification (Web Enrichment)

Phase 4 is a separate analytical activity that operates on the locked review from Phases 1-3. It does not modify any upstream findings.

### Phase 4a: Factual Claim Extraction

Launch the claim-extractor agent to systematically extract every verifiable factual claim from the source document.

    subagent_type: "document-review-pipeline:claim-extractor"
    prompt: |
      Extract all verifiable factual claims from this document: $ARGUMENTS
      Read the entire document. For each claim, provide: exact quote with line number,
      category, materiality (LOAD-BEARING or BACKGROUND), temporality (STATIC or TEMPORAL),
      and assertion polarity (ASSERTED, ATTRIBUTED, or REFUTED).
      Do NOT evaluate correctness. Do NOT search the web. Extraction only.

Wait for completion. Collect all extracted claims.

---

### Phase 4a-v: Claim Smoke Test

Launch 2-3 smoke-test agents in parallel to verify that extracted claims accurately represent the source document. Split the extracted claims into batches.

For each batch, use the Task tool:

    subagent_type: "document-review-pipeline:smoke-test"
    prompt: |
      Source document: $ARGUMENTS

      Verify that every quoted claim below accurately represents the source document.
      For each claim: check that the quote is exact, the line reference is correct,
      and the extracted claim accurately represents what the document says (no added
      precision, no dropped qualifiers, correct assertion polarity).

      For each claim: VERIFIED or FAILED with detail.

      CLAIMS TO VERIFY:
      [paste the batch of extracted claims here]

Launch smoke-test agents in parallel.

Drop any FAILED claims. Only VERIFIED claims proceed to Phase 4b.

---

### Phase 4b: Web Verification (LOAD-BEARING claims only)

From the VERIFIED claims, select only those tagged LOAD-BEARING. Batch them into groups of 5-10 and launch web-verifier agents in parallel.

For each batch, use the Task tool:

    subagent_type: "document-review-pipeline:web-verifier"
    prompt: |
      Source document: $ARGUMENTS

      Verify these LOAD-BEARING factual claims against authoritative web sources.
      For each claim: CONFIRMED, CONTRADICTED, DISPUTED, or UNVERIFIABLE.
      Include source URL, evidence quote, temporal note, circular confirmation check,
      and source authority tier.

      Cannot modify upstream review findings. Annotation only.

      CLAIMS TO VERIFY:
      [paste the batch of LOAD-BEARING claims here]

Launch web-verifier agents in parallel.

---

### Phase 4c: Cross-Reference Table

The orchestrator (no new agent) produces a cross-reference comparing Phase 4b web results against Phase 1-3 confirmed findings:

| Claim | Document Says | Web Says | Verdict | Red Caught? | Finding # |
|---|---|---|---|---|---|
| [claim] | [quote] | [web evidence] | CONFIRMED/CONTRADICTED/DISPUTED/UNVERIFIABLE | Yes/No | [#] or — |

Compute exploratory metrics (no action threshold defined):
- **Red catch rate:** % of CONTRADICTED claims that Red already flagged
- **Web-only discoveries:** CONTRADICTED claims Red missed
- **Temporal drift count:** Claims that were once-correct but are now outdated
- **Circular confirmation count:** Claims CONFIRMED only by the same source the document cites

---

## Output Structure

    # Document Review: [document name]

    **Pipeline:** Red vs Blue v4 | **Findings:** X generated → Y survived Blue → Z verified

    ## Document Strengths
    [From Red-Strategic agent's assessment]

    ## Findings

    ### [Priority] Finding 1: [Title]
    **Altitude:** Textual | Analytical | Strategic
    **Quote:** "[exact text]" (line N)
    **Issue:** [description]
    **Suggested Fix:** [recommendation]
    **Blue Defense:** CONFIRMED — [evidence summary]
    **Smoke Test:** VERIFIED

    [... remaining findings ...]

    ## Audit Trail

    ### Red Team Output
    - Red-Textual: [N] findings generated
    - Red-Analytical: [N] findings generated
    - Red-Strategic: [N] findings generated
    - Total: [N] findings

    ### Blue Defense Results
    - CONFIRMED: [N] findings survived
    - DISPROVED: [N] findings killed
    - Kill rate: [N]%

    ### Cross-Altitude Agreement
    [Table showing findings flagged by multiple Red agents]

    ### Smoke Test Results
    - VERIFIED: [N]
    - FAILED: [N]

    ## External Verification Appendix

    > **Reading Order:** Read the review findings (above) and form your own assessment
    > BEFORE reading this appendix. The review is grounded solely in what your document
    > says. This appendix contains external verification that may confirm, contradict,
    > or complicate claims — but it is a separate analytical activity with different
    > evidence standards and different failure modes.

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

    ## Pipeline Metadata
    - Phase 1 Red: 3 agents (Opus), parallel execution
    - Phase 1b Blue: [N] agents (Sonnet), 1 per finding, parallel execution
    - Phase 2: Compilation and deduplication
    - Phase 3: [N] smoke-test agents (Sonnet), parallel execution
    - Phase 4a: 1 claim-extractor agent (Opus)
    - Phase 4a-v: [N] smoke-test agents (Sonnet), parallel execution
    - Phase 4b: [N] web-verifier agents (Sonnet), parallel execution
    - Phase 4c: Orchestrator cross-reference
    - Total tokens: [N]
