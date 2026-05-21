# Test_coordinate_Transfer — Team Lead Brief

## Project Summary

Build a single-file browser HTML application that transforms 3D measurement data between five coordinate systems used in machine testing. The program reads a CSV file via file picker, performs a least-squares similarity transform calibration (Stage 1), then applies the calibration to transform all test points into every coordinate system (Stage 2). Output is a downloadable CSV.

## Coordinate Systems

### 1. Global (East, North, Elevation)
- Units: **meters**
- Arbitrary origin, measured by UTS survey instrument
- Axis ordering in data: East, North, Elevation

### 2. Machine (X, Y, Z)
- Units: **millimeters**
- Origin: center of machine at local ground level
- X = toward front of machine, Y = toward top of machine, Z = right-hand rule (X × Y = Z, so Z points LEFT when facing front)
- Machine sits on an incline — orientation relative to Global is unknown and must be determined from data

### 3. MachineG (XG, YG, ZG)
- Units: **millimeters**
- Origin: same as Machine (center of machine at ground level)
- YG parallel to Global Elevation (gravity-aligned vertical)
- XG toward front of machine, projected onto horizontal plane (perpendicular to YG), normalized
- ZG = XG × YG (right-hand rule)
- This is a **pure rotation** from Machine coordinates (no scale, no translation)

### 4. MachineP (RP, AP, YP) — Cylindrical
- Origin: same as Machine
- YP = Machine Y (axial direction along machine vertical)
- RP = sqrt(X² + Z²) — radial distance from Y axis in the XZ plane
- AP = atan2(Z, X) in **degrees** — angle from +X, positive toward +Z (left of front)

### 5. MachinePG (RPG, APG, YPG) — Cylindrical
- Origin: same as Machine
- YPG = MachineG YG (gravity-aligned vertical)
- RPG = sqrt(XG² + ZG²) — radial distance from YG axis in the XGZG plane
- APG = atan2(ZG, XG) in **degrees** — angle from +XG, positive toward +ZG (left of front)

## Input CSV Format

See `Test_coordinate_Transfer/Run_Example_1.csv` for the reference file.

### Structure
```
Run_5-4-2026_1                          ← Run label (row 1)
(blank row)
-- Center find UTS DATA --              ← Stage 1 UTS header
point id,,c1,c2,...,c9                   ← Column labels
UTS-J,East,<9 values>                   ← East coordinates (meters)
UTS-J,North,<9 values>                  ← North coordinates (meters)
UTS-J,Elevation,<9 values>              ← Elevation coordinates (meters)
-- Center find Machine DATA --          ← Stage 1 Machine header
Point J Prime,X,<9 values>              ← X coordinates (mm)
Point J Prime,Y,<9 values>              ← Y coordinates (mm)
Point J Prime,Z,<9 values>              ← Z coordinates (mm)
----------,----------,...               ← SEPARATOR (all columns contain "----------")
-- Test Run UTS DATA --                 ← Stage 2 UTS header
point id,,w1,w2,...,wN                  ← Column labels (variable N)
UTS-J,East,<N values>                   ← Stage 2 UTS East
UTS-J,North,<N values>                  ← Stage 2 UTS North
UTS-J,Elevation,<N values>              ← Stage 2 UTS Elevation
-- Test Run Machine DATA --             ← Stage 2 Machine header
<point name>,X,<N values>              ← Each point has 3 rows (X, Y, Z)
<point name>,Y,<N values>
<point name>,Z,<N values>
... (variable number of points)
```

### Parsing Rules
- The separator row (all columns = "----------") divides Stage 1 from Stage 2
- Stage 1 always has exactly one UTS point (UTS-J) and one Machine point (Point J Prime), each with 3 axis rows
- Stage 2 has one UTS point (UTS-J, 3 rows) followed by a variable number of Machine points (each 3 rows: X, Y, Z)
- Column count in Stage 2 may differ from Stage 1
- Point names can contain spaces and numbers
- The run label is in cell A1

## Mathematical Approach

### Stage 1: Calibration (Global → Machine Similarity Transform)

The 9 paired measurements define a **7-parameter similarity transformation**:

```
P_machine = s · R · P_global + t
```

Where:
- `s` = scalar scale factor (expected ≈ 1000, meters→mm)
- `R` = 3×3 rotation matrix (orthogonal, det = +1)
- `t` = 3×1 translation vector (mm)

**Algorithm (SVD-based Umeyama method):**
1. Arrange the 9 Global points as columns of matrix `P` (3×9) and Machine points as `Q` (3×9)
2. Compute centroids: `p̄ = mean(P, axis=1)`, `q̄ = mean(Q, axis=1)`
3. Center: `P' = P - p̄`, `Q' = Q - q̄`
4. Cross-covariance: `H = P' · Q'^T` (3×3)
5. SVD: `H = U · Σ · V^T`
6. Handle reflection: `d = sign(det(V · U^T))`, then `S = diag(1, 1, d)`
7. Rotation: `R = V · S · U^T`
8. Scale: `s = trace(S · Σ) / trace(P'^T · P')` — equivalently `s = trace(R · H) / Σ‖p'_i‖²`
9. Translation: `t = q̄ - s · R · p̄`

**Residual computation:**
For each calibration point i:
- `predicted_i = s · R · P_global_i + t`
- `residual_i = P_machine_i - predicted_i`
- `error_i = ‖residual_i‖` (Euclidean norm, in mm)
- `RMS = sqrt(mean(error_i²))`

### Stage 1: Machine → MachineG Rotation

1. Global Elevation direction in Global coords: `e_elev = [0, 0, 1]` (third axis = Elevation)
2. Transform to Machine frame: `YG_machine = R · e_elev` (just the rotation, normalized — should already be unit length)
3. Machine front direction: `e_front = [1, 0, 0]` (Machine +X)
4. Project onto plane ⊥ YG: `XG_machine = e_front - (e_front · YG_machine) · YG_machine`, then normalize
5. Complete the frame: `ZG_machine = XG_machine × YG_machine`
6. Build rotation matrix `R_MG` whose rows are `XG_machine`, `YG_machine`, `ZG_machine`:
   ```
   P_machineG = R_MG · P_machine
   ```
   (No translation — same origin. No scale — both in mm.)

### Stage 1: Cylindrical Conversions (no matrix, just formulas)

**Machine → MachineP:**
```
RP = sqrt(X² + Z²)
AP = atan2(Z, X) × (180/π)    [degrees, positive = left of front]
YP = Y
```

**MachineG → MachinePG:**
```
RPG = sqrt(XG² + ZG²)
APG = atan2(ZG, XG) × (180/π)  [degrees, positive = left of front]
YPG = YG
```

## Stage 2: Applying Transformations

### UTS-J Translation
Apply the inverse of the Global→Machine similarity transform to bring UTS data into Machine coordinates:
```
UTS-J_machine = s · R · UTS-J_global + t
```
Label these transformed points as "UTS-J" in the Machine coordinate system. Then also compute MachineG, MachineP, and MachinePG representations of UTS-J.

### Machine Points
For each machine-frame point in Stage 2:
- Already in Machine (X, Y, Z) — include as-is
- Compute MachineG: `R_MG · [X, Y, Z]^T`
- Compute MachineP: cylindrical conversion from Machine coords
- Compute MachinePG: cylindrical conversion from MachineG coords

## Output CSV Format

Filename: `<input_filename_without_extension>_OUTPUT.csv`

### Stage 1 Section
```
=== STAGE 1: CALIBRATION ===
Run: <run label>
Calibration Points: 9

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
row1,r11,r12,r13
row2,r21,r22,r23
row3,r31,r32,r33

--- Calibration Residuals ---
Point,Predicted X,Predicted Y,Predicted Z,Actual X,Actual Y,Actual Z,Error X,Error Y,Error Z,Error Magnitude (mm)
c1,...
...
c9,...
RMS Error (mm):,<rms>
```

### Stage 2 Section
```
=== STAGE 2: TRANSFORMED DATA ===
Measurement Columns:,w1,w2,...,wN

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
<name>,X,<values>              (original machine coords)
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
(repeat for each machine point)
```

## Implementation Requirements

### Technology
- **Single HTML file** — no external dependencies except math.js CDN (`https://cdnjs.cloudflare.com/ajax/libs/mathjs/12.4.0/math.min.js`) for SVD and matrix operations
- Browser file picker for input
- CSV download button for output
- Clean, functional UI (no framework needed)

### UI Elements
- File picker with drag-and-drop support
- "Process" button (enabled after file loads)
- Status/progress area showing parsing and computation steps
- Results summary panel: scale factor, RMS error, number of points transformed
- "Download Output CSV" button
- Display the calibration residuals in a table on-screen as well

### Code Quality
- Well-commented, especially the math sections
- Modular functions: `parseCSV()`, `computeSimilarityTransform()`, `computeMachineGRotation()`, `toMachineP()`, `toMachinePG()`, `transformStage2()`, `formatOutputCSV()`
- Error handling for malformed CSV, missing sections, mismatched row counts
- All numeric output to 6 significant figures in the CSV

### Testing / Verification
- After implementing, run against `Run_Example_1.csv`
- Verify scale factor is approximately 1000
- Verify RMS residual is reasonable (expect < 50mm given survey uncertainty)
- Verify that MachineP RP values for Point J Prime calibration data reconstruct to the correct X, Z via `X = RP·cos(AP)`, `Z = RP·sin(AP)`
- Verify MachineG YG **differences** between calibration points match Elevation **differences** × 1000 (the absolute YG values will NOT equal Elevation × 1000 because the machine origin is not at zero elevation — but deltas must agree)

## Acceptance Criteria

1. Parses the example CSV correctly — identifies all Stage 1 and Stage 2 data
2. Similarity transform scale factor is ≈ 1000 (within 0.1%)
3. Calibration RMS error is < 100mm (reasonable for combined survey + machine uncertainty)
4. All five coordinate representations computed for every Stage 2 point
5. Output CSV is well-formatted and human-readable
6. UI shows residual table and summary statistics
7. Program handles variable numbers of Stage 2 points and measurement columns
8. Cylindrical angles are in degrees, not radians
9. No external dependencies except math.js CDN
10. Local git repo initialized and commits made at each milestone (see Version Control below)

## Version Control

- **Initialize a local git repo** in `Test_coordinate_Transfer/` at the start of implementation
- Add a `.gitignore` that excludes any output CSVs (`*_OUTPUT.csv`)
- **Commit at each milestone:** after CSV parser works, after similarity transform works, after all transforms work, after UI is complete, after testing passes
- Use descriptive commit messages (e.g., "feat: CSV parser for Stage 1 and Stage 2 sections")
- Do NOT push to a remote — Paul will set up GitHub later
