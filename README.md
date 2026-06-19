# 🛢️ PVT Adjustment, Reimagined: A Pandas DataFrame Approach

> *Same well. Same physics. Same correction factors. A completely different engine under the hood — pandas instead of pure Python dictionaries.*

---

## 📌 Branch Overview

This is the **`Using-Dataframe` branch** of the PVT Workbook Extraction project. It re-implements the entire PVT adjustment workflow — originally built using plain Python dictionaries and `for` loops on `main` — using **pandas DataFrames** and **vectorized NumPy operations** instead.

The well, the lab data, and the underlying petroleum engineering logic are identical to the main branch. What changes is the *engineering of the code itself*: how the data is stored, how lookups are performed, and how the row-wise adjustment logic is applied.

This branch represents a natural next step after learning core Python — moving from manual dictionary manipulation to the tabular, vectorized thinking that pandas was built for, which is also exactly how a real petroleum engineer would expect to interact with PVT data in practice (think: Excel tables, but programmable).

---

## 🔬 The Engineering Context (Unchanged from Main)

PVT analysis underpins every reservoir simulation and material balance calculation. Laboratory CCE and DLE measurements are taken under single-stage separator conditions; real surface facilities run multi-stage separator trains. The adjustment workflow corrects lab-measured Bo and Rs to match the field's actual optimum separator condition:

```
Excel Workbook (.xlsx) → pandas DataFrames → Optimum Separator Selection → Correction Factors → Adjusted PVT DataFrame
```

**Well:** NDI-OML-40 #22 | **Reservoir Temp:** 207.2°F | **Bubble Point:** 988 psia | **Fluid:** Black Oil

---

## 🆚 What's Different in This Branch

| Aspect | `main` (Dictionary-Based) | `Using-Dataframe` (This Branch) |
|---|---|---|
| **Data structure** | Python `dict` of lists, one per sheet | `pandas.DataFrame`, one per sheet |
| **Excel import** | `openpyxl`, sheet-by-sheet manual parsing | `pd.read_excel(file, sheet_name=None)` — all sheets in one call |
| **Data source** | Local upload | **Read directly from a GitHub raw URL** — no local file needed |
| **Bubble point lookup** | `DLE['pressure'].index(Pb)` | `dle_df[dle_df['Pressure'] == Pb]` — boolean masking |
| **Optimum separator search** | `seperator_test['formation volume factor'].index(min(...))` | `sep_df.loc[sep_df['Bo'].idxmin()]` — one line, no manual indexing |
| **Row-wise adjustment logic** | Explicit `for` loop with `if/else` per row | `np.where(condition, value_if_true, value_if_false)` — vectorized, no loop |
| **Aligning CCE data to DLE pressures** | Implicit (same list order assumed) | Explicit `.set_index().reindex()` — robust even if row order differs |
| **Output** | Dictionary printed to console | Clean `adjusted_pvt_df` — ready for `.to_csv()`, `.plot()`, or further analysis |

The numerical results are **identical** between both branches — this is intentional. The same correction factors emerge:

| Factor | Value |
|---|---|
| **CF_Bo** | 0.9668 |
| **CF_Rs** | 0.8919 |

What changes is *how confidently and safely* that result was produced — and how much further the resulting object can travel (a DataFrame plugs directly into plotting, export, and merge operations; a dictionary needs manual unpacking first).

---

## 📥 Data Loading — Straight from GitHub

Unlike the `main` branch which works from a locally uploaded Excel file, this branch reads the workbook **directly from its raw GitHub URL**:

```python
file_path = "https://github.com/Calebchike/PVT-Extraction-Adjustment-Project/raw/Using-Dataframe/PVT_Workbook%20.xlsx"
all_sheets = pd.read_excel(file_path, sheet_name=None)

summary_df = all_sheets['Summary']
cce_df     = all_sheets['CCE']
dle_df     = all_sheets['DLE']
sep_df     = all_sheets['Separator Tests']
```

This makes the notebook **fully self-contained and reproducible** — anyone can run it in Colab without uploading a single file, since the data travels with the repository itself.

---

## ⚙️ Key Code Walkthroughs

### 1. Finding DLE Values at Bubble Point — Boolean Masking

```python
bubble_point_pressure = summary_df.loc[
    summary_df['Well information'] == 'Bubble Point Pressure', 'Value'
].iloc[0]

dle_Pb = dle_df[dle_df['Pressure'] == bubble_point_pressure]
```

Instead of manually finding a list index, pandas filters the entire DLE table down to the single row matching the bubble point pressure — readable, and immune to ordering assumptions.

---

### 2. Selecting the Optimum Separator — `idxmin()`

```python
optimum_sep = sep_df.loc[sep_df['Bo'].idxmin()]
```

`idxmin()` returns the index of the row with the lowest Formation Volume Factor in a single call. The result — **3rd Sep Test, 102 psia / 90°F** — matches the manual dictionary-based selection on `main` exactly, but with none of the manual `.index(min(...))` bookkeeping.

---

### 3. The Adjustment Logic — Vectorized with `np.where`

This is the heart of the branch. Where `main` used an explicit `for` loop with an `if/else` per pressure step, this branch applies the same logic across the **entire column at once**:

```python
condition_cce_adjustment = adjusted_pvt_df['Pressure'] > pb_value

adjusted_pvt_df['Bo_adjusted'] = np.where(
    condition_cce_adjustment,
    adjusted_pvt_df['Bo_DLE'] * cce_relative_volume_aligned,  # P > Pb: CCE correction
    adjusted_pvt_df['Bo_DLE'] * Bo_correction_factor          # P ≤ Pb: separator correction
)

adjusted_pvt_df['Rs_adjusted'] = np.where(
    condition_cce_adjustment,
    adjusted_pvt_df['Rs_DLE'],                                 # P > Pb: Rs constant
    adjusted_pvt_df['Rs_DLE'] * Rs_correction_factor           # P ≤ Pb: separator correction
)
```

No loop, no index tracking, no risk of off-by-one errors between the CCE and DLE tables — `np.where` handles the entire conditional branching for all 16 pressure steps in two lines.

---

### 4. Aligning Two Tables with Different Row Counts

A subtle but important detail: the CCE table has 14 pressure rows, while DLE has 16. Before the CCE relative volume can be multiplied against the DLE table, the two need to be aligned on pressure:

```python
cce_relative_volume_aligned = (
    cce_df.set_index('Pressure')['Relative Volume']
          .reindex(dle_df['Pressure'])
          .reset_index(drop=True)
)
```

This guarantees that each DLE pressure step gets matched to the *correct* CCE relative volume — or `NaN` if no match exists below the bubble point (which is expected, since CCE relative volume isn't measured there).

---

## 📊 Final Output — `adjusted_pvt_df`

| Pressure (psia) | Bo_DLE | Rs_DLE | **Bo_adjusted** | **Rs_adjusted** |
|---|---|---|---|---|
| 5500 | 1.159 | 261.5 | 1.1038 | 261.50 |
| 4090 | 1.173 | 261.5 | 1.1314 | 261.50 |
| 988 *(Pb)* | 1.217 | 261.5 | 1.1766 | 233.24 |
| 800 | 1.208 | 227.8 | 1.1679 | 203.18 |
| 400 | 1.162 | 126.9 | 1.1234 | 113.19 |
| 15 | 1.066 | 0.0 | 1.0306 | 0.00 |

*Full 16-row table generated in the notebook — values verified identical to the `main` branch dictionary output.*

The transition at **988 psia (bubble point)** is the key inflection: above it, Bo is corrected by the CCE relative volume curve and Rs stays constant; at and below it, both properties are corrected using the fixed separator correction factors.

---

## 🛠️ Tools & Libraries

| Tool | Purpose |
|---|---|
| **Python 3** | Core language |
| **pandas** | DataFrame construction, Excel I/O, boolean masking, `idxmin()`, `reindex()` |
| **NumPy** | Vectorized conditional logic via `np.where()` |
| **openpyxl** | Excel engine used internally by `pd.read_excel()` |
| **Google Colab** | Notebook execution environment |

---

## 📁 Repository Structure (This Branch)

```
Using-Dataframe/
│
├── PVT_workbook_extraction.ipynb     ← This notebook (DataFrame implementation)
├── PVT_Workbook .xlsx                ← Source workbook, read directly via raw GitHub URL
└── README.md                         ← This file
```

> **Note:** This branch's notebook fetches the Excel file directly from its own raw GitHub URL — no local upload step is required to reproduce the results.

---

## 🔗 Related Branches

| Branch | Approach | Best For |
|---|---|---|
| [`main`](https://github.com/Calebchike/PVT-Extraction-Adjustment-Project) | Python dictionaries + `for` loops | Understanding the raw logic step-by-step, beginner-friendly |
| **`Using-Dataframe`** *(this branch)* | pandas DataFrames + vectorized `np.where()` | Production-style code, scalability, integration with further analysis (plotting, export, merging) |

---

*This branch reflects a deliberate progression — taking a working dictionary-based solution and rebuilding it in pandas to practice the data structures and vectorized thinking that scale to real-world reservoir datasets.*

---

> *"A dictionary tells you what the data is. A DataFrame tells you what the data can become."*
