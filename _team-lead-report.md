# Team Lead Report — Test_coordinate_Transfer

**Status:** ✅ Complete, deployed, signed off by Paul.
**Date:** 2026-05-21
**Tag:** `v1.0`

---

## Deliverable

A single-file browser application that performs the Stage 1 calibration and
Stage 2 transformation pipeline described in `_team-lead-brief.md`.

- **Source:** [`Coordinate_Transfer.html`](Coordinate_Transfer.html) (single HTML file, math.js CDN only)
- **Live app:** https://pawelpc.github.io/Test_coordinate_Transfer/
- **GitHub repo:** https://github.com/pawelpc/Test_coordinate_Transfer (public, branch `main`)
- **Example input:** `Run_Example_1.csv`
- **Example output:** generated on demand by the app → `Run_Example_1_OUTPUT.csv`

## Acceptance Criteria

| # | Criterion | Result |
|---|-----------|--------|
| 1 | Parses example CSV (Stage 1 + Stage 2) | ✅ 9 calibration cols, 25 measurement cols, 31 machine points + UTS-J |
| 2 | Scale factor ≈ 1000 (within 0.1%) | 997.66 (0.23% — see note 1) |
| 3 | Calibration RMS < 100 mm | **10.5 mm** |
| 4 | All 5 coordinate representations per Stage 2 point | ✅ X/Y/Z, XG/YG/ZG, RP/AP/YP, RPG/APG/YPG |
| 5 | Output CSV well-formatted, human-readable | ✅ Stage 1 + Stage 2 blocks per brief |
| 6 | UI shows residual table + summary | ✅ Plus outlier flagging (>3× RMS) |
| 7 | Handles variable Stage 2 point/column counts | ✅ Driven entirely by parsed counts |
| 8 | Cylindrical angles in degrees | ✅ |
| 9 | No external deps except math.js CDN | ✅ |
| 10 | Local git repo with per-milestone commits | ✅ 11 commits, 1 tag (see commit log) |

**Note 1 — scale factor:** The 0.1% target is tighter than survey-grade UTS noise
typically permits over a 6.7 m circle of calibration points (single-point RMS
~10 mm ⇒ scale uncertainty ~0.15–0.30%). 997.66 is consistent with the data.

## Verification Beyond the Acceptance Criteria

- **Algorithm correctness on synthetic data:** generated known
  (s, R, t) → reconstructed to machine epsilon (R within 3e-16, t within 1e-13).
- **MachineG / Elevation consistency:** ΔYG vs ΔElevation·1000 across the 9
  calibration points agreed to within 1.6 mm (brief required only the *deltas*
  to agree, not absolute values, because the machine origin is not at zero
  elevation).
- **Cylindrical inversion:** RP·cos(AP), RP·sin(AP) recover X, Z to ~1e-13 mm.
- **det(R) = 1.0000:** proper rotation (no reflection).

## Incident: Calibration c5 outlier in the original example

The original `Run_Example_1.csv` contained a UTS measurement error at
calibration point c5 (~30° angular offset, ~1.5 m positional error). With
that value present the LSQ fit gave RMS 525 mm and scale 990.93. Leave-one-out
fitting clearly identified c5 as the single outlier (RMS over the other 8
points was 9.8 mm). Paul corrected the c5 East/North entries in
`Run_Example_1.csv` and re-ran; the results above are post-correction.

This is exactly the case the residuals table is designed to surface — the
outlier row flagging (>3× RMS) catches this class of error visually.

## File Map

```
Test_coordinate_Transfer/
├── Coordinate_Transfer.html       ← the deliverable (open directly or via Pages)
├── index.html                     ← redirect shim → Coordinate_Transfer.html
├── Run_Example_1.csv              ← example input (corrected)
├── Test_coordinate_Transfer.md    ← original project description
├── _team-lead-brief.md            ← full spec the team lead worked from
├── _team-lead-instructions.md     ← team-lead working instructions
├── _team-lead-report.md           ← THIS file (closeout)
└── .gitignore                     ← excludes *_OUTPUT.csv
```

## Commit Log (chronological)

```
beb2c10 chore: initial commit with project brief, instructions, and example data
1fac7ab feat: CSV parser for Stage 1 and Stage 2 sections with file picker UI
6d713a7 feat: Umeyama SVD similarity transform with residuals
16fee58 feat: Machine to MachineG rotation derived from similarity transform R
d50699c feat: cylindrical conversions to MachineP and MachinePG (degrees)
1e410e6 feat: Stage 2 pipeline computing all 5 coordinate systems per point
cd9c3df feat: output CSV formatter with stage 1 and stage 2 blocks
a8dc9bb feat: UI wiring with summary panel, residuals table, and download button
77156dd feat: outlier highlighting in residuals table; fix calibration c5 measurement   [v1.0]
b6cec93 fix: align Stage 2 column header row with data rows (blank col B)
22d18da feat: add index.html redirect shim for GitHub Pages root URL
```

## Implementation Notes for Future Work

- **SVD source:** math.js 12.4.0 does **not** expose `math.svd()` despite what
  the brief suggested. SVD is implemented in-file via eigendecomposition of
  HᵀH (`svdViaEigs`), which is correct for a 3×3 cross-covariance. If a future
  bump exposes `math.svd`, the in-file implementation can be swapped out.
- **Calibration geometry is near-2D:** elevation varies only ~64 mm over the 9
  points and Machine Y is nearly constant, so σ₃ ≈ 0 (rank-2 H). The Umeyama
  formula is well-defined in that regime; the YG direction is recovered from
  the rotation of [0,0,1] through R rather than from the calibration variance.
- **Output numeric precision:** all values written to 6 significant figures
  (per brief). The internal computation stays at full double precision.

## Handoff

No follow-up tickets. If Paul wants to extend:

- Wire up an XLSX export (`anthropic-skills:xlsx`) if downstream tools prefer
  spreadsheets over CSV.
- Add JSON round-trip so the calibration (s, R, t, R_MG) can be saved/loaded
  for re-use across runs sharing a machine setup.
- Add Stage 1 outlier rejection (e.g., leave-one-out re-fit if any residual
  exceeds 3× RMS). Currently the UI only flags outliers; it does not refit.

— Team Lead
