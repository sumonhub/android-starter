# Shared Git & PR Workflow (all architectures)

> Applies regardless of which architecture pattern was chosen. `/init-project` writes this
> into every project's `CLAUDE.md`, unmodified.

## Commits
- Conventional commits: `feat:`, `fix:`, `refactor:`, `test:`, `chore:`, `docs:`
- No direct commits to `main` / `develop` — always via a pull request

## Pull Requests
- PR description includes: what changed, why, screenshots (for any UI change), testing done

## Claude's own behavior
- Drafting a commit message or PR description is fine at any time
- Never push a commit, open a PR, merge, or force-push on the person's behalf without an explicit confirmation for that specific action — a general "commit this" instruction does not authorize pushing to a shared branch
