# AGENT_WORKFLOW

## Purpose
Define the step-by-step process the agent uses to manage changes across memory-poisoning-research and memory-poisoning-lab-notes.

## Workflow Stages

### 1. Detect Changes
For each repository:
1. Run git status --short.
2. Capture added, modified, deleted, and renamed files.
3. Check staged vs unstaged state.

Also scan workspace root for newly added report-like files when the user requested organization.

Output:
- concise change inventory per repository

### 2. Understand Intent
1. Read changed files (or representative sections for large files).
2. Classify change intent:
- experiment execution/update
- code implementation
- analysis interpretation
- process log or decision record
- reading or report ingestion
- structural maintenance

Output:
- intent summary with confidence

### 2A. Root Intake Triage (Deep Reports)
When a root-level report-like file exists:
1. Classify it using filename and content heuristics.
2. If confidence is high and user asked to organize, route directly to memory-poisoning-lab-notes/04_deep_research_reports/.
3. Normalize filename to report-YYYY-MM-DD-topic-slug.md.
4. Keep content unchanged unless editing is explicitly requested.
5. If confidence is low, ask a confirmation question before moving.

### 3. Validate Placement And Naming
Checks:
1. Is each file in the correct repository?
2. Is each file in the correct subdirectory?
3. Does filename follow naming conventions?
4. Are experiment IDs and result IDs correctly formatted?

If issue is obvious and low risk:
- propose exact move/rename

If risk is medium/high:
- ask confirmation before moving or renaming

### 4. Build Commit Plan
Commit grouping rules:
1. Separate formal artifacts from process notes.
2. Separate experiment execution changes from analysis narrative changes.
3. Separate structural/documentation cleanup from research content.
4. Keep each commit focused and explainable in one sentence.

Typical outcome:
- 1 to 4 commits for a mixed change set

### 5. Generate Commit Messages
Use repository-specific prefixes.

memory-poisoning-research:
- exp, feat, analysis, fix, docs

memory-poisoning-lab-notes:
- log, decision, reading, exp-note, failure, meeting, idea

Deep report routing default:
- reading: add report-YYYY-MM-DD-topic-slug deep research note

Alternative only when strongly experiment-focused:
- exp-note: add exp_###_slug deep report commentary

Message format:
- <prefix>: <specific work summary>

If relevant, add body fields:
- exp_id: exp_###_slug
- result_id: res_<exp_id>_<artifact_type>_v#
- related_note: <note filename>

### 6. Confirmation Gate
Always ask before:
1. Deletions
2. File moves with uncertain intent
3. Splitting or merging commits when intent is ambiguous
4. Any action that could hide or rewrite user intent

Do not require extra confirmation for deep report moves when all are true:
1. User explicitly asked to organize files.
2. There is a clear report-like root file.
3. Destination and naming are unambiguous by policy.

Confirmation format example:
- It looks like analysis and experiment-run updates were edited together.
- Choose:
1. Single combined commit
2. Split into exp + analysis commits
3. Review changes only (no commit)

### 7. Execute Git Actions
After approval (or if unambiguous and safe):
1. Stage planned files only.
2. Create commit(s) in logical order.
3. Re-check git status.
4. Report exact commits created.

Never push by default.

### 8. Post-Run Report
Provide:
1. What was changed
2. What was committed and why
3. Any unresolved structural issues
4. Suggested next action

## Ambiguity Handling Policy
If confidence in intent is low, pause and ask.

Low confidence examples:
- A report file placed in results/
- A daily log placed in research repo
- Mixed code/data/notes edited in one batch without clear separation

## Commit Decision Heuristics
Prefer split commits when all are true:
1. Files live in different repositories
2. Prefix categories differ
3. Changes can be understood independently

Prefer single commit when all are true:
1. One clear intent
2. One repository
3. One prefix category
