# cc-dev-superpowers

A Claude Code plugin that encodes an opinionated .NET development methodology: tests first, clean-context code review, and documentation that stays in sync.

## Install

```
/plugin install https://github.com/JacobIhle/CC-Dev-Superpowers
```

Once installed, the plugin's agents, skills, and permission rules become available in any Claude Code session in the consuming project.

## What's in the plugin

**Subagents** (clean-context, dispatched at workflow checkpoints):
- `code-reviewer` — reviews changes for correctness, stability, and security.
- `docs-maintainer` — keeps the README and ADRs in sync with code changes.

**Skills** (auto-load when relevant):
- `feature-development-workflow` — the default pipeline (TDD → review → docs).
- `test-driven-development` — red/green/refactor methodology.
- `reviewing-code` — used by the code-reviewer agent.
- `writing-documentation` — README and ADR conventions for greenfield docs.
- `maintaining-documentation` — used by the docs-maintainer agent for in-sync edits.

**Permissions** (`settings.json`) — auto-allows the .NET dev loop (`dotnet build/restore/clean/test/format/...`), `git fetch/pull/push`, ripgrep. Pushes to `main` or `dev`, force-pushes, and `--all`/`--mirror` pushes still prompt.

## After install

Add `tmp/` to your project's `.gitignore`. The plugin's CLAUDE.md instructs agents to use `./tmp/` for clone-and-explore work (external repos, reference material) so reads stay inside the project boundary and don't trigger permission prompts.

## Known gaps

- **`git push` to a tracking branch** isn't visible in the command string, so bare `git push` and `git push origin` slip past the `ask` rules even when the tracking branch is `main` or `dev`. Add a `PreToolUse` hook if you want airtight protection.

## Fallbacks if the plugin doesn't auto-apply

The Claude Code plugin docs don't currently confirm two behaviors this plugin relies on. If either fails after install, here's the manual workaround:

**Permissions don't apply.** Copy the `permissions` block from this repo's `settings.json` into your project's `.claude/settings.json` (or merge with what's there).

**CLAUDE.md doesn't load.** Skills auto-load reliably regardless. If you want the CLAUDE.md content as ambient reinforcement, add an `@`-import line to your project's CLAUDE.md pointing at the plugin's `CLAUDE.md` path. Run `/context` to verify it's loaded.
