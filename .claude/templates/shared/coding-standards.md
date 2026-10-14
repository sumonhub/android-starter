# Shared Coding Standards (all architectures)

> Applies to every project, whatever architecture pattern was chosen (MVI, MVI+Clean, MVVM,
> MVVM+Clean). `/init-project` copies this into every project's `CLAUDE.md` without changes.
> It goes right after the Hard Rules (`shared/hard-rules.md` + the architecture's own).

## Code Style
- Follow the official Kotlin style guide. Format with ktlint conventions
- No hardcoded dimensions or colors. Use `MaterialTheme` tokens (`MaterialTheme.colorScheme`, `MaterialTheme.typography`) or `core/ui/theme/Dimens.kt`
- Prefer immutability: `val` over `var`, and data classes over mutable state holders

## Error Handling
- Never swallow exceptions silently. Every catch block must do one of two things:
  - map the exception to the project's error result (the `Error` case of its sealed result type), or
  - re-throw it
- An empty catch block is always a bug, not a shortcut
- Every network/DB/I/O call returns a sealed result type with Success / Error cases (the one the architecture's `core/` defines). Its normal return path is never a raw nullable or a bare thrown exception
- Loading is UI state in the screen's State. It is never a result case
- Log through the central `core/util/AppLogger`. Never scatter raw `Log.d` / `println` calls through feature code

## Logging
- No PII, auth tokens, or passwords in any log line, at any log level
- In release builds, logging should be kept to a minimum or removed. For example, use Koin to wire in a no-op `AppLogger` implementation for release builds
- **Every error log must be self-contained enough to debug from the log line alone.** A bare stack trace or a raw `e.message` with no context is not enough. Include these, in this order:
  1. **What operation failed** — the specific action, not just the class name (e.g. "Failed to fetch order list", not just "OrderRepository error")
  2. **Relevant identifiers/params** — IDs, query values, feature name (never PII, tokens, or passwords — see above)
  3. **The exception itself** — pass the `Throwable` to the logger so it keeps the stack trace. Do not log only `e.message`
  4. **Where it happened** — a consistent tag (class or feature name), so you can trace the log back to a file
- Standard call shape: `AppLogger.e(tag = "OrderRepository", message = "Failed to fetch order list for userId=$userId", throwable = e)`. Never use `Log.e("TAG", e.message)` or a bare `e.printStackTrace()`
- This format also helps outside human debugging. People often paste a log line into an AI assistant for help. If the line already states the operation, the inputs and the full exception, the assistant can find the cause without first reading the surrounding code. A bare message or stack trace alone usually is not enough for that
