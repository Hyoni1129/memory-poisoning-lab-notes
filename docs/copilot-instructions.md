# Copilot Instructions For This Workspace

This workspace uses a dual-repository research workflow:
- memory-poisoning-research: formal, reproducible artifacts
- memory-poisoning-lab-notes: process notes, logs, and decisions

## Always Follow These Local Guides
Before deciding git actions or organization steps, consult:
- .github/research-workflow/AGENT_GUIDE.md
- .github/research-workflow/AGENT_WORKFLOW.md
- .github/research-workflow/SKILL_PROPOSAL_GUIDE.md
- .github/research-workflow/QUICK_COMMANDS.md

## Operating Defaults
1. Analyze changes in both repositories first.
2. Read changed files and infer intent before committing.
3. Validate placement and naming conventions.
4. Propose logical commit groups instead of one giant commit.
5. Ask for confirmation when intent is ambiguous or risky.
6. Never push unless explicitly requested.

## Commit Prefix Policy
For memory-poisoning-research use:
- exp, feat, analysis, fix, docs

For memory-poisoning-lab-notes use:
- log, decision, reading, exp-note, failure, meeting, idea

## File Placement Policy
- Keep formal artifacts in memory-poisoning-research.
- Keep process narratives in memory-poisoning-lab-notes.
- Preserve cross-repository traceability via shared experiment IDs.

## Root Intake Default (Deep Research Reports)
When the user asks to organize root-level files and a deep-research style report is detected:
1. Move the file to memory-poisoning-lab-notes/04_deep_research_reports/.
2. Normalize name to report-YYYY-MM-DD-topic-slug.md.
3. Keep content unchanged unless explicitly asked to edit.
4. Prefer commit prefix reading: for this report flow.
5. If the report is clearly experiment commentary (strong exp_id focus + run interpretation), allow exp-note: instead.

## Safety
- Do not delete files without confirmation.
- Do not move files aggressively if intent is unclear.
- Do not fabricate commit meaning.
- Avoid noisy micro-commits.
