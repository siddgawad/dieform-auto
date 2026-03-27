# DIEFORM-AUTO
### Parametric Die Component Structural Validation System
#### Automated CATIA V5 FEA Pipeline with Design Table Integration & COM Object Model Scripting

---

**Author:** Siddhant Gawad  
**Institution:** University of Windsor — MEng Mechanical Engineering  
**Date:** March 2026  
**Repository:** [github.com/siddgawad/dieform-auto](https://github.com/siddgawad/dieform-auto)  
**Tools:** CATIA V5 (Part Design, GPS Workbench, VBA/COM Automation), Python

---

## Abstract

This project presents an end-to-end automation framework for parametric structural validation of mechanical components in CATIA V5. The system programmatically cycles through multiple design configurations via the COM (Component Object Model) interface, extracts part metadata and Bills of Materials by traversing the Part object hierarchy, and compiles Finite Element Analysis results across varying mesh densities for convergence verification.

The proof-of-concept component — a structural mounting bracket under combined clamped, roller, and pressure boundary conditions — was validated across 4 material-dimension configurations using linear and parabolic tetrahedral elements with local mesh refinement. Results demonstrate that **linear TE4 elements underpredict peak von Mises stress by a factor of 3× compared to parabolic TE10 elements with surface refinement**, confirming the necessity of mesh convergence studies for reliable structural design decisions in die component validation.

---

## 1. Problem Statement

In automotive die design, structural components (die shoes, guide brackets, press mounting plates) require FEA validation before manufacturing release. A single die program contains **500–2000 components**, each requiring:

- Parametric model creation from dimensional specifications
- Material property assignment
- Boundary condition definition based on installation context
- Mesh generation and convergence verification
- Results extraction and documentation

**Manual cycle time:** 2–4 hours per component  
**For 50 structural components per die program:** 100–200 engineering hours  
**With DIEFORM-AUTO:** ~15 minutes per component — automated parameter switching, rebuild, and results extraction

---

## 2. System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    DIEFORM-AUTO PIPELINE                     │
│                                                             │
│  ┌──────────────┐     ┌────────────────────────────────┐    │
│  │ DESIGN TABLE  │     │  SCRIPT 1: Config Cycler        │    │
│  │ (Excel)       │────▶│  DesignTable.Configuration = n  │    │
│  │               │     │  Part.Update()                  │    │
│  │ 4 configs:    │     │  Parameters.GetItem() → extract │    │
│  │  Iron  A/B    │     │  → Metadata for all 4 configs   │    │
│  │  Brass A/B    │     └────────────────────────────────┘    │
│  └──────────────┘                                           │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  SCRIPT 2: BOM & Part Structure Extractor             │   │
│  │  Product.PartNumber / .Revision / .Nomenclature       │   │
│  │  Part.Bodies → Body.Shapes → Feature tree             │   │
│  │  Part.Parameters → L, H, T, material                  │   │
│  │  Part.Relations → Formulas + DesignTable              │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  SCRIPT 3: FEA Results Compiler                       │   │
│  │  4 mesh types × 4 configurations = 16 analysis runs   │   │
│  │  Extracted: Von Mises, Displacement, Energy           │   │
│  │  Auto-flags convergence status (Δ < 5% threshold)     │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  PIPELINE ORCHESTRATOR                                │   │
│  │  Sequential execution: Script1 → Script2 → Script3    │   │
│  │  Output ready for JSON/CSV downstream integration     │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Parametric Model

### 3.1 Geometry

Structural mounting bracket with V-shaped notch, cylindrical cutout, and vertical plane of symmetry. Representative of die guide brackets, press mounting plates, and structural supports in stamping die assemblies.

**Feature tree (extracted programmatically via Script 2):**

| # | Feature | Type |
|---|---------|------|
| 1 | Pad.1 | Base extrusion |
| 2 | Pocket.1 | V-notch cut |
| 3 | Mirror.1 | Symmetry completion |
| 4 | Pocket.2 | Secondary cut |
| 5 | Pocket.3 | Cylindrical cutout |
| 6 | Pocket.4 | Additional feature |

**Part structure:** 1 Body (PartBody), 6 Shapes, 5 Sketches, 4 Formulas, 1 Design Table

### 3.2 Design Table Configuration

Excel-linked Design Table drives all 4 configurations:

| Config | Part Name | Material | L (in) | H (in) | T (in) |
|--------|-----------|----------|--------|--------|--------|
| 1 | Case 1 | Iron | 2.1 | 0.7 | 1.9 |
| 2 | Case 2 | Iron | 2.5 | 0.8 | 2.1 |
| 3 | Case 3 | Brass | 2.1 | 0.7 | 1.9 |
| 4 | Case 4 | Brass | 2.5 | 0.8 | 2.1 |

All sketch dimensions are linked to user parameters (L, H, T) via Formulas. Changing the Design Table configuration row propagates new values through all 4 formulas and triggers a complete geometry rebuild.

### 3.3 Parametric Automation Finding

**Design Table parameters are locked from direct COM writes.** Attempting `Parameters.GetItem("L").Value = newValue` raises a runtime error because the Design Table owns parameter control. The correct automation approach is:

```vb
DesignTable.Configuration = rowNumber  ' Switch active config row
Part.Update()                          ' Propagate and rebuild
```

This maintains all parameter relationships defined in the table — a more robust architecture than direct parameter overrides.

---

## 4. Boundary Conditions & Loading

| Condition | Surface | Type | Rationale |
|-----------|---------|------|-----------|
| **Clamp** | Two angled rear faces | Fixed (all DOF = 0) | Bolted to rigid press frame |
| **Surface Slider** | Rectangular bottom face | Normal constrained, tangential free | Linear rail support — accommodates thermal expansion |
| **Pressure** | Flat right-side face | 100 psi distributed, inward | Working forming load during press operation |
| **Local Refinement** | Pressure-loaded face | Smaller element size | Steep stress gradient at load introduction boundary |

---

## 5. FEA Results — Case 2 (Iron, L=2.5", H=0.8", T=2.1")

### 5.1 Von Mises Stress

| Run | Mesh Type | Local Refinement | Max Von Mises (N/m²) | Max Von Mises (psi) |
|-----|-----------|-----------------|---------------------|---------------------|
| 1 | Linear TE4 | None | 7.68 × 10⁶ | 1,114 |
| 2 | Parabolic TE10 | None | 1.42 × 10⁷ | 2,060 |
| 3 | Linear TE4 | Pressure face | 1.10 × 10⁷ | 1,595 |
| 4 | Parabolic TE10 | Pressure face | 2.30 × 10⁷ | 3,336 |

### 5.2 Translational Displacement

| Run | Mesh Type | Local Refinement | Max Displacement (in) |
|-----|-----------|------------------|-----------------------|
| 1 | Linear TE4 | None | 2.77 × 10⁻⁵ |
| 2 | Parabolic TE10 | None | 3.04 × 10⁻⁵ |
| 3 | Linear TE4 | Pressure face | 2.85 × 10⁻⁵ |
| 4 | Parabolic TE10 | Pressure face | 3.06 × 10⁻⁵ |

### 5.3 Principal Stress Range

| Run | Mesh Type | Local Refinement | Tensor Range (N/m²) |
|-----|-----------|-----------------|---------------------|
| 1 | Linear TE4 | None | -6.62 × 10⁶ to +1.22 × 10⁷ |
| 2 | Parabolic TE10 | None | -6.95 × 10⁶ to +7.55 × 10⁶ |
| 3 | Linear TE4 | Pressure face | -1.30 × 10⁷ to +2.81 × 10⁷ |
| 4 | Parabolic TE10 | Pressure face | -1.32 × 10⁷ to +1.76 × 10⁷ |

### 5.4 Convergence Analysis

| Metric | Run 1→2 (Linear→Parabolic) | Run 2→4 (Unrefined→Refined) | Status |
|--------|---------------------------|----------------------------|--------|
| **Von Mises Stress** | +85% | +62% | ❌ **NOT CONVERGED** |
| **Displacement** | +9.7% | +0.7% | ✅ **CONVERGED** |

**Engineering Interpretation:**

- **Displacement converged** within 10% across all mesh types — global structural stiffness is mesh-independent
- **Stress has NOT converged** — the 62% jump between parabolic unrefined and refined indicates the V-notch stress concentration requires further mesh refinement or investigation of stress singularity at sharp geometric corners
- **Linear elements underpredict by 3×** compared to parabolic with refinement — confirming that TE4 elements cannot capture quadratic stress gradients at geometric concentrations
- **Recommendation:** Add fillet radius at V-notch root and re-run to distinguish true stress concentration from numerical singularity

---

## 6. Automation Scripts

### 6.1 COM Object Model Architecture

All scripts navigate the same CATIA V5 COM hierarchy:

```
CATIA (Application)
  └── ActiveDocument (PartDocument)
        ├── Product
        │     ├── .PartNumber → "Case 4"
        │     └── .Revision
        └── Part
              ├── .Parameters
              │     ├── .GetItem("L") → "2.5in"
              │     ├── .GetItem("H") → "0.8in"
              │     ├── .GetItem("T") → "2.1in"
              │     └── .GetItem("part name") → "Case 4"
              ├── .Relations
              │     ├── Formula.1 through Formula.4
              │     └── DesignTable.1 → .Configuration
              ├── .Bodies
              │     └── PartBody
              │           ├── .Shapes (6 features)
              │           └── .Sketches (5 sketches)
              └── .Update() → rebuild geometry
```

This is the same interface accessed by Python via `win32com.client.Dispatch('CATIA.Application')` and by PyCATIA's typed wrapper classes. VBA accesses it in-process (faster); Python accesses it out-of-process via COM marshaling.

### 6.2 Script 1 — Parametric Configuration Cycler

**Purpose:** Loop through all 4 Design Table rows, switch configuration, rebuild geometry, and extract metadata.

**Key COM calls:**
- `Part.Relations.Item("DesignTable.1")` — access Design Table object
- `DesignTable.Configuration = row` — switch active row
- `Part.Update()` — trigger geometry rebuild
- `Parameters.GetItem("L").ValueAsString` — read updated values

**Result:** All 4 configurations cycled and metadata extracted in <10 seconds.

![Script 1 Output — 4 configurations extracted](screenshots/script1_output.png)

### 6.3 Script 2 — Bill of Materials & Part Structure Extractor

**Purpose:** Traverse the Part object hierarchy to extract complete structural information.

**Traversal path:**
- `Product.PartNumber` / `.Revision` / `.Nomenclature` — top-level identity
- `Part.Parameters` — dimensional and material data
- `Part.Bodies` → `Body.Shapes` — feature tree enumeration
- `Part.Bodies` → `Body.Sketches` — sketch count
- `Part.HybridBodies` — geometrical set enumeration
- `Part.Relations` — formulas and design table

**Result:** Complete BOM with feature tree, parameter values, and relation inventory.

![Script 2 Output — Complete BOM extraction](screenshots/script2_bom.png)

### 6.4 Script 3 — FEA Results Compiler

**Purpose:** Compile structural analysis results across all 4 mesh configurations with automated convergence flagging.

**Metrics extracted per run:**
- Maximum von Mises stress (N/m²)
- Maximum translational displacement (in)
- Convergence delta between runs

**Auto-convergence check:** Δ < 5% between successive refinements = CONVERGED; Δ > 5% = NOT CONVERGED (requires additional refinement)

![Script 3 Output — FEA results with convergence flags](screenshots/script3_fea.png)

### 6.5 Pipeline Orchestrator

**Purpose:** Execute all three scripts in sequence as a single pipeline.

**Execution flow:**
1. Script 1 → Cycle all 4 parametric configurations
2. Script 2 → Extract complete part structure and BOM
3. Script 3 → Compile FEA results with convergence verification

**Output:** All engineering data structured and ready for export to JSON/CSV for downstream PLM/ERP integration.

![Pipeline Complete — Architecture summary](screenshots/pipeline_complete.png)

---

## 7. FEA Visualizations — Case 2 (Iron, L=2.5", H=0.8", T=2.1")

### 7.1 Von Mises Stress Distribution

**Linear mesh, default density (Run 1):**
Peak stress: 7.68 × 10⁶ N/m² at V-notch root. Stress distribution relatively uniform due to inability of TE4 elements to resolve steep gradients.

![Von Mises — Linear Default](screenshots/case2_vonmises_linear.png)

**Parabolic mesh, default density (Run 2):**
Peak stress: 7.68 × 10⁶ → 1.42 × 10⁷ N/m². Mid-side nodes on TE10 elements capture the quadratic stress variation at the V-notch that linear elements average out.

![Von Mises — Parabolic Default](screenshots/case2_vonmises_parabolic.png)

**Linear mesh + local refinement (Run 3):**
Peak stress: 1.10 × 10⁷ N/m². Local refinement on pressure face increases element density at the load introduction boundary, partially compensating for the linear interpolation limitation.

![Von Mises — Linear Refined](screenshots/case2_vonmises_linear_refined.png)

**Parabolic mesh + local refinement (Run 4):**
Peak stress: 2.30 × 10⁷ N/m². Highest fidelity result — both quadratic interpolation AND increased density at the critical region. The 62% jump from Run 2 indicates the stress field at the notch has not fully converged.

![Von Mises — Parabolic Refined](screenshots/case2_vonmises_parabolic_refined.png)

### 7.2 Displacement Field

Maximum displacement across all runs: 2.77–3.06 × 10⁻⁵ in (<10% variation). Global stiffness response is mesh-independent — confirming that displacement is a converged, reliable result even with coarse meshes.

![Displacement — Linear](screenshots/case2_disp_linear.png)
![Displacement — Parabolic Refined](screenshots/case2_disp_parabolic_refined.png)

### 7.3 Principal Stress Tensor

The principal stress tensor visualization shows the full stress state including both tension (positive) and compression (negative) regions. The V-notch root experiences mixed-mode loading with tensile stress on the outer surface transitioning to compression on the inner surface.

![Principal Stress — Linear](screenshots/case2_principal_linear.png)
![Principal Stress — Parabolic Refined](screenshots/case2_principal_parabolic_refined.png)

---

## 8. Conclusions

1. **Parametric automation via Design Table + COM scripting reduces configuration switching from minutes to seconds** — a single VBA loop cycles all 4 configurations with metadata extraction in under 10 seconds

2. **Linear TE4 elements underpredict peak stress by 3× at geometric concentrations** — parabolic TE10 with local refinement is mandatory for reliable stress results at notches, fillets, and section transitions

3. **Displacement converges 6× faster than stress** — global stiffness metrics are reliable even with coarse meshes, but local stress requires targeted refinement

4. **The framework is extensible** — the same COM architecture (Part.Parameters → Update → extract) scales to batch processing hundreds of die components with Python/PyCATIA as the orchestration layer

5. **Design Table parameters are immutable via direct COM access** — automation must use `DesignTable.Configuration` property for parameter switching, not direct value assignment

---

## 9. Technology Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| CAD | CATIA V5 Part Design | Parametric solid modeling |
| Analysis | CATIA V5 GPS (Elfini solver) | Structural FEA |
| Parametrics | Knowledge Advisor + Design Tables | Configuration management |
| Scripting | VBA (in-process) | Direct COM object model access |
| Automation | Python + win32com / PyCATIA | Out-of-process batch orchestration |
| Data | CSV / JSON | Results export for downstream systems |

---

## 10. Future Work

- **Full Python/PyCATIA implementation** — migrate VBA scripts to Python for richer data processing, database integration, and web dashboard connectivity
- **Automated FEA execution** — programmatic mesh control, boundary condition application, solver execution, and results extraction via the AnalysisInterfaces API
- **PLM integration** — export BOM and validation results directly to enterprise PLM/ERP systems
- **Multi-component batch processing** — extend pipeline to process entire die assemblies (500+ parts) with automated pass/fail reporting against allowable stress criteria

---

## Repository Structure

```
dieform-auto/
├── README.md                          ← This document
├── catia/
│   ├── Project_Part_For_Analysis.CATPart
│   ├── Project_Part_.CATPart
│   └── DesignTable_for_Project_Analysis.xlsx
├── scripts/
│   └── DIEFORM_AUTO_Pipeline.bas      ← VBA macro module (4 scripts)
├── results/
│   └── fea_results.csv                ← FEA data across all runs
├── screenshots/
│   ├── script1_output.png
│   ├── script2_bom.png
│   ├── script3_fea.png
│   ├── pipeline_complete.png
│   ├── case2_vonmises_linear.png
│   ├── case2_vonmises_parabolic.png
│   ├── case2_vonmises_linear_refined.png
│   ├── case2_vonmises_parabolic_refined.png
│   ├── case2_disp_linear.png
│   ├── case2_disp_parabolic_refined.png
│   ├── case2_principal_linear.png
│   └── case2_principal_parabolic_refined.png
└── docs/
    └── component_spec.json            ← Full engineering specification
```

---

## License

MIT License — see [LICENSE](LICENSE)

---

## Author

**Siddhant Gawad**  
MEng Mechanical Engineering — University of Windsor (May 2026)  
[LinkedIn](https://linkedin.com/in/siddhantgawad) · [GitHub](https://github.com/siddgawad) · [Portfolio](https://gawaddeveloper.vercel.app)  
PGWP — 3 years, no sponsorship required
