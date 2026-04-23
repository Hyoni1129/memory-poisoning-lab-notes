# AGENT_GUIDE

## Mission
This agent manages research workflow across two repositories so the researcher can focus on writing, running experiments, and thinking.

Managed repositories:
- memory-poisoning-research: formal, reproducible research artifacts
- memory-poisoning-lab-notes: process notes, logs, decisions, and reflective context

## Operating Rules
1. Analyze first, act second.
2. Read changed files before deciding commit strategy.
3. Keep commits small, meaningful, and traceable.
4. Preserve dual-repo separation.
5. Require confirmation for risky actions.
6. Do not push unless explicitly requested.

## Default Agent Loop
1. Detect changes in both repositories.
2. Infer intent from file contents and paths.
3. Validate placement and naming.
4. Propose commit grouping.
5. Ask confirmation if ambiguous or risky.
6. Stage and commit with specific messages.
7. Report final status and next recommendations.

## Commit Conventions (Strict Mode)
### memory-poisoning-research prefixes
- exp: experiment setup/run/result updates
- feat: new research capability or implementation
- analysis: interpretation, failure analysis, insights
- fix: bug fixes and corrections
- docs: documentation updates

### memory-poisoning-lab-notes prefixes
- log: daily progress notes
- decision: decision records
- reading: reading notes or paper summaries
- exp-note: experiment commentary notes
- failure: failure analysis and lessons
- meeting: meeting notes
- idea: idea capture and refinement

Message pattern:
- <prefix>: <specific summary>

Optional commit body fields when relevant:
- exp_id: exp_###_slug
- result_id: res_<exp_id>_<artifact_type>_v#
- related_note: note filename in lab-notes

## File Organization Rules
Place files by intent, not convenience:

- Research code and scripts -> memory-poisoning-research/src/, scripts/
- Experiment definitions/configs -> memory-poisoning-research/experiments/, configs/
- Raw/processed/final datasets -> memory-poisoning-research/data/
- Result artifacts (tables/figures/logs) -> memory-poisoning-research/results/
- Formal interpretation writeups -> memory-poisoning-research/analysis/
- Paper drafting artifacts -> memory-poisoning-research/paper/

- Daily logs -> memory-poisoning-lab-notes/03_daily_log/
- Decisions -> memory-poisoning-lab-notes/02_decision_log/
- Experiment process notes -> memory-poisoning-lab-notes/05_experiment_notes/
- Failures -> memory-poisoning-lab-notes/06_failures/
- Meetings -> memory-poisoning-lab-notes/07_meetings/
- Reading and paper notes -> memory-poisoning-lab-notes/08_reading_list/ or 01_background/papers/
- Deep reports -> memory-poisoning-lab-notes/04_deep_research_reports/
- Idea notes -> memory-poisoning-lab-notes/09_ideas/

## Root Intake Rules (Deep Research Report)
When the user says "organize" or equivalent and a report was created in workspace root:
1. Detect report-like files first (typically .md, .txt, .pdf).
2. If it is a deep research reference report, move to memory-poisoning-lab-notes/04_deep_research_reports/.
3. Normalize filename to report-YYYY-MM-DD-topic-slug.md.
4. Keep original content intact unless the user requested rewriting.

Recommended commit mapping for lab-notes:
- reading: add or organize deep research report

Conditional alternative:
- exp-note: use only if report is primarily experiment commentary tied to a specific exp_id.

## Naming Rules
- Experiment ID: exp_###_<short_snake_case_slug>
- Result ID: res_<exp_id>_<artifact_type>_v#
- Daily note: daily-YYYY-MM-DD.md
- Decision note: decision-YYYY-MM-DD-topic-slug.md
- Experiment note: exp-note-exp_###_slug.md
- Meeting note: meeting-YYYY-MM-DD-topic-slug.md
- Failure note: failure-YYYY-MM-DD-topic-slug.md
- Idea note: idea-YYYY-MM-DD-topic-slug.md
- Deep report: report-YYYY-MM-DD-topic-slug.md

## Deep Report Heuristics (Operational)
A root file is treated as a deep research report when one or more are true:
1. Filename contains deep, report, research, survey, or llm.
2. Content has report-style sections (for example summary, key findings, references, limitations).
3. The user explicitly identifies it as a study/reference report.

If confidence is low, ask confirmation before moving.

## Cross-Repository Consistency Checklist
Before commit:
1. Shared experiment IDs are identical across both repos.
2. Experiment status in memory-poisoning-research/experiments/experiment_registry.md is updated if needed.
3. Related lab note filenames include matching exp_id where relevant.
4. Commit body includes exp_id/result_id links when cross-repo traceability is useful.

## Safety Rules
- Never delete files without explicit confirmation.
- Never move files automatically if intent is unclear.
- Never overwrite user-authored content silently.
- Never create noisy micro-commits.
- Never fabricate commit meaning.

## How To Interact With The Agent
Use direct commands such as:
- analyze changes only
- organize and commit
- commit everything (safe grouping)
- check cross-repo consistency
- propose new skill opportunities

If ambiguity exists, the agent asks a short decision question with numbered options.
