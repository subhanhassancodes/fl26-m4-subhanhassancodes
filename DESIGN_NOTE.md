# M4 Design Note

## 1. Components and threads

- **`MainWindow`** (GUI thread): owns `Workspace`, the current `ProcessingCore`, `workspace_version_`, and a `CorpusBuilder`. Decides if a finished build is accepted. Only class that touches widgets.
- **`CorpusBuilder`** (QObject, public functions on GUI thread): `start(workspace copy, version, delay_ms)` creates a `BuildJob` and its `std::thread`. `take(id)` joins that thread and returns the result. Emits `build_finished(id)`. Destructor cancels and joins all jobs. Jobs are kept as `std::unique_ptr<BuildJob>` since `BuildJob` holds an atomic.
- **`BuildJob`**: `std::thread`, `std::future<std::unique_ptr<ProcessingCore>>`, `std::atomic<bool> cancel`, and the version it was started for.
- **`CorpusBuilder::run`** (worker thread): uses its own copy of the Workspace, sleeps `delay_ms` per document in 10 ms steps checking `cancel`, builds a new `ProcessingCore` and calls `rebuild()`, stores the core or the exception in the promise, then posts `job_done(id)` with `QMetaObject::invokeMethod(..., Qt::QueuedConnection)`.

Shared between threads: only each job's `cancel` flag (GUI writes, worker reads, atomic) and the promise/future state (synchronized by the standard library).

## 2. Corpus states

**Empty** (no documents) → **NotBuilt** (documents, no current corpus) → **Building(v)** (job for version `v` running, Build disabled) → **Current** (core matches workspace, chunk count shown) or **Failed** (NotBuilt plus an error message).

Add document: `++workspace_version_`, cancel running job, → NotBuilt. Build: → Building(v). Job done with `v == workspace_version_`: success → Current, error → Failed. Job done with an old `v`: result dropped, whether it succeeded or failed, state unchanged. Reset: `++workspace_version_`, cancel jobs, → Empty.

## 3. Threatening sequences

- Add or Reset during a build: version changes, so the result is stale and dropped.
- Reset, re-add the same files, Build again before the old job ends: the old job has an older version even with identical content, so only the new one is accepted.
- A stale build fails: the error is dropped too, since it is about a workspace that no longer exists.
- A current build fails: `future.get()` rethrows on the GUI thread, status shows the error, Build re-enabled (R3).
- A job finishes while the Exit confirmation is open: handled normally; declining Exit keeps the new state.
- Thread buildup: each job is joined in `take()` once it has finished (R4).

**A. Add during a build.** Build at version 3 starts job 7. User adds a document: version becomes 4, job 7 cancelled, state NotBuilt. Job 7 posts `job_done(7)`; `take(7)` joins it, sees version 3 ≠ 4, and destroys the core. Nothing changes on screen.

**B. Exit during a build.** `~CorpusBuilder` sets all `cancel` flags and joins every thread. A job still in the delay stops within about 10 ms. A job already inside `rebuild()` cannot be interrupted, so exit waits until that `rebuild()` returns. Any `job_done` posted before the join is discarded by Qt since the receiver is gone. No detached threads, no crash.

## 4. Ownership of one built corpus

Created by `std::make_unique` in `run()` → moved into the job's promise → held by the `future` in `BuildJob` (owned by `CorpusBuilder`, owned by `MainWindow`) → `take(id)` returns the `unique_ptr` on the GUI thread → if valid, move-assigned into `processing_core_`, otherwise destroyed at the end of `build_finished()` → an accepted core is destroyed by the next accepted build, Reset, or `~MainWindow`.

## 5. Rejected alternative

Calling `rebuild()` from the worker directly on `MainWindow`'s `processing_core_` with an `atomic<bool> building` flag. The GUI could read the core while the worker writes it, which is a data race, and a stale build would overwrite the old corpus before it could be checked.

