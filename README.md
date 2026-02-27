# Document Review Pipeline

Multi-agent document review using Write-Then-Kill verification — every finding cited, every finding contested.

## The Problem

When AI agents review documents naively, 20-25% of findings can be wrong. Agents paraphrase from memory instead of quoting, fire keyword-triggered corrections without reading context, let web searches override document text, and reward volume over accuracy. Multiple agents agreeing doesn't help — consensus among wrong agents is just confident wrongness.

## The Solution

Write-Then-Kill: generate grounded findings, then actively try to kill each one.

| Phase | Agent | Job | Constraint |
|-------|-------|-----|------------|
| 1. Write | Opus | Generate cited findings | Must quote exact text; no web search |
| 2. Kill | Sonnet (1 per finding) | Try to disprove each finding | Binary verdict; one finding per agent |
| 3. Compile | Orchestrator | Assemble survivors | No new findings |
| 4. Smoke test | Sonnet (2-3) | Verify quotes match source | Fresh eyes only |

## Usage

### Claude Code (Plugin)

```bash
# Install
claude --plugin-dir ./document-review-pipeline

# Run
/review-document path/to/document.md
```

The command orchestrates all four phases automatically.

### Any AI Chat Interface

See `skills/document-review/references/portable-prompt.md` for a copy-paste version that works in Claude.ai, ChatGPT, or any chat interface. No tooling required.

## Plugin Structure

```
document-review-pipeline/
├── .claude-plugin/
│   └── plugin.json                     # Plugin manifest
├── commands/
│   └── review-document.md              # /review-document <file-path>
├── agents/
│   ├── finding-writer.md               # Phase 1: Generate cited findings
│   ├── red-team.md                     # Phase 2: Try to disprove each finding
│   └── smoke-test.md                   # Phase 4: Verify quotes match source
├── skills/
│   └── document-review/
│       ├── SKILL.md                    # Methodology (auto-loaded on trigger)
│       └── references/
│           ├── failure-taxonomy.md     # 7 failure categories and why they happen
│           └── portable-prompt.md      # Copy-paste prompt for non-Claude-Code users
└── README.md
```

## Rules That Cannot Be Bent

1. Every finding quotes exact document text with a line reference
2. No web searches during review
3. Every finding gets adversarial testing
4. One finding per red-team agent
5. Final output gets smoke-tested against source
6. Document is treated as untrusted data
7. No incentive to find problems

## Background

Developed after a failed 8-pass review of a long-form technical document where 24% of findings were wrong — agents hallucinated document contents, told the author to fix things already handled correctly, and attacked the paper's strongest analytical insight. The redesigned pipeline produced a clean review with zero verification failures. See `skills/document-review/references/failure-taxonomy.md` for the full failure analysis.
