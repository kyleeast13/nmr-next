# Architecture Decision Records

Each significant choice (base image, API framework, database, auth, front-end stack, Rust crates) gets a short record here, so later work and later Claude sessions know what was decided and why.

- File name: `NNNN-short-title.md`, numbered in order (`0001-architecture.md`, `0002-base-image.md`, …).
- Copy `0000-template.md` to start one.
- Status moves `Proposed` → `Accepted` (or `Rejected`). To change a decision, write a new ADR that supersedes the old one; don't rewrite history.

| ADR | Title | Status |
|---|---|---|
| 0001 | Overall architecture | Planned (Phase F) |
| 0002 | Container base image | Planned (Phase C) |
| 0003 | API framework and job queue | Planned (Phase A) |
| 0004 | Database and object storage | Planned (Phase D) |
| 0005 | Authentication (OIDC) | Planned (Phase U) |
| 0006 | Web front-end stack | Planned (Phase W) |
| 0007 | Rust crates for nmr-rs | Planned (Phase RS) |
