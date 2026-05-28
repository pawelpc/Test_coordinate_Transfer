# Coordinate Transfer Tool

A single-file browser application that transforms 3D machine test data
between five coordinate systems used in machine testing. Drop in a CSV,
click **Process**, download the transformed CSV.

- **Live app:** https://pawelpc.github.io/Test_coordinate_Transfer/
- **Source:** [`Coordinate_Transfer.html`](Coordinate_Transfer.html) — one self-contained HTML file, only external dependency is the math.js CDN.

## Quick start

1. Open the live app (or open `Coordinate_Transfer.html` directly in any modern browser — no server needed).
2. Drag a CSV onto the drop zone, or click to pick a file.
3. Click **Process**.
4. Review the summary panel and the calibration residuals table.
5. Click **Download Output CSV** — file is named `<input_basename>_OUTPUT.csv`.

## Coordinate Systems

| # | Name | Axes | Units | Origin | Notes |
|---|------|------|-------|--------|-------|
| 1 | **Global** | East, North, Elevation | meters | arbitrary (UTS site origin) | Survey frame; elevation is the **third** axis. |
| 2 | **Machine** | X, Y, Z | mm | center of machine at ground level | X = front, Y = top, Z = X × Y (points left when facing front). Machine sits on an incline — orientation vs Global is determined from data. |
| 3 | **MachineG** | XG, YG, ZG | mm | same as Machine | YG = gravity-aligned vertical (parallel to Global Elevation). XG = machine front projected onto the horizontal plane. ZG = XG × YG. Pure rotation from Machine. |
| 4 | **MachineP** | RP, AP, YP | mm, **degrees**, mm | same as Machine | Cylindrical view of Machine. YP = Y. RP = √(X²+Z²). AP = atan2(Z, X). |
| 5 | **MachinePG** | RPG, APG, YPG | mm, **degrees**, mm | same as Machine | Cylindrical view of MachineG. YPG = YG. RPG = √(XG²+ZG²). APG = atan2(ZG, XG). |

---

## Input CSV format

The file is split into a **Stage 1** (calibration) block and a **Stage 2**
(test data) block, separated by a row of `----------` values.

### Top-level layout

```
<run label>                                 ← row 1, cell A
<blank row>
-- Center find UTS DATA --                  ← Stage 1 UTS header
point id,,c1,c2,...,c9                      ← column labels (col B blank, labels start col C)
UTS-J,East,<9 values>                       ← meters
UTS-J,North,<9 values>                      ← meters
UTS-J,Elevation,<9 values>                  ← meters
-- Center find Machine DATA --              ← Stage 1 Machine header
Point J Prime,X,<9 values>                  ← mm
Point J Prime,Y,<9 values>
Point J Prime,Z,<9 values>
----------,----------,...,----------        ← SEPARATOR: every non-empty cell == "----------"
-- Test Run UTS DATA --                     ← Stage 2 UTS header
point id,,w1,w2,...,wN                      ← column labels (variable N)
UTS-J,East,<N values>
UTS-J,North,<N values>
UTS-J,Elevation,<N values>
-- Test Run Machine DATA --                 ← Stage 2 Machine header
<point name>,X,<N values>                   ← each point has 3 rows (X, Y, Z)
<point name>,Y,<N values>
<point name>,Z,<N values>
... (variable number of machine points)
```

### Parsing rules enforced by the app

- The **separator row** is identified as any row whose non-empty cells are
  all the literal string `----------`. It must appear exactly once.
- **Stage 1** must contain exactly one UTS point (UTS-J, 3 axis rows) and
  one Machine point (Point J Prime, 3 axis rows).
- **Stage 2** must contain exactly one UTS point (UTS-J, 3 axis rows) and
  one or more Machine points (each with 3 axis rows: X, Y, Z).
- The Stage 1 and Stage 2 column counts may differ.
- Point names may contain spaces and digits (e.g. `Boom Vertex 1`).
- The run label is whatever appears in row 1, column A.
- Trailing empty cells in any row are tolerated.

A malformed file produces a red error in the status panel; processing is
blocked until a valid file loads.

---

## Output CSV format

Filename: `<input_basename>_OUTPUT.csv`. Numbers are written to **6
significant figures**. Line endings are CRLF.

### Stage 1 section

```
=== STAGE 1: CALIBRATION ===
Run:,<run label>
Calibration Points:,9

--- Global to Machine Similarity Transform ---
Scale Factor:,<s>
Units:,meters to mm

Rotation Matrix R:
,col1,col2,col3
row1,r11,r12,r13
row2,r21,r22,r23
row3,r31,r32,r33

Translation Vector t (mm):
tX,<tx>
tY,<ty>
tZ,<tz>

--- Machine to MachineG Rotation ---
Rotation Matrix R_MG:
,col1,col2,col3
row1, ...
row2, ...
row3, ...

--- Calibration Residuals ---
Point,Predicted X,Predicted Y,Predicted Z,Actual X,Actual Y,Actual Z,Error X,Error Y,Error Z,Error Magnitude (mm)
c1, ...
...
c9, ...
RMS Error (mm):,<rms>
```

### Stage 2 section

```
=== STAGE 2: TRANSFORMED DATA ===
Measurement Columns:,,w1,w2,...,wN          ← col B is intentionally blank so the
                                              labels line up with values in the data
                                              rows below (which use cols A,B for
                                              point name and axis label)

--- UTS-J (transformed from Global to Machine) ---
UTS-J,X,<values>
UTS-J,Y,<values>
UTS-J,Z,<values>
UTS-J,XG,<values>
UTS-J,YG,<values>
UTS-J,ZG,<values>
UTS-J,RP,<values>
UTS-J,AP,<values>
UTS-J,YP,<values>
UTS-J,RPG,<values>
UTS-J,APG,<values>
UTS-J,YPG,<values>

--- <Point Name> ---
<name>,X,<values>      ← original machine X/Y/Z, included verbatim
<name>,Y,<values>
<name>,Z,<values>
<name>,XG,<values>
<name>,YG,<values>
<name>,ZG,<values>
<name>,RP,<values>
<name>,AP,<values>
<name>,YP,<values>
<name>,RPG,<values>
<name>,APG,<values>
<name>,YPG,<values>
(repeated for every Stage 2 machine point)
```

Every Stage 2 point therefore produces **12 rows** of N values — three rows
for each of the four non-Global representations (Machine, MachineG, MachineP,
MachinePG).

---

## Major operations inside the code

All operations live in `Coordinate_Transfer.html` (one file, one `<script>`).
The pipeline executes top-to-bottom each time you click **Process**.

### 1. CSV parsing — `parseCSV(text)`

- `splitCsvLine` tokenizes each row (supports `"`-quoted cells).
- `isSeparatorRow` and `isBlankRow` find the Stage 1 / Stage 2 boundary.
- `parseStageBlock` locates the UTS DATA header, Machine DATA header, and
  column-label row inside each stage.
- `parsePointRows` groups consecutive axis-labelled rows by point name and
  hands each group to `finalisePoint`, which validates that exactly 3 axes
  are present and normalises keys (`Elevation` → `elev`, etc.).
- Output: a structured object with `runLabel`, `stage1`, and `stage2` blocks,
  each containing typed arrays of measurement values.

### 2. Stage 1 similarity transform (Umeyama / SVD) — `computeSimilarityTransform`

Implements the 7-parameter similarity transform
`P_machine ≈ s · R · P_global + t`:

1. Compute centroids `p̄`, `q̄` of source (Global) and target (Machine).
2. Center both point sets and accumulate `Σ‖p'ᵢ‖²` (used for the scale).
3. Build the 3×3 cross-covariance `H = P' · Q'ᵀ` (per the brief; this is the
   transpose of Umeyama's convention, so signs are handled accordingly).
4. SVD `H = U Σ Vᵀ` via `svdViaEigs` (eigendecomposition of `HᵀH`).
   math.js 12.4.0 does not expose `math.svd()`, so we implement it in-file —
   correct and stable for a 3×3 cross-covariance.
5. Reflection correction: `d = sign(det(V · Uᵀ))`, then `S = diag(1, 1, d)`.
6. Rotation `R = V · S · Uᵀ`. Determinant is +1 by construction.
7. Scale `s = trace(diag(S) · σ) / Σ‖p'ᵢ‖²`.
8. Translation `t = q̄ − s · R · p̄`.

### 3. Residuals — `computeResiduals`

For each calibration point: predict `s·R·pᵢ + t`, subtract from actual,
record the per-axis errors and the Euclidean magnitude. RMS is `sqrt(mean(errMag²))`.

### 4. Machine → MachineG rotation — `computeMachineGRotation(R)`

Derives `R_MG` from the similarity transform's `R` so that
`P_machineG = R_MG · P_machine`:

1. `YG_machine = R · [0, 0, 1]` — the Global Elevation direction expressed
   in the Machine frame. Renormalised defensively.
2. `XG_machine = e_front − (e_front · YG) · YG`, normalised — the Machine
   front direction projected onto the plane perpendicular to YG.
3. `ZG_machine = XG × YG`.
4. `R_MG` has those three vectors as its rows. No translation (same origin),
   no scale (both in mm).

### 5. Cylindrical conversions — `toMachineP`, `toMachinePG`

Pure scalar formulas. Angles returned in **degrees**, positive toward +Z
(or +ZG), measured from +X (or +XG).

### 6. Stage 2 pipeline — `transformStage2(stage2, sim, R_MG)`

For every measurement column:

- **UTS-J:** apply the similarity transform meters → mm to bring the UTS
  measurement into the Machine frame, then compute MachineG / MachineP /
  MachinePG from there.
- **Each Machine point:** Machine X/Y/Z is already known; apply `R_MG` to
  get MachineG, then cylindrical conversions for MachineP and MachinePG.

`fillAllSystemsForColumn` is the single helper that, given a Machine-frame
(x, y, z), writes all 12 axis values into one column of the result.

### 7. Output CSV formatting — `formatOutputCSV(parsed, results)`

- `fmt` formats numbers to 6 significant figures, trimming trailing zeros.
- `csvCell` quotes only when a cell contains `,`, `"`, or a newline.
- The Stage 2 column-header row uses a blank cell B so labels stay in
  vertical alignment with the per-point value cells below.

### 8. UI — `loadFile`, `runProcessing`, `renderSummary`, `renderResiduals`, `downloadOutput`

- File picker + drag-and-drop wired into the same `loadFile` path.
- Status panel logs each pipeline step with timestamps.
- Summary panel: run label, calibration point count, scale factor, RMS,
  max residual, Stage 2 column/point counts.
- Residuals table: per-point predicted/actual/error. **Any row whose error
  magnitude exceeds 3× the RMS is flagged red** as a likely measurement
  outlier (this catches things like a UTS angular mis-read on one calibration
  point, which can otherwise hide inside a still-numerically-finite RMS).
- Download button serialises the prepared CSV via a `Blob` URL.

---

## Verification (against `Run_Example_1.csv`)

| Metric | Result | Spec |
|---|---|---|
| Scale factor | 997.66 | ≈ 1000 |
| RMS residual | 10.5 mm | < 100 mm |
| Max residual | 21.0 mm | — |
| det(R) | 1.0000 | +1 |
| ΔYG vs ΔElevation·1000 (calibration) | < 1.6 mm | should match |
| Cylindrical inversion error | ~1e-13 mm | exact |

Algorithm correctness was additionally verified on synthetic data with a
known `(s, R, t)`: recovery to machine epsilon (R within 3e-16, t within 1e-13).

## Files in this repository

```
Coordinate_Transfer.html     the application (open in any browser)
index.html                   redirect shim → Coordinate_Transfer.html (for GitHub Pages root)
Run_Example_1.csv            example input
README.md                    this file
Test_coordinate_Transfer.md  original project description
_team-lead-brief.md          full spec the build was worked from
_team-lead-instructions.md   working instructions for the build
_team-lead-report.md         closeout report
.gitignore                   excludes *_OUTPUT.csv
```
