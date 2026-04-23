# SKILL_PROPOSAL_GUIDE

## Purpose
Define how the agent proposes workflow skills that reduce repetitive work while keeping user control.

## What A Skill Means Here
A skill is a reusable mini-workflow that automates repeated repository management patterns.

Examples:
- recurring deep-report ingestion flow
- recurring experiment result + note linking flow
- recurring multi-commit batching pattern

## When The Agent Should Propose A Skill
Propose only when repetition is clear. Trigger conditions:
1. The same workflow pattern appears 3 or more times.
2. The pattern has stable steps and clear inputs/outputs.
3. Automating it would reduce friction without reducing clarity.

Do not propose skills for one-off tasks.

## Proposal Format (Required)
Every proposal must include:
1. Skill name (short and descriptive)
2. Problem it solves
3. Trigger phrase(s) the user can type
4. Inputs expected from user
5. Actions the skill performs
6. Outputs produced
7. Safety boundaries (what it never does automatically)
8. Estimated benefit (time saved, fewer mistakes, better consistency)

## Approval Rules
1. Never auto-create a skill without explicit user approval.
2. Ask for approval in a clear yes/no form.
3. If approved, implement in a minimal first version.
4. After first use, review and refine only with user consent.

## Safe Proposal Script
Use this pattern:

I noticed a repeated workflow: <pattern>.
I can propose a skill: <skill name>.
It would:
- <benefit 1>
- <benefit 2>
- <benefit 3>

Would you like me to draft this skill? (yes/no)

## Non-Negotiable Boundaries
Proposed skills must not:
- delete files without confirmation
- move files aggressively without explanation
- push to remote without explicit instruction
- collapse unrelated work into one commit

## Suggested Candidate Skills For This Project
1. Experiment Pairing Assistant
- ensures exp_id and result_id consistency across both repos

2. Deep Report Ingestion Assistant
- routes reports to 04_deep_research_reports and prepares matching commit messages

3. Session Finalization Assistant
- checks status, suggests commit split, and prepares commit commands
