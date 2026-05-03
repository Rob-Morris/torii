# T020 — Preserve typed model errors in gRPC helper

Status:
- queue: `02-later`

Summary:
- stop erasing typed storage lookup errors in `DojoWorld::model()`

Why this task exists:
- `crates/grpc/server/src/lib.rs` has a `DojoWorld::model()` helper that wraps
  `storage.model()` in `anyhow!("Failed to get model from cache: {}")`
- that collapses `StorageError::ModelNotFound { .. }` and other typed storage
  variants back into an untyped error if the helper gains call sites
- it reintroduces the same kind of error-shape loss that `T009` was removing
  from the storage surface

Why later:
- there are no current in-repo call sites for the helper
- fixing it cleanly may require changing a helper signature or deciding the
  intended gRPC not-found behavior at that boundary
- worth keeping queued, but not worth widening the typed-storage-error PR scope
