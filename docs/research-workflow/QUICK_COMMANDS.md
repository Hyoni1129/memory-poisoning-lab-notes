# .quick-commands

Use these direct commands with the agent.

## Core Commands
- analyze changes only
  - inspects both repos, reports findings, makes no edits or commits

- organize and commit
  - validates placement/naming, proposes commit groups, asks confirmation, commits

- commit everything (safe grouping)
  - commits all tracked changes but still splits by intent and repository

- check cross-repo consistency
  - verifies exp_id and result naming/linking across both repos

- suggest file moves
  - proposes placement fixes without moving files automatically

## Workflow Commands
- prepare experiment commit plan for exp_###_slug
  - builds staged commit plan for experiment-related files only

- prepare lab-notes commit plan for YYYY-MM-DD
  - groups daily/decision/reading/exp-note updates for a session

- analyze changes and ask split options
  - forces confirmation step before any staging

- organize root deep report and commit
  - detects report file in root, moves to 04_deep_research_reports, normalizes name, and commits

- ingest deep research report from root
  - same as above, but includes a short dry-run summary first

- deep report dry run
  - shows proposed destination/name/commit message without moving or committing

## Skill Commands
- propose new skills from current workflow
  - detects repetition and drafts skill proposals for approval

- list candidate skills
  - shows currently useful skill opportunities, no creation

## Safety Commands
- dry run only
  - simulates actions and commit plan without touching git index

- stop before commit
  - analyze and organize, but do not stage or commit
