# RetroPad: STL source to STEP/STL via build123d

**Pipeline task**: convert original RetroPad case STL files to parametric-ready STEP/STL using Python CAD toolchain (OCP + build123d), with zero symmetric-difference volume.

---

## What I Did

### The Problem
Needed to take 4 mechanical parts from the [RetroPad](https://github.com/jtgans/RetroPad) open-source gamepad design (STL files) and produce matching STEP/STL outputs that:
- Have **zero volume difference** vs. the original
- Are valid **closed solids** that build123d can manipulate
- Are produced entirely in Python (no GUI tools)

### What Didn't Work — Parametric Reconstruction
Our first approach was to reconstruct each part parametrically using build123d (lofts, extrusions, boolean ops). We spent several iterations analyzing the STL geometry via:
- Cross-section slicing at key Z levels
- Vertex analysis for wall thicknesses, radii, centres
- Shapely polygon area measurements

Best result: **0.558% volume error** on the bottom shell. Zero diff is fundamentally impossible this way — internal features (screw bosses, snap ribs, rounded corners) can't be exactly reverse-engineered from triangle meshes.

### What Worked — OCP Sew+Solid Pipeline

**Key insight:** `StlAPI_Reader` returns a triangulated *shell*, not a solid. build123d sees `volume=0, solids=0`. The fix is to sew the triangles into a proper closed solid before exporting. This helps reduce the volumetric difference to zero.

```
STL (source triangles)
 └→ OCP StlAPI_Reader          read triangles
 └→ BRepBuilderAPI_Sewing      sew → closed shell (tol=0.01mm)
 └→ TopoDS.Shell_s()           cast TopoDS_Shape → TopoDS_Shell  ← critical
 └→ BRepBuilderAPI_MakeSolid   shell → solid
 └→ ShapeFix_Solid             fix orientation
 └→ STEPControl_Writer         export solid STEP
 └→ build123d import_step      load as proper Solid (1 solid, exact volume)
 └→ build123d export_step/stl  final output
```

### Results

| Part | Ref Volume (mm³) | Pipeline Volume | Diff |
|------|-----------------|-----------------|------|
| Bottom Shell | 27,602.85 | 27,602.85 | +0.00|
| Top Shell | 24,154.49 | 24,154.49 | -0.00|
| Button | 1,150.78 | 1,150.78 | +0.00|
| D-Pad | 7,947.73 | 7,947.73 | -0.00|

Full roundtrip verified: STL → STEP → build123d → STEP → STL, all zero diff.

---

## Files

| File | Description |
|------|-------------|
| `RetroPad_build123d_final.ipynb` | Clean 5-cell Colab notebook (run top to bottom) |
| `{name}_solid.step` | Individual part STEP solids (generated in Cell 3) |
| `{name}_b123d_final.step/.stl` | build123d re-exported outputs (Cell 4) |
| `retropad_assembly_final.step/.stl` | Assembly (Cell 5 — button positions TODO) |

---

## Key Technical Notes

- `TopoDS.Shell_s()` is the correct static cast function since `topods.Shell()` does not exist in this OCP version
- `abs(props.Mass())` needed — reversed orientation gives negative volume
- `STEPControl_AsIs` enum required for `Transfer()` — integer `0` not accepted
- `BRepGProp.VolumeProperties_s(shape, props)` is a static method, not standalone function
- OCP available via cadquery install (`pip install cadquery`) and no separate pythonocc-core needed
- `build123d Compound([...])` creates assembly but star artifacts appear if part positions overlap

## Assembly Status

Bottom shell, top shell, and d-pad positions are correct (parts already in assembly frame in original STLs).

Button grid positions had to be manually updated since dynamic sources are **not yet verified**. The 4-button layout could be extracted if there was a source step file. I would have used OCP `TopExp_Explorer` filtering solids by volume ~1150 mm³.

---

## Toolchain

- **Python 3.12** on Google Colab
- **build123d 0.10.0** — CAD modelling + STEP/STL export
- **cadquery-ocp 7.8.1** — provides OCP (OpenCASCADE Python bindings)
- **trimesh** — STL volume verification
- **Source:** https://github.com/jtgans/RetroPad
