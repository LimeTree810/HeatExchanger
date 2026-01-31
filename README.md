# A Study of Designing Heat Exchanger
This repository supports the workflow:

1) Generate design samples (LHS)  
2) Convert design samples to porous parameters (Darcy–Forchheimer: 1/K, C2, etc.)  
3) Train GP surrogate models (Q'' and ΔP) and optionally run GA optimization / prediction
4) Validate the Value with CFD Data

## I. Step-by-Step

### 0) Setup (recommended)

From the repository root (`HeatExchanger/`):

```bash
# Go to repo root
cd /path/to/HeatExchanger

# (Recommended) create & activate a venv
python3 -m venv .venv
source .venv/bin/activate

# install dependencies
pip install -U pip
pip install numpy pandas scipy scikit-learn pygad
```


### 0.1 (Optional) Conda environment

```bash
cd /path/to/HeatExchanger

conda create -n wind-hx python=3.10 -y
conda activate wind-hx

# install dependencies
pip install -U pip
pip install -r requirements.txt
```

> Pick **one** environment manager: venv or conda (you don't need both).


## 🚀 Installation & Execution

### Required Packages
```bash
pip install numpy pandas scipy
Run
python au.py
Key Configuration Parameters (top of file)
N_SAMPLES = 1000000                      # number of samples to generate
OUTPUT_FILENAME = "LHS_Porous_Params_Result.csv"
RANDOM_SEED = 2025                       # reproducibility
N_POINTS_FITTING = 25                    # curve-fitting points (15–50)
AMB_TEMP_C = 14.8                        # ambient temperature [°C]
DESIGN_VELOCITY = 2.019                  # design velocity [m/s]
📊 Output File Structure
Filename
LHS_Porous_Params_Result.csv

Main Columns
Input Variables
S1_mm: transverse tube pitch (45–200 mm)

fin_height_fh_mm: fin height (6 mm to constraint-based upper bound)

fin_spacing_fs_mm: fin spacing (2–8 mm)

Porous-Media Parameters (CFD Inputs)
Viscous_Resistance_1_m2: viscous resistance (1/K) 
1
/
𝑚
2
1/m 
2
 

Inertial_Resistance_1_m: inertial resistance (C2) 
1
/
𝑚
1/m

Porosity: porosity (epsilon, ε)

a_fs_1_m: specific surface area 
1
/
𝑚
1/m

Thermo-Hydraulic Performance
dP_total_Pa: total pressure drop (4-row bank) 
𝑃
𝑎
Pa

h_fs_W_m2K: heat transfer coefficient 
𝑊
/
𝑚
2
/
𝐾
W/m 
2
 /K

Re_Dc: Reynolds number

R2_fit: fitting accuracy (recommended: ≥ 0.99)

🧠 Theoretical Background
Design Constraints
S1 (Pitch)        : 45 ~ 200 mm
Fin Height (fh)   : 6 <= fh <= 0.5 * (S1/2 - 24.0) - 0.4
Fin Spacing (fs)  : 2 ~ 8 mm
Geometric constraints are automatically applied to prevent tube collision.

Infeasible combinations are automatically removed during LHS sampling.

Physical Models
Step 1: Geometry Computation
Annular-fin geometric parameters (area ratio, porosity, minimum flow-area ratio)

Step 2: Pressure Drop (Nir, 1991)
4-row staggered fin–tube bank

Evaluate at 25 velocity points over 0.5–3.5 m/s

Step 3: Darcy–Forchheimer Fitting
ΔP/L = A·v + B·v^2

Viscous term: 1/K = A/μ
Inertial term: C2 = 2B/ρ
Step 4: Heat Transfer (Briggs & Young, 1963)
Compute Nu and h at the design velocity (2.019 m/s)

References
Nir, A. (1991). “Heat Transfer and Friction Factor Correlations for Crossflow over Staggered Finned Tube Banks.” Heat Transfer Engineering, 12(1), 43–58.

Briggs, D.E., Young, E.H. (1963). “Convection Heat Transfer and Pressure Drop of Air Flowing Across Triangular Pitch Banks of Finned Tubes.”

⚡ Performance Guide
Expected Runtime
# Samples	Fitting Points	Expected Time
10,000	15	~10 s
100,000	25	~2 min
1,000,000	25	~20 min
Optimization Tips
For quick testing: start with N_SAMPLES = 10000

Speed-first: reduce to N_POINTS_FITTING = 15

Accuracy-first: increase to N_POINTS_FITTING = 50

🔍 Data Validation
Example checks for generated data quality:

import pandas as pd

df = pd.read_csv("LHS_Porous_Params_Result.csv")

# basic statistics
print(df.describe())

# R² check (recommended: >= 0.99)
print(f"R2 mean: {df['R2_fit'].mean():.4f}")
print(f"R2 min : {df['R2_fit'].min():.4f}")

# error samples
errors = df[df["ok"] == False]
print(f"Error samples: {len(errors)}")
📁 Project Structure
.
├─ au.py                         # main script
└─ LHS_Porous_Params_Result.csv   # output (generated after execution)
💡 Use Cases
CFD porous-zone inputs: directly use Viscous/Inertial Resistance in Fluent/StarCCM+ porous zone settings

Surrogate model training: train ML/DL models using up to 1M samples

Optimal design exploration: analyze pressure-drop vs heat-transfer trade-offs

Design of Experiments (DOE): select efficient experimental points via LHS

### 3) Train GP Surrogate Models (Q'' and ΔP)

This stage trains Gaussian Process models for:
- `Q''` (heat flux, W/m²)
- `ΔP` (pressure drop, Pa)

> Your surrogate training script(s) may differ by filename. Use the commands below as a template.

#### 3.1 Train surrogate

```bash
cd /path/to/HeatExchanger

# Example: run a surrogate training script
python3 scripts/surrogate/train_surrogate.py --data data/total_2D_Data.xlsx
```

If your script is a plain Python file without CLI args, just run it directly:
```bash
cd /path/to/HeatExchanger
python3 scripts/surrogate/train_surrogate.py
```

#### 3.2 Run GA optimization (optional)

```bash
cd /path/to/HeatExchanger

python3 scripts/surrogate/train_surrogate.py --mode ga --data data/total_2D_Data.xlsx
```

#### 3.3 Predict using a user-defined design (optional)

```bash
cd /path/to/HeatExchanger

python3 scripts/surrogate/train_surrogate.py --mode predict   --data data/total_2D_Data.xlsx   --s1 181.0394 --fh 28.9923 --fs 2.6038
```

---

### 4) Validate the value with CFD data

Compare:
- Surrogate-predicted `Q''` and `ΔP`
vs.
- CFD-evaluated `Q''` and `ΔP`

#### 4.1 Prepare CFD validation dataset

Put your CFD results into `HeatExchanger/data/` as a CSV or Excel file.

**Recommended columns**
- Inputs:
  - `S1_mm`, `fin_height_fh_mm`, `fin_spacing_fs_mm`
- CFD outputs (suggested names):
  - `Q_CFD` (or `Qpp_CFD`)
  - `dP_CFD` (or `DeltaP_CFD`)

```bash
cd /path/to/HeatExchanger
ls -lh data | grep -i cfd
```

#### 4.2 Run validation script

```bash
cd /path/to/HeatExchanger

python3 scripts/validation/validate_with_cfd.py   --train data/total_2D_Data.xlsx   --cfd   data/CFD_validation.csv   --out   data/validation_report.csv
```

Expected outputs:
- `data/validation_report.csv` (per-sample error summary)
- (Optional) parity plots / error histograms (if implemented)

#### 4.3 What to report

Common validation metrics:
- R² (CFD vs surrogate)
- MAE / RMSE for `Q''` and `ΔP`
- Parity plot (y = x) and error distribution
- Worst-case error samples (top-k)

---

## II. Dataset

### A) Prebuilt dataset (download)

Place into:
- `HeatExchanger/data/`

| Dataset | Purpose | Target path (repo) | File name | Download link | Notes |
|---|---|---|---|---|---|
| porousdata | Prebuilt porous parameters dataset (1/K, C2, etc.) | `data/` | `porousdata.xlsx` | [Download](<PUT_LINK_HERE>) | Put the file exactly at `HeatExchanger/data/porousdata.xlsx` |
| total_2D_Data | CFD/2D training dataset for surrogate (Q'', ΔP) | `data/` | `total_2D_Data.xlsx` | [Download](<PUT_LINK_HERE>) | Used by GP surrogate training + GA optimization |

Examples:
- `data/porousdata.xlsx`
- `data/total_2D_Data.xlsx`

### B) Locally generated dataset

Generated by:
- `scripts/surrogate/sampling.py`

Output example:
- `data/LHS_Porous_Params_Result.csv`

---
