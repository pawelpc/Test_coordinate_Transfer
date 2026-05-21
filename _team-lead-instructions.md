# Team Lead Instructions — Test_coordinate_Transfer

## Your Role
You are building a browser-based coordinate transformation tool for machine test data. Read `_team-lead-brief.md` for the full specification.

## Startup
1. Read `_team-lead-brief.md` thoroughly
2. Read the example input file: `../Test_coordinate_Transfer/Run_Example_1.csv`
3. Plan your implementation before writing code

## Version Control
Initialize a local git repo in this project folder before writing code. Commit at each milestone below. Descriptive messages. Do NOT push to any remote — Paul will handle GitHub later. Add a `.gitignore` excluding `*_OUTPUT.csv`.

## Implementation Order
1. **Git init + .gitignore** — set up version control first.
2. **CSV Parser** — get the data structure right first. Parse Stage 1 and Stage 2 sections. Log parsed point counts to console for verification. **Commit.**
3. **Similarity Transform** — implement the Umeyama SVD method. Verify scale ≈ 1000 and residuals are reasonable. **Commit.**
4. **Machine→MachineG rotation** — derive from the similarity transform's R matrix. **Commit.**
5. **Cylindrical conversions** — MachineP and MachinePG. **Commit.**
6. **Stage 2 pipeline** — transform UTS-J, then all machine points. **Commit.**
7. **Output CSV formatter** — human-readable blocks per the brief. **Commit.**
8. **UI** — file picker, process button, results display, download button. **Commit.**
9. **Testing** — run against the example CSV, verify all acceptance criteria. **Commit as v1.0 tag.**

## Key Technical Notes
- Use math.js for SVD (`math.svd()`), matrix multiplication, and cross products
- UTS data is in **meters**, Machine data is in **millimeters** — the similarity transform absorbs this as a scale factor
- Global axis ordering is (East, North, Elevation) — Elevation is the **third** axis, index 2
- The Machine Y axis is NOT aligned with Elevation — the machine is inclined
- For MachineG derivation: the Elevation direction in Global coords is `[0, 0, 1]`, rotate it by R to get YG direction in Machine frame

## Output Location
Save the HTML file as: `../Test_coordinate_Transfer/Coordinate_Transfer.html`

## What NOT to Do
- Don't use Node.js or npm — this is a browser-only HTML file
- Don't add unnecessary frameworks (React, Vue, etc.)
- Don't skip the residual computation — it's essential for verifying the fit
- Don't hard-code the number of Stage 2 points or measurement columns
- Don't push to GitHub — local git only
- Don't assume MachineG YG equals Elevation × 1000 — the machine origin is offset from zero elevation. Only **differences** in YG should match differences in Elevation × 1000
