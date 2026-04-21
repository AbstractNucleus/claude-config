---
name: engine/UI thread boundary
description: How the Qt-free engine notifies the Qt UI when a run ends on its own
type: project
originSessionId: 28d8dfd0-b146-4a83-81ca-d597800cc47b
---
`ClickEngine` is Qt-free by spec §9.6 (engine/ excluded from Qt coupling). When a run finishes (stop condition hit OR manual stop), the engine fires an optional `on_finished: Callable[[], None]` from the worker thread — no Qt knowledge.

The composition root (`app.py`) wires `on_finished=window.run_finished.emit`. `run_finished` is a `Signal()` on `MainWindow`; Qt auto-queues cross-thread signal emissions, so the handler runs safely on the main thread.

**Why:** Keeps the engine portable/testable without Qt, and avoids polling/timers. The emit-from-worker pattern is only safe because Qt signals use queued connections across threads — don't replace it with a direct callback that touches widgets.

**How to apply:** If you need any other engine→UI event (progress, error, click-count), add another `Signal` on `MainWindow` and pass `signal.emit` as the engine callback. Never import Qt into `engine/`. The `_on_run_finished` handler in `app.py` also double-fires during manual stop (because `_stop` already resets state); the handler must stay idempotent.
