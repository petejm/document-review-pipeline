---
name: review-document
description: Run a multi-agent document review using the Write-Then-Kill pipeline. Every finding cited, every finding contested.
argument: file_path - Path to the document to review
---

# Document Review Pipeline — Write-Then-Kill

Reviewing: `$ARGUMENTS`

!`test -f "$ARGUMENTS" && echo "File found: $(wc -l < "$ARGUMENTS") lines" || echo "ERROR: File not found: $ARGUMENTS"`

---

## Phase 1: Generate Cited Findings

Launch the `finding-writer` agent (Opus) to read the full document and generate findings. Every finding must quote exact text with line numbers.

Use the Task tool to launch the finding-writer agent with:

```
subagent_type: "document-review-pipeline:finding-writer"
prompt: |
  Review this document: $ARGUMENTS

  Read the entire document. Generate findings — factual errors, logical gaps, unsourced claims,
  structural issues. Every finding MUST quote exact text from the document with line numbers.
  Do NOT search the web. Accuracy over volume.
```

Wait for the agent to complete. Parse its output to extract individual findings.

---

## Phase 2: Red Team — Contest Every Finding

For EACH finding from Phase 1, launch a separate `red-team` agent (Sonnet). Run them all in parallel. Each agent gets ONE finding and grep access to the source document.

For each finding, use the Task tool:

```
subagent_type: "document-review-pipeline:red-team"
prompt: |
  Source document: $ARGUMENTS

  Your job is to DISPROVE this finding. Search the source document for evidence
  that contradicts it. Check if the quote is accurate. Check if the document
  addresses this concern elsewhere. Binary verdict only: CONFIRMED or DISPROVED.

  FINDING:
  [paste the individual finding here]
```

Launch ALL red-team agents in a single message using multiple Task tool calls to maximize parallelism.

Collect verdicts. Kill any finding that received a DISPROVED verdict.

---

## Phase 3: Compile Survivors

This phase does NOT need a separate agent — assemble the results directly.

From the findings that received CONFIRMED verdicts in Phase 2, compile the final review:

1. **Brief overall assessment** of the document's strengths (from the finding-writer's summary).
2. **Each surviving finding** with its original quote, line reference, and suggested fix.
3. **Priority ranking** for each finding:
   - **Fix now** — factual error or serious credibility risk
   - **Important** — meaningful issue that should be addressed
   - **Worth doing** — genuine improvement, lower urgency
   - **Minor** — small polish item

Include a summary line: "X findings generated → Y survived red team → Z in final review"

Present the compiled review to the user for inspection before proceeding to Phase 4.

---

## Phase 4: Smoke Test — Verify Every Quote

Launch 2-3 `smoke-test` agents (Sonnet) in parallel. Split the surviving findings into batches. Each agent verifies its batch against the source document.

For each batch, use the Task tool:

```
subagent_type: "document-review-pipeline:smoke-test"
prompt: |
  Source document: $ARGUMENTS

  Verify that every quote and line reference in these findings matches the source
  document exactly. For each finding: VERIFIED or FAILED.

  FINDINGS TO VERIFY:
  [paste the batch of findings here]
```

Launch smoke-test agents in parallel.

---

## Gate Check

After Phase 4 completes:

- If ALL findings are VERIFIED: deliver the final review with confidence.
- If ANY finding is FAILED: flag the specific failures prominently before delivering. Include the smoke test results so the user can see what didn't match.

Present the final review with the smoke test results appended.

---

## Output Structure

```markdown
# Document Review: [document name]

**Pipeline:** Write-Then-Kill | **Findings:** X generated → Y survived → Z verified

## Document Strengths
[From Phase 1 summary]

## Findings

### [Priority] Finding 1: [Title]
**Quote:** "[exact text]" (line N)
**Issue:** [description]
**Suggested Fix:** [recommendation]
**Red Team:** CONFIRMED — [evidence summary]
**Smoke Test:** VERIFIED

[... remaining findings ...]

## Pipeline Metadata
- Phase 1: [N] findings generated (finding-writer, Opus)
- Phase 2: [N] confirmed, [N] disproved (red-team, Sonnet)
- Phase 3: [N] survivors compiled with priority rankings
- Phase 4: [N] verified, [N] failed (smoke-test, Sonnet)
```
