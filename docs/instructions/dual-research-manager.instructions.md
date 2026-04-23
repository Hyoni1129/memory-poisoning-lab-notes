---
description: "Use when user asks to analyze changes, organize files, manage commits, maintain naming conventions, enforce cross-repository consistency, or route deep research reports in the memory-poisoning dual-repository project. Trigger phrases include: organize and commit, analyze changes only, check cross-repo consistency, commit everything, deep research report, root report organize, root에 보고서 정리, 보고서 옮기고 커밋."
---

# Dual-Repository Research Manager

For repository management tasks in this workspace:

1. Follow .github/research-workflow/AGENT_WORKFLOW.md as the execution process.
2. Apply naming and placement rules from .github/research-workflow/AGENT_GUIDE.md.
3. Use commit prefix and grouping rules from .github/research-workflow/AGENT_GUIDE.md.
4. Use skill proposal constraints from .github/research-workflow/SKILL_PROPOSAL_GUIDE.md.
5. Offer quick command style interaction from .github/research-workflow/QUICK_COMMANDS.md.

Behavioral requirements:
- Analyze both repositories before proposing git actions.
- Keep commits small and meaningful.
- Ask confirmation when ambiguity exists.
- Never push unless explicitly requested.
- If a root-level deep research report is obvious and user asked to organize, route it to memory-poisoning-lab-notes/04_deep_research_reports/ and normalize filename.
