# Shared Response Format (all architectures)

> Applies regardless of which architecture pattern was chosen. `/init-project` writes this
> into every project's `CLAUDE.md`, unmodified. This governs how Claude reports its own work
> in this project — not a UI or code convention.

For any bug fix, feature, or refactor that touches code, structure the response in two parts:

## Before touching any code (diagnosis)
- **Issue** — one or two lines: what's broken or requested, in plain terms
- **Root Cause** — the actual underlying reason, traced to the specific file/function/logic — not just the symptom (e.g. not "list screen crashes", but "`OrderListViewModel.processIntent` calls `.data!!` on a `Result.Error` branch")
- **Action Plan** — the concrete steps to be taken, in order, before writing any code

## After the change is done
- **What Changed** — files touched + a concise summary of the change (not the full diff repeated)
- **Why Changed** — ties each change back to the Root Cause above — how this specific edit addresses it
- **Issue Resolved** — explicit statement of what now works, and how to verify it (what to run, what to click, what output to expect)

## When to skip this structure
- Trivial lookups, explanations, or one-line answers that don't touch code
- The person explicitly asks for a quick/short answer instead

Skipping is the exception, not the default — for anything that changes code in this project, use the full Issue → Root Cause → Action Plan, then What Changed → Why Changed → Issue Resolved structure.
