# Commit Message Guide

Use practical, prefix-based commit messages.

## Prefixes
- `log`: daily log additions/updates
- `decision`: decision documentation
- `reading`: reading notes and summaries
- `exp-note`: experiment commentary updates
- `report`: deep research report updates
- `meeting`: meeting notes
- `failure`: failure analysis notes
- `idea`: idea log updates
- `docs`: documentation changes
- `chore`: maintenance

## Message Pattern
`<prefix>: <short summary>`

Examples:
- `log: add daily-2026-04-24 note`
- `decision: document bm25 baseline adoption`
- `reading: add note for poison memory attack survey`
- `exp-note: analyze exp_002_poison_authority retrieval mismatch`

## Optional Cross-Repo Reference In Body
```text
exp_id: exp_002_poison_authority
result_id: res_exp_002_poison_authority_figure_v1
research_repo_ref: feat: add retrieval scoring patch
```

Consistency is the priority. Keep commits small and meaningful.
