# nmr-next: Project Plan

**Owner:** Kyle East · **Drafted:** 1 Oct 2026 · **Pace:** 30–60 min per workday, phone only
**Companion:** daily-task roadmap with a Gantt chart (169 workday tasks, 5 Oct 2026 → 8 Jun 2027), kept as a private Claude artifact. Task ids in that roadmap (e.g. `C-3`) match the phase letters below.

---

## 1. Goals

| | Goal | Done when |
|---|---|---|
| **Goal 1** | Fork NMRdock, modernize the container, and wrap NMRPipe in an API + database + web app that replaces nmrDraw. | Your group can upload Bruker/Varian data, process 1D/2D/3D (including NUS and LP), view/phase spectra in a browser, and find their data and recipes again later. Target: **v1.0 ~24 Feb 2027**. |
| **Goal 2** | Replace the NMRPipe engine with a Rust processing framework, without changing the API or web app. | Rust engine matches NMRPipe within agreed tolerances on all fixtures; it is the default backend; NMRPipe remains an optional legacy backend. Target: **v2.0 ~8 Jun 2027**. |

**Scope agreed so far:** 1D, 2D, 3D triple-resonance, NUS reconstruction and linear prediction · users: your group now, enterprise later · hosting: home (EVO-X1) now, Azure/AWS later.

---

## 2. Guiding principles

1. **One processing language, two engines.** The API accepts a backend-neutral *processing recipe* (JSON). Goal 1 translates it to an nmrPipe script; Goal 2 runs it natively in Rust. The web app and database never know which engine ran.
2. **NMRPipe is the reference, not just the engine.** Every processed output from Goal 1 becomes a "golden" reference that Goal 2 must reproduce.
3. **Keep NMRPipe's file format as the interchange format.** `.ft1/.ft2/.ft3` output keeps compatibility with Sparky, POKY, CcpNmr, TALOS, etc.
4. **Design for enterprise and cloud now, deploy them later.** OIDC login from day one, S3-style object storage, config via environment variables, everything in containers.
5. **Phone-first workflow.** Each daily task is sized to be started from your phone: prompt an agent → it opens a PR → CI runs → you review and merge.

---

## 3. Target architecture

```
                 ┌──────────────────────────────────────────┐
  Browser  ───▶  │  Web app (React + TypeScript, WebGL)     │
  (phone/desk)   │  viewer · phasing · recipe builder       │
                 └───────────────┬──────────────────────────┘
                                 │ HTTPS (OIDC token)
                 ┌───────────────▼──────────────────────────┐
                 │  API (FastAPI)                            │
                 │  datasets · recipes · jobs · spectra      │
                 └──────┬───────────────┬───────────────────┘
                        │               │
             ┌──────────▼───┐   ┌───────▼────────┐   ┌──────────────┐
             │ PostgreSQL   │   │ Redis queue    │   │ Object store │
             │ metadata,    │   │                │   │ MinIO → Blob │
             │ provenance   │   └───────┬────────┘   │ / S3: FIDs,  │
             └──────────────┘           │            │ .ft results  │
                              ┌─────────▼─────────┐  └──────────────┘
                              │ Worker            │
                              │  backend = nmrpipe│ ◀── Goal 1: engine image
                              │  backend = rust   │ ◀── Goal 2: engine-rs image
                              └───────────────────┘
  Auth: Keycloak (home) → Entra ID / Cognito (enterprise)
  Network: Tailscale (home) → cloud ingress later
```

### Recommended stack (each confirmed in an ADR before building)

| Layer | Recommendation | Why |
|---|---|---|
| Base image | Ubuntu 24.04 LTS, multi-stage | Current LTS; i386 multiarch still available for NMRPipe binaries |
| Engine (Goal 1) | NMRPipe in headless `engine` image; optional `gui` image with noVNC | API doesn't need X11; noVNC lets you open legacy nmrDraw in a phone browser |
| API | Python FastAPI + Redis queue | Python fits the NMR ecosystem (nmrglue for reading files); async jobs for long 3D/NUS runs |
| Database | PostgreSQL + MinIO (S3 API) — **placeholder until you define the database** | Metadata in SQL, large spectra as objects; maps directly to Azure Blob / S3 |
| Auth | OIDC via Keycloak | Swapping to Entra ID / Cognito later is configuration, not a rewrite |
| Web | React + TypeScript (Vite), WebGL plotting | 2D/3D contour plots need GPU rendering to stay smooth |
| Engine (Goal 2) | Rust: `ndarray`, `rustfft`/`realfft`, `rayon`, `memmap2`, `PyO3`, `wasm-bindgen` | Fast, memory-safe, parallel; Python bindings for the worker; WASM for live phasing in the browser |
| CI/CD | GitHub Actions → GHCR | Builds and tests run in the cloud, so you can drive everything from your phone |

---

## 4. Goal 1 — NMRPipe behind a modern API and web app

### Phase F · Foundations (≈2 weeks)
1. Fork `compbiocore/nmrdock` (done: `kyleeast13/nmr-next`); enable Issues and Projects.
2. Set up phone tooling: Tailscale on iPhone, Claude Code connected to the fork, GitHub mobile.
3. Create the project board, labels and an ADR (Architecture Decision Record) folder.
4. **Supply-chain safety:** archive the NMRPipe installer files with SHA-256 checksums; check the dead-or-alive status of the Brown S3 URLs the old Dockerfile depends on.
5. **Licensing:** read NMRPipe's current terms for archiving installers and for processing data on behalf of other users (matters for enterprise).
6. ADR-001: overall architecture.

### Phase C · Modern container (≈3 weeks)
1. Audit the existing Dockerfile (Ubuntu 16.04, Miniconda2/Python 2, i386 libs, Sparky, relax, two Brown S3 dependencies).
2. Rebuild on Ubuntu 24.04: multi-stage, non-root, pinned versions, no Python 2.
3. Install NMRPipe from archived installers with checksum verification; CI checks that 32-bit binaries load.
4. Split into `engine` (headless) and `gui` (nmrDraw/Sparky via noVNC).
5. Move `testdata/` into Git LFS fixtures; CI smoke tests on bruker2D/3D and varian2D/3D.
6. Save processed outputs as golden references. Tag **v0.1.0**.

### Phase R · Processing recipe and engine wrapper (≈2.5 weeks)
1. **You** list the NMRPipe functions you actually use (top 20–30) — this is the spec for both goals.
2. Define the recipe JSON schema: dataset → conversion → per-dimension function chain (incl. NUS and LP steps).
3. Write example recipes from your real `fid.com` / `nmrproc.com` scripts.
4. Build the translator (recipe → csh nmrPipe pipeline) for 1D, 2D, 3D.
5. Auto-generate conversion steps from Bruker `acqus` / Varian `procpar`.
6. Engine runner returns outputs + logs; tests prove recipes reproduce golden outputs.

### Phase A · API and job queue (≈2.5 weeks)
1. FastAPI skeleton with OpenAPI docs and docker compose.
2. Upload (zipped vendor directory, auto-detect vendor) → submit recipe → queued job → worker.
3. Job status, logs and live progress.
4. Data endpoints the viewer needs: 1D traces, 2D planes/contours, 3D planes and strips, quick phasing preview.
5. Result downloads (`.ft` files + generated script).

### Phase D · Database and storage (≈2 weeks)
1. **You** define the database requirements (what to store about users, samples, experiments, spectra).
2. Schema v1 + migrations; uploads/results in MinIO.
3. Provenance: every result records recipe version, engine, engine version, user, time.
4. Searchable metadata from `acqus`/`procpar` (nucleus, field, temperature, pulse program).
5. Backups with a tested restore.

### Phase U · Authentication and users (≈1.5 weeks)
1. Keycloak with a realm for your group; API validates tokens.
2. Project roles: owner / member / viewer; audit log.
3. Onboard group members.

### Phase W · Web app — the nmrDraw replacement (≈4.5 weeks)
1. Projects, upload, dataset pages.
2. 1D viewer with interactive phasing.
3. Recipe builder (and import from existing `fid.com` / `nmrproc.com`).
4. 2D contour viewer, traces/projections, row/column phasing.
5. 3D plane browser and strip plots.
6. NUS/LP options, job queue view, threshold peak picking, exports (peak lists, images).
7. Phone layout pass and a real end-to-end usability run.

### Phase H · Home deployment and v1.0 (≈2 weeks)
1. Production compose on the EVO-X1, HTTPS inside your tailnet via Tailscale Serve.
2. Worker resource limits, monitoring, nightly backups.
3. Cloud-readiness checklist (no cloud deploy yet). Release **v1.0** + short user guide.

---

## 5. Goal 2 — Rust processing framework

### Phase RS · Rust ramp-up and workspace (≈1.5 weeks)
- Rust basics (ownership especially), `ndarray`.
- Cargo workspace `nmr-rs`: crates `io`, `core`, `nus`, `cli`, `py`. Rust CI (fmt, clippy, tests).
- **Parity harness:** compares Rust output with NMRPipe golden files (RMSD, max absolute difference, peak positions/intensities) per fixture.

### Phase IO · File formats (≈2 weeks)
- NMRPipe format read/write (512-float header; 1D–4D; file series and streams; memory-mapped).
- Bruker (`acqus`/`fid`, digital filter / GRPDLY) and Varian/Agilent (`procpar`/`fid`) readers; JEOL optional.
- Parity with `bruk2pipe` / `var2pipe`.

### Phase P1 · Core 1D processing (≈2.5 weeks)
- Data model with quadrature modes; EM, GM, SP; ZF, EXT; FT; PS; POLY; SOL; digital-filter handling.
- CLI that runs a recipe JSON. 1D parity on all fixtures.

### Phase P2 · 2D and 3D (≈2 weeks)
- TP/ZTP transposes, indirect-dimension quad modes, first-point scaling options.
- `rayon` parallelism and out-of-core 3D via memory mapping. Benchmarks vs NMRPipe on the EVO-X1.

### Phase LP · Linear prediction (≈1.5 weeks)
- Forward, forward-backward, mirror-image LP; root reflection. Parity vs NMRPipe.

### Phase NUS · NUS reconstruction (≈3 weeks)
- Sampling schedules and NUS data layout.
- IST (1 then 2 indirect dimensions); parity vs NMRPipe's IST output.
- SMILE-style reconstruction **implemented from the published paper only** (not from NMRPipe binaries); compare quality on your lab's NUS data.

### Phase INT · Integration and WebAssembly (≈2 weeks)
- PyO3 bindings; `engine-rs` worker image (no NMRPipe, no 32-bit libraries).
- API backend selection per job; **shadow mode** runs both engines and logs differences automatically.
- Core compiled to WebAssembly for live 1D/2D phasing in the browser.

### Phase CUT · Cutover and v2.0 (≈1.5 weeks)
- Rust becomes the default (1D/2D first, then 3D/LP/NUS after your sign-off).
- NMRPipe kept as optional legacy backend. Choose a license for `nmr-rs`. Release **v2.0**.

---

## 6. Validation strategy

| Level | What's checked | When |
|---|---|---|
| Image | NMRPipe binaries load; conversion works | Every PR (CI) |
| Recipe | Generated scripts reproduce golden outputs | Every PR touching engine/translator |
| Parity | Rust vs NMRPipe within tolerance on every fixture | Every Rust PR |
| Shadow | Both engines on real jobs; differences logged | Phase INT onward |
| Expert | You inspect spectra and peak lists side by side | End of each phase |

Suggested tolerances to start with (adjust after first comparisons): normalized RMSD < 1e-4 for linear operations; peak positions within 0.5 digital points; NUS/LP judged by peak intensity correlation and artifact level rather than point-by-point identity.

---

## 7. Risks and mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| NMRPipe download site disappears | Can't rebuild the image | Archive installers + checksums in Phase F |
| Brown S3 resources already gone | Old Dockerfile can't build | Replace with explicit package lists in the new Dockerfile |
| NMRPipe terms restrict hosting for others | Blocks enterprise use of Goal 1 | Check in Phase F; Goal 2 removes the dependency |
| 32-bit libraries drop out of future Ubuntu | Image breaks on upgrade | Pin base image; Goal 2 removes the need |
| NUS/LP parity is hard | Goal 2 slips | Parity measured on spectral quality, not bit-identity; NMRPipe stays as fallback backend |
| Large 3D/NUS jobs swamp the EVO-X1 | Box becomes unresponsive | Worker CPU/RAM limits, one heavy job at a time |
| Data exposure | Unpublished research leaks | Tailscale-only access, OIDC, audit log, backups |
| Phone-only work hits a wall (e.g. debugging a GUI) | Days lost | Buffer day at the end of each phase; save desk-only tasks for evenings/weekends |

---

## 8. A typical workday from your phone

1. Open the Gantt chart, check today's task.
2. Start a Claude Code session on the fork with a focused prompt (e.g. *"Add an upload endpoint that accepts a zipped Bruker directory, stores it in MinIO, and detects the vendor. Include tests. Open a PR."*).
3. While it works, read the related docs or the previous PR.
4. Review the PR and CI results in GitHub mobile; ask for fixes in the same session.
5. Merge, tick the task off, and note anything that changes the plan as an issue.

Anything needing your home box (pulling images, running heavy 3D jobs) runs through a Claude Code session on the EVO-X1 reached over Tailscale.

---

## 9. Open decisions

- [ ] Database requirements (yours to define — Phase D, task 1)
- [ ] NMRPipe terms for archiving and multi-user hosting (Phase F)
- [ ] Your top 20–30 NMRPipe functions (Phase R, task 1)
- [ ] Keep Sparky and relax in the `gui` image, or drop them?
- [ ] Repository license for the fork (existing NMRdock is GPL-3.0) and for `nmr-rs`
- [ ] Cloud target when the time comes: Azure vs AWS
