# 🛢️ From Excel to Python: Automated PVT Workbook Processing for Black Oil Reservoirs

> *What used to be done manually in Excel — correction factors, separator selection, Bo and Rs adjustment — now runs in Python with full traceability, reproducibility, and zero formula errors.*

---

## 📌 Project Overview

This project automates the **Pressure-Volume-Temperature (PVT) adjustment workflow** for black oil reservoir fluid characterization. Previously, the entire adjustment process was performed manually in an Excel workbook — calculating correction factors by hand, selecting the optimum separator condition visually, and applying multiplications cell-by-cell.

This Python implementation replicates — and validates — every step of that Excel workflow programmatically, using structured dictionaries, index-based lookups, and correction factor arithmetic. It is a direct translation of petroleum engineering logic into clean, annotated Python code.

---

## 🔬 The Engineering Context

PVT analysis is the foundation of every reservoir simulation and material balance calculation. The raw data from laboratory experiments (CCE and DLE) is measured under ideal single-separator conditions. Real surface facilities operate with **multi-stage separator trains** — and the PVT parameters must be corrected to match the actual separator conditions before being used in simulation.

The three-step correction process this project automates:

```
DLE Lab Data → Optimum Separator Selection → Correction Factors → Adjusted PVT Table
```

This is standard practice in petroleum engineering, and previously required careful, error-prone manual work in spreadsheets.

---

## 🗂️ Well & Fluid Summary

| Parameter | Value | Unit |
|---|---|---|
| **Initial Reservoir Pressure** | 4,089.7 | psia |
| **Reservoir Temperature** | 207.2 | °F |
| **Bubble Point Pressure (Pb)** | 988 | psia |
| **Fluid Type** | Black Oil | — |
| **API Gravity** | 40.79 | °API |
| **Mol% C7+** | 52.9 | % |
| **Fluid Mol Weight** | 122.05 | g/mol |
| **Viscosity @ Initial Pressure** | 0.989 | cP |
| **Viscosity @ Bubble Point** | 0.554 | cP |

---

## 🔄 Processing Pipeline

<img width="900" height="400" alt="08_workflow_pipeline" src="https://github.com/user-attachments/assets/34fcc5ee-bffe-438b-97d6-39d2c5edb21b" />


The notebook is structured in the exact order a PVT engineer would work through the problem — from raw data input to final adjusted table.

---

## 📊 Experiments & Datasets

### Constant Composition Expansion (CCE)

The CCE study was conducted at **207.2°F** across 14 pressure steps from 5,500 psia down to 200 psia. It captures the fluid's volumetric behaviour — how the oil expands as pressure drops — and identifies the bubble point through a discontinuity in relative volume.

Above the bubble point (single-phase region), relative volume increases smoothly and gradually. At 988 psia the fluid reaches saturation — below that, gas liberates and the relative volume expands sharply. The Y-function takes over in the two-phase region.

---

Oil density decreases monotonically with pressure in the single-phase region — from **0.7663 g/cm³ at 5,500 psia** down to **0.7298 g/cm³ at bubble point**. Isothermal compressibility was also captured and ranges from 8.32×10⁻⁶ to 1.33×10⁻⁵ psi⁻¹.

---

### Differential Liberation Experiment (DLE)

The DLE study at **207.2°F** across 16 pressure steps models the stepwise liberation of gas as pressure drops below the bubble point — simulating reservoir depletion.


Key observations:
- **Rs (Solution GOR)** remains flat at **261.5 scf/stb** above the bubble point — no gas has liberated yet. Below Pb it drops steadily to 0 at stock-tank conditions (15 psia).
- **Bo (Oil FVF)** reaches its maximum of **1.217 bbl/stb at bubble point**, then contracts as dissolved gas escapes below Pb.

---

Below the bubble point, liberated gas properties become available. **Bg** (gas formation volume factor) rises steeply as pressure falls — from 0.0217 cu ft/scf at 800 psia to 1.2233 cu ft/scf at 15 psia. The **gas Z-factor** was calculated using the Lee-Gonzalez correlation.

---

### Separator Tests

Four single-stage separator tests were conducted at varying Stage 1 conditions, all at 90°F and with Stage 2 fixed at 15 psia / 60°F. The optimum separator is identified as the one with the **lowest Formation Volume Factor** — minimising shrinkage and maximising stock-tank liquid recovery.

<img width="900" height="500" alt="05_separator_fvf" src="https://github.com/user-attachments/assets/deaefe9c-a02c-477e-959e-fc039ba56999" />


The **3rd separator test at 102 psia / 90°F** was selected as optimum, with:
- Formation Volume Factor = **1.1766 bbl/stb** (minimum)
- Stock Tank API = **42.27°API** (maximum)
- Total GOR = **233.24 scf/stb** (minimum)

In the Python implementation, this selection is automated — the code finds the index of `min(formation_volume_factor)` and extracts all corresponding parameters in one pass. No manual scanning of a table required.

---

## ⚙️ Correction Factors & Adjustment Methodology

With the optimum separator identified, two correction factors are calculated to bring the DLE data into alignment with real surface conditions:

| Factor | Formula | Value |
|---|---|---|
| **CF_Bo** | Fob_sep / Bob_DLE | **0.9668** |
| **CF_Rs** | Rsp_sep / Rsb_DLE | **0.8919** |

These factors are then applied pressure-by-pressure across the full DLE table using a conditional loop:

```python
# Above bubble point: CCE relative volume correction applies to Bo
# Rs is constant (no gas has liberated)
if pressure > Pb:
    Bo_adjusted = Bo_DLE * CCE_relative_volume
    Rs_adjusted = Rs_DLE_at_Pb   # constant

# Below bubble point: separator correction factors apply
else:
    Bo_adjusted = Bo_DLE * CF_Bo
    Rs_adjusted = Rs_DLE * CF_Rs
```

---

## 📈 Adjusted PVT Results

<img width="900" height="500" alt="06_bo_adjustment_comparison" src="https://github.com/user-attachments/assets/65ca582d-2e2e-4425-890c-2d80b3981f7a" />


The adjusted Bo values are consistently **3.3% lower** than the DLE values across all pressures — reflecting the shrinkage correction from the laboratory single-stage condition to the actual 3rd separator condition. The divergence begins precisely at bubble point, where the correction factor transitions from CCE-based to separator-based.

---

<img width="900" height="500" alt="07_rs_adjustment_comparison" src="https://github.com/user-attachments/assets/0f348fce-c206-4db2-8a90-483f6631b3c0" />


The adjusted Rs values below the bubble point are **10.8% lower** than DLE Rs — accounting for the additional gas that remains dissolved under the optimum separator conditions rather than being flash-liberated. Above Pb, Rs is correctly held constant at the bubble-point value for both curves.

---

## 🔁 Excel vs Python: What Changed

| Step | Excel Approach | Python Approach |
|---|---|---|
| Data storage | Hardcoded cells per sheet | Python dictionaries with inline units |
| Bubble point extraction | Manual cell reference | `DLE['pressure'].index(Pb)` — auto-lookup |
| Optimum separator selection | Visual inspection of table | `min()` + `.index()` — fully automated |
| Correction factor calculation | Typed formula in cell | Named variables with comments |
| PVT adjustment | Row-by-row formula drag | `for` loop with conditional logic |
| Reproducibility | Re-open file, check cells | Re-run notebook — deterministic |
| Audit trail | Hidden in formula bar | Fully visible, annotated code |

---

## 🛠️ Tools & Libraries

| Tool | Purpose |
|---|---|
| **Python 3** | Core language |
| **Google Colab** | Notebook execution environment |
| **openpyxl** | Excel workbook read/write |
| **Built-in Python** | `dict`, `list`, `for`, `min()`, `.index()` |
| **Microsoft Excel** | Reference workbook (original manual method) |

> No external petroleum engineering libraries were used. All PVT logic was implemented from first principles using standard Python.

---

*This project bridges two disciplines — petroleum engineering PVT analysis and Python programming — demonstrating how manual spreadsheet workflows in reservoir engineering can be fully automated with clean, reproducible code.*

---

> *"The reservoir does not change. But how we understand it depends entirely on the quality of our PVT data — and the rigour of our corrections."*
