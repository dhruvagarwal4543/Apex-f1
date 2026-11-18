# Publication verification — 12 November 2026

Verification used a separate directory exported exclusively from the reviewed Git index. It contained no original `node_modules`, virtual environment, frontend build, raw cache or ignored local artifacts. A new external Python 3.11.9 environment was installed from the project's requirements; its exact resolved Windows environment is recorded in `requirements-lock.txt`. Installing that lockfile and `pip check` both succeeded.

The clean copy ran its own FastAPI instance on a verified-free port (8013), while existing application instances were left untouched. Frontend tests used that clean copy's installed dependencies and production build. No training, recalibration, model-artifact modification, dataset replacement or force-push occurred.

## Results

| Check | Result |
|---|---|
| Frozen-artifact integrity | 409 data/model/public-asset files verified by SHA-256 and length; 121,487,192 bytes |
| Python offline suite | **268 passed**, 1 live test deselected, 0 failures; 192.28 seconds |
| Python dependencies | Lockfile installation passed; `pip check`: no broken requirements |
| Frontend installation | Clean `npm ci` passed using the committed lockfile |
| Production build | Passed using **Vite 6.4.6**; clean-copy final build 1.86 seconds |
| Frontend unit suite | **43 passed**, 0 failed |
| Main browser journey | **121 checks passed**, no unexpected browser/API/asset errors |
| GLB integration | All five bodies across a valid 22-driver/11-constructor field; cache, camera, failure handling and final-rank synchronization passed |
| Constructor logos | **79 checks passed**; all 65 peak constructors verified, 37 era-compatible logos and badge fallbacks |
| Portraits | 194/208 coverage; all 5 recently added portraits, 14 missing-photo fallbacks and failed-image fallback passed |
| GPU | Hardware WebGL on NVIDIA RTX 4060 Laptop GPU; Spa Overview/Follow, Monaco Onboard and heavy-rain checks passed at approximately 240 FPS in the final isolated run |
| npm audit | **0 vulnerabilities** after the compatible Vite update |
| Repository review | No credential/private-key or machine-specific-path findings in reviewed text; no environments, node_modules or caches staged; whitespace review passed |
| README imagery | Nine real 1600×960 captures, zero browser errors, compressed to WebP; total 1,205,462 bytes |

The main browser journey covers editing, alphabetical lists, accent-insensitive search, removed badges, P1–P22 labels, drag-and-drop, full-constructor seat swaps, Undo/Redo, invalid-grid blocking, changing model/circuit/weather, real inference, start lights, pause/resume, live timing, camera tracking, final podium/full standings, repeat prediction and mobile width. Randomized motion unit tests use 22 cars and densely sample non-overlap during overtakes.

## Dependency change

The inherited Vite 5.4.11 development dependency had audited advisories. A same-major patch did not resolve the current advisory set. **Vite 6.4.6** is supported by the existing `@vitejs/plugin-react` 4.3.4 peer range, retains Node 18/20 compatibility, uses patched esbuild, and clears the npm audit. The final clean-copy frontend build, unit, main browser, GLB and GPU checks were repeated with this exact version. No UI or prediction logic was changed for the upgrade. See [Vite's security advisories](https://github.com/vitejs/vite/security/advisories), including [the advisory patched in 6.4.6](https://github.com/vitejs/vite/security/advisories/GHSA-p4f5-76q4-ffq7).

Pillow is now an explicit pinned direct dependency because circuit geometry and screenshot optimization import it. The serialized-model dependency versions remain unchanged.

## Repository and screenshots

The publication tree is approximately 126 MiB before Git compression. Its largest file is the 17.98 MiB frozen feature CSV. All required artifacts fit ordinary Git, so there is no LFS setup or private download step. There are 95 serialized model bundles: production models plus the evaluation/selection inputs required by retained tests and scientific workflows.

Real screenshot capture used the actual UI to replace both Toyota seats with peak Aston Martin cars, retaining 22 unique drivers and 11 two-seat constructors. It selected Logistic Regression and Ridge, configured damp conditions, ran a seeded prediction, captured live order and modern-car Follow/Onboard views, then checked all 22 final standings against the returned model prediction. No API responses were mocked. Capture provenance is in `assets/screenshots/capture.json`; the reusable capture/optimization scripts are committed.

The index-export check established reproducibility before publication. Post-push verification of the actual public GitHub repository then succeeded:

- Repository: `https://github.com/dhruvagarwal4543/Apex-f1`, public, default branch `main`.
- Tested implementation commit: `0467ba4a0d2a71eabfca13519229dc16639d3bee`. The original ten local development commits are preserved as ancestors.
- A real network clone matched that commit and verified all 409 frozen artifacts without downloading models/data separately.
- The clone passed 31 additional focused peak-grid/prediction tests, plus `npm ci` and a production build. Its working tree stayed clean after these checks.
- All nine raw GitHub screenshot URLs returned valid WebP images. In the rendered GitHub README, all nine images decoded at 1600×960; the page was visually inspected in Edge.
- Final publication includes a documentation-only follow-up recording these checks; runtime source, model/data/asset bytes and lockfiles are unchanged from the tested implementation commit.

## Limits and cleanup status

The Python run emits 96 pre-existing deprecation, sparse-history and imputation warnings; they did not fail tests. Hardware FPS is machine-specific, not a general performance guarantee. The live internet collection test is intentionally excluded. Historical GLB simplification, image/logo fallbacks and experimental cross-era accuracy limits remain documented.

Automated safety review rejected physical cleanup deletion, so local disk recovery is **0 bytes**. The published tree excludes the verified obsolete artifacts and retains all required runtime files. See `CLEANUP.md` and the per-path `CLEANUP_MANIFEST.csv` for the precise distinction.
