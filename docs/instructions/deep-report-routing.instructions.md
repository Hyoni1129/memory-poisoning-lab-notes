---
description: "Use when user mentions deep research reports, LLM-generated reference reports, root report cleanup, report routing, or asks to move and commit study reports from root. Trigger phrases include: deep report, deep research report, ingest report, organize root report, root에 보고서 정리, 보고서 옮기고 커밋, 참고용 보고서 정리."
---

# Deep Report Routing Instruction

For root-level deep research report handling:

1. Detect report-like files in workspace root.
2. If clearly a deep research reference report, move to memory-poisoning-lab-notes/04_deep_research_reports/.
3. Rename to report-YYYY-MM-DD-topic-slug.md.
4. Preserve content exactly unless rewriting is requested.
5. Commit in memory-poisoning-lab-notes using reading: prefix by default.
6. Use exp-note: only when the report is primarily experiment commentary tied to exp_id.

Safety constraints:
- If report classification is unclear, ask one short confirmation question before moving.
- Never delete original content without confirmation.
- Never push unless explicitly requested.
