# Commit Linking System

This repository stays loosely linked to `memory-poisoning-research` through naming and documentation conventions only.

## Linking Rules
1. Use shared experiment IDs across both repositories.
2. Mention related experiment/result IDs in commit body when useful.
3. Use note filenames that include either date or experiment ID.
4. Keep links lightweight; no submodule or subtree coupling.

## Suggested Commit Body Fields
```text
exp_id: exp_###_slug
result_id: res_exp_###_slug_table_v1
related_note: decision-YYYY-MM-DD-topic-slug.md
```

## Example Pairing
- Research repo commit:
  - `exp: run exp_002_poison_authority initial setup`
- Lab-notes repo commit:
  - `exp-note: analyze exp_002_poison_authority poisoning phrasing`
