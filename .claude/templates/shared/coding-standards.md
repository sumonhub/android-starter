# Shared Coding Standards (all architectures)

> Applies regardless of which architecture pattern was chosen (MVI, MVI+Clean, MVVM,
> MVVM+Clean). `/init-project` writes this into every project's `CLAUDE.md`, unmodified,
> right after the Hard Rules (`shared/hard-rules.md` + the architecture's own).

## Code Style
- Follow the official Kotlin style guide; format with ktlint conventions
- No hardcoded dimensions or colors — use `MaterialTheme` tokens (`MaterialTheme.colorScheme`, `MaterialTheme.typography`) or `core/ui/theme/Dimens.kt`
- Prefer immutability: `val` over `var`, data classes over mutable state holders

## Error Handling
- Never swallow exceptions silently — every catch block either maps to the project's error result (the `Error` case of its sealed result type) or re-throws; an empty catch block is always a bug, not a shortcut
- Any network/DB/I/O call returns a sealed result type with Success / Error cases (whichever the architecture's `core/` defines) — never a raw nullable or a bare thrown exception as the "normal" return path. Loading is UI state in the screen's State, never a result case
- Log through the central `core/util/AppLogger` — never raw `Log.d` / `println` scattered through feature code

## Logging
- No PII, auth tokens, or passwords in any log line, at any log level
- Logging should be minimized or stripped in release builds (e.g. a no-op `AppLogger` implementation wired in via Koin for release builds)
- **Every error log must be self-contained enough to debug from the log line alone** — a bare stack trace or a raw `e.message` with no context is not enough. Include, in this order:
  1. **What operation failed** — the specific action, not just the class name (e.g. "Failed to fetch order list", not just "OrderRepository error")
  2. **Relevant identifiers/params** — IDs, query values, feature name (never PII, tokens, or passwords — see above)
  3. **The exception itself** — pass the `Throwable` to the logger so the stack trace is preserved, don't just log `e.message`
  4. **Where it happened** — a consistent tag (class or feature name), so the log can be traced back to a file
- Standard call shape: `AppLogger.e(tag = "OrderRepository", message = "Failed to fetch order list for userId=$userId", throwable = e)` — never `Log.e("TAG", e.message)` or a bare `e.printStackTrace()`
- This format matters beyond human debugging too: when a log line is copy-pasted into an AI assistant for help, one that already states the operation, the inputs, and the full exception lets the assistant diagnose the failure without needing the surrounding code first — a bare message or stack trace alone usually doesn't
