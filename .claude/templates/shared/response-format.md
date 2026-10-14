# Shared Response Format (all architectures)

> Applies to every project, whatever architecture pattern was chosen. `/init-project` copies this
> into every project's `CLAUDE.md` without changes. It sets how Claude reports its own work
> in this project. It is not a UI or code convention.

For any bug fix, feature, or refactor that touches code, write the response in two parts:

## Before touching any code (diagnosis)
- **Issue** — one or two lines, in plain words: what is broken or what was asked for
- **Root Cause** — the real underlying reason, traced to the specific file/function/logic. Do not just describe the symptom. For example, not "list screen crashes", but "`OrderListViewModel.processIntent` calls `.data!!` on a `Result.Error` branch"
- **Action Plan** — the concrete steps you will take, in order, before you write any code

## After the change is done
- **What Changed** — the files you touched, plus a short summary of the change. Do not repeat the full diff
- **Why Changed** — link each change back to the Root Cause above. Say how this specific edit fixes it
- **Issue Resolved** — state clearly what now works and how to check it (what to run, what to click, what output to expect)

## When to skip this structure
- Simple lookups, explanations, or one-line answers that don't touch code
- The person explicitly asks for a quick/short answer instead

Skipping is the exception, not the default. For anything that changes code in this project, use the full Issue → Root Cause → Action Plan structure, then What Changed → Why Changed → Issue Resolved.
