# nmr-next

Fork of `compbiocore/nmrdock` (originally by Andrew Leith and Kyle East, Brown CBC). The goal is to turn a Docker image that wraps NMRPipe into a modern bioNMR processing platform, then replace NMRPipe with a Rust engine.

The owner, Kyle, is an NMR spectroscopist. He knows NMRPipe and bioNMR processing deeply, and IT well, but is not a professional software engineer. He drives most work **from his phone during the workday, in 30–60 minute sessions**. Write code, PRs and explanations with that in mind.

Full plan: `docs/plan.md`. Decisions: `docs/adr/`.

## The two goals

1. **Goal 1 (v1.0):** modernize the container, wrap NMRPipe behind an API (FastAPI + job queue), add a database and object storage, and build a web app that replaces nmrDraw. Users: Kyle's research group now, enterprise later. Hosting: his home server (GMKtec EVO-X1, Windows 11 + WSL + Docker, reached over Tailscale) now, Azure/AWS later.
2. **Goal 2 (v2.0):** a Rust processing engine (`nmr-rs`) that runs the same processing recipes and matches NMRPipe output, then becomes the default backend.

Scope for both: 1D, 2D, 3D triple-resonance, NUS reconstruction (SMILE, IST) and linear prediction. Vendors: Bruker and Varian/Agilent first.

## Core design rule: one recipe, two engines

The API accepts a backend-neutral **processing recipe** (JSON) describing conversion plus a per-dimension function chain (SP, ZF, FT, PS, EXT, TP, POLY, LP, NUS…). Goal 1 translates a recipe into an nmrPipe csh pipeline. Goal 2 executes the same recipe in Rust. The web app and database must never depend on which engine ran. Do not add engine-specific fields to API responses without an ADR.

NMRPipe's own file format (`.fid`, `.ft1/.ft2/.ft3`) stays the interchange format so outputs keep working in Sparky, POKY, CcpNmr and TALOS.

## Current repo layout (legacy, from the original NMRdock)

| Path | What it is |
|---|---|
| `Dockerfile` | Ubuntu 16.04 image: i386 libs, csh/tcsh, Miniconda2 (Python 2), NMRPipe downloaded from ibbr.umd.edu at build, NMRFAM-Sparky and relax. Depends on two Brown S3 URLs that may be gone. Being rebuilt in Phase C. |
| `nmr_wrappers/` | csh shims (`bruker`, `varian`, `nmrDraw`, `sparky`) put on PATH in the old image |
| `testdata/` | Real datasets: `bruker2D` (600 MHz ¹H-¹⁵N HSQC, echo-antiecho), `bruker3D`, `varian2D`, `varian3D`, `config`, plus `testcommands.txt`. Several include the original `fid.com` / `nmrproc.com`. These are the reference fixtures for every test. |
| `docs/` | Old MkDocs site (index, overview, walkthroughs, media) plus the new `plan.md` and `adr/` |
| `macQS`, `linuxQS`, `windowsQS.txt`, `ipconfig.PNG` | Old X11 quick-start helpers. Will be replaced by noVNC in the `gui` image. |
| `.travis.yml`, `ready.yml`, `mkdocs.yml` | Legacy CI/docs config. Travis is dead; GitHub Actions replaces it in Phase C. |

Planned layout as the work lands (create directories only when the task needs them):

```
docker/engine/   headless NMRPipe image (for the API worker)
docker/gui/      legacy nmrDraw/Sparky image with noVNC
recipe/          Python package: recipe schema + recipe→nmrPipe translator
api/             FastAPI app and worker
web/             React + TypeScript (Vite) front end
nmr-rs/          Rust workspace (Goal 2)
tests/fixtures/  test data (Git LFS) and golden outputs
docs/            plan, ADRs, user guide
```

## Rules

- **Never commit NMRPipe binaries or installers.** They are archived privately; the image fetches them by URL/secret and verifies SHA-256 checksums. Check NMRPipe's license terms before anything that redistributes it.
- **Golden outputs are the source of truth.** Don't regenerate or edit files under a golden/reference directory to make a test pass. If an output legitimately changes, say so in the PR and explain why.
- **Don't modify the original datasets in `testdata/`.** Copy or extract them in tests.
- **SMILE and other published algorithms are implemented from their papers only**, never by decompiling or copying NMRPipe.
- Prefer small, single-purpose PRs that match one task in the plan. One PR ≈ one 30–60 minute review on a phone.
- Don't add a dependency, service or framework that the plan's ADRs don't cover without asking first or proposing a new ADR.
- No secrets in the repo. Config comes from environment variables (12-factor) so home and cloud deployments differ only in config.
- Auth is OIDC from the start (Keycloak at home, Entra ID / Cognito later). Don't build a custom password system.

## NMR conventions

- Use NMRPipe function names and argument spellings (`SP -off 0.5 -end 1.00 -pow 1 -c 0.5`, `PS -p0 -p1 -di`, `EXT -x1 11ppm -xn 5.5ppm -sw`) in recipes, docs and UI labels. Kyle and other users think in these terms.
- Chemical shifts in ppm, spectral widths in Hz, observe frequencies in MHz, phases in degrees.
- Dimension labels follow NMRPipe: `x` is the directly detected dimension; `y`, `z`, `a` are indirect.
- When unsure about NMR semantics (quadrature modes, digital filter handling, NUS schedules), ask Kyle rather than guess.

## Working from a phone

- Kyle reviews PRs in GitHub mobile and the Claude app. Write PR descriptions that are readable on a phone: a 2–3 sentence summary, a short "How to check" list, and anything he must decide, at the top.
- CI (GitHub Actions) does the building and testing, so a PR should be verifiable from its checks without a local machine.
- Anything that must run on the home server (pulling images, heavy 3D or NUS jobs) is done in a Claude Code session on the EVO-X1 over Tailscale. Say clearly in the PR when a step needs that.
- Keep explanations plain. Define a software term the first time it matters; skip jargon that doesn't help him decide.

## Git

- Default branch: `master`. Work on a branch per task; name it after the plan task, e.g. `c3-multistage-dockerfile`.
- Commit messages: imperative, short subject, body explaining why.
- Open a PR for every change; don't push to `master` directly.
