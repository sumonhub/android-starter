# Shared Git & PR Workflow (all architectures)

> Applies to every project, whatever architecture pattern was chosen. `/android-starter:init-project` copies this
> into every project's `CLAUDE.md` without changes.

## Commits
- Use conventional commits: `feat:`, `fix:`, `refactor:`, `test:`, `chore:`, `docs:`
- Never commit directly to `main` / `develop`. Always go through a pull request

## Pull Requests
- A PR description includes: what changed, why, screenshots (for any UI change), and the testing done

## Claude's own behavior
- You may draft a commit message or PR description at any time
- Never push a commit, open a PR, merge, or force-push for the person unless they explicitly confirm that specific action. A general "commit this" instruction does not allow pushing to a shared branch
