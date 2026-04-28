# cc-dev-superpowers

This file ships with the `cc-dev-superpowers` plugin. It restates the conventions the plugin's skills enforce, as ambient reinforcement when CLAUDE.md is loaded.

## Default workflow for code-writing tasks

For any non-trivial production code change, follow the `feature-development-workflow` skill:

1. Write a failing test first (`test-driven-development` skill).
2. Write the minimum code to pass.
3. Refactor with green tests.
4. Loop until the unit is complete.
5. Dispatch the `code-reviewer` subagent for a clean-context review.
6. Address critical findings.
7. Dispatch the `docs-maintainer` subagent if the change affected anything user-visible.

Skip the pipeline for throwaway scripts, doc-only changes, and trivial mechanical edits. Honor explicit user opt-out.

## Code style

Style rules are derived from canonical example files and codified in the `coding-style` skill (Tier 2 of the methodology — not yet populated). Until then, follow the conventions of the surrounding code in the consuming project.
