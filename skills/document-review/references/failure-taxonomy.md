# Document Review Failure Taxonomy

Seven categories of failure observed in multi-agent document review, derived from a real-world failed review of a long-form technical document where 24% of findings were wrong.

## Category 1: Ungrounded Claims

**What it looks like:** "The document claims X" with no quote or line reference.

**What's happening:** The agent is asserting what the document says based on its memory of having read it. Memory-based paraphrasing is unreliable — the agent reconstructs what it thinks the document said, often incorrectly.

**Why it happens:** No citation requirement means the agent never has to verify its claims against the actual text. The agent feels confident in its reading even when wrong.

**Pipeline fix:** Mandatory line citations with exact quoted text. If you can't point to the words, the finding doesn't exist.

## Category 2: Keyword-Triggered Corrections

**What it looks like:** Agent sees a name, regulation, or technical term and generates a standard correction based on domain knowledge — without reading the surrounding context.

**What's happening:** The agent "knows" the correct answer from training data and assumes the document got it wrong. It pattern-matches on the keyword and fires a correction without checking whether the document makes the same argument.

**Real example:** Agent saw a specific regulation cited, assumed the document claimed the regulation applies to the subject matter. The document actually argues the opposite — that it does NOT apply — and calls this its main policy finding. The agent attacked the document's strongest analytical insight.

**Pipeline fix:** Red-team agents search the full document for context around each finding. If the document addresses the concern, the finding is DISPROVED.

## Category 3: Web-First Architecture

**What it looks like:** Agent finds something via web search that contradicts its memory of the document. The web "fact" overrides the document text.

**What's happening:** When agents search the web while reviewing, external information displaces their understanding of the actual text. The web becomes ground truth; the document becomes secondary.

**Real example:** Agent found correct date via web search, told author to update the document — document already contained the correct date. Agent compared web result to its (incorrect) memory of the document instead of to the actual text.

**Pipeline fix:** No web searches during review. The review is about what the document says. External fact-checking is a separate, clearly-labeled activity done after the review is complete.

## Category 4: Additive-Only Incentives

**What it looks like:** Review produces many findings, most of which are real — but some are fabricated or exaggerated. More passes = more findings.

**What's happening:** Every agent's objective is to FIND problems. More findings looks like more thorough work. No countervailing force exists to KILL bad findings. Volume is implicitly rewarded.

**Pipeline fix:** Adversarial layer (Phase 2). A separate agent tries to disprove each finding. Findings must survive active attempts to kill them. No incentive to produce volume — neutral framing only.

## Category 5: Consensus Without Evidence

**What it looks like:** Multiple agents agree on a finding. Cross-verification pass confirms it. But the finding is still wrong.

**What's happening:** If Agent A hallucinated "the document says X" and Agent B's domain knowledge priors say "yeah, documents like this usually do say that," the finding survives consensus. Agreement among wrong agents is just confident wrongness.

**Pipeline fix:** Red-team verification is adversarial, not consensus-based. The red-team agent's job is to DISPROVE, not to confirm. And verification checks the finding against the source text, not against other agents' opinions.

## Category 6: Context Window Saturation

**What it looks like:** Later findings are worse than earlier ones. Agent starts missing details it caught in early sections.

**What's happening:** Long documents + accumulated findings + web search results = massive context. Key document details get pushed out of active attention as agents accumulate external information. The more they search, the less they "remember" about the document.

**Pipeline fix:** Single reading pass by a capable model (Opus) for Phase 1. Red-team agents each get only one finding + grep access to the source, keeping scope minimal. No web search to pollute the context window.

## Category 7: Blind Reviewer Anti-Pattern

**What it looks like:** An "independent" reviewer produces findings that sound authoritative but are based entirely on domain knowledge, not document evidence.

**What's happening:** The blind reviewer was designed to provide independent perspective by reviewing without seeing other agents' findings. In practice, it also hasn't thoroughly internalized the document and produces findings from priors — things it "knows" should be issues in this kind of document.

**Real example:** Blind agent knew the general domain rules and a specific regulation's scope. Concluded the document must be wrong to cite the regulation. Never read the section where the document makes that exact argument.

**Pipeline fix:** No blind review. Every agent must ground findings in the document text. Independent perspective comes from the adversarial structure (Phase 2), not from ignorance of the document.

## Quick Diagnostic

| Symptom | Likely Category | First Check |
|---------|----------------|-------------|
| "The document claims X" (no quote) | 1: Ungrounded | Ask for the exact quote |
| Agent corrects something doc handles correctly | 2: Keyword-trigger | Search doc for context around the term |
| Finding sounds authoritative, no line ref | 1 or 7: Ungrounded/Blind | Ask for evidence from the text |
| Web search result contradicts doc | 3: Web-first | Check if doc text actually matches the web result |
| Multiple agents agree, still wrong | 5: Consensus | Verify claim against source text, not other agents |
| Later findings worse than earlier ones | 6: Saturation | Check context window usage |
| More passes made it worse, not better | 4: Additive | Count wrong findings per pass |
