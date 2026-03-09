# EEM Analysis: Step-by-Step Calculation Pipeline

## Study: Impact of German Electricity Market Zone Splitting on Charging Demand

**Corridor:** Scandinavian-Mediterranean (NO, FI, SE, DK, DE, AT, IT)
**Model:** TransComp (Julia/JuMP + Gurobi)
**Horizon:** 2020–2060 (annual steps)

---

## Scenario Matrix

| # | Folder name | Spread | Split from | Description |
|---|------------|--------|-----------|-------------|
| 1 | `DE_unified` | 0% | — | Baseline: single German price zone (Go RES) |
| 2 | `DE_split_low_from2020` | 5% | 2020 | Low spread, immediate |
| 3 | `DE_split_low_from2032` | 5% | 2032 | Low spread, delayed |
| 4 | `DE_split_low_from2040` | 5% | 2040 | Low spread, late |
| 5 | `DE_split_mid_from2020` | 10% | 2020 | Mid spread, immediate |
| 6 | `DE_split_mid_from2032` | 10% | 2032 | Mid spread, delayed |
| 7 | `DE_split_mid_from2040` | 10% | 2040 | Mid spread, late |
| 8 | `DE_split_high_from2020` | 20% | 2020 | High spread, immediate |
| 9 | `DE_split_high_from2032` | 20% | 2032 | High spread, delayed |
| 10 | `DE_split_high_from2040` | 20% | 2040 | High spread, late |

**Spread rule:** North = price × (1 − spread/2), South = price × (1 + spread/2).
Before the split year, all German nodes keep the baseline (unified) price.

**Zone definition (TSO boundaries):**
- **North** (50Hertz + TenneT-North): DE30, DE40, DE50, DE60, DE80, DE91–DE94, DEF0, DED2, DED4, DED5, DEE0, DEG0
- **South** (Amprion + TransnetBW + Bavaria): DE11–DE14, DE21–DE27, DE71–DE73, DEA1–DEA5, DEB1–DEB3, DEC0

---

## Pipeline Steps

### Step 0: Prerequisites (already done)

Ensure the source NUTS-2 data exists:

```
examples/moving_loads_SM/input_data/fixed_nuts_2/
```

This contains the pre-processed NUTS-2 geographic network, paths, and OD pairs (including both international and domestic routes).

---

### Step 1: Generate EEM scenario input files

**Script:** `create_eem_scenarios.py`
**What it does:**
1. **Generates baseline** using `SM_preprocessing.py` with `keep_de_cross_zone=True`:
   - Runs full preprocessing from `fixed_nuts_2/` source data
   - Keeps international routes passing through DE + German domestic N-S cross-zone routes
   - Applies filters: ≥360 km path length, ≥500 t freight volume, transit through Germany
   - Excludes 249 routes not passing through DE (AT-IT, DK-NO via SE, etc.)
   - Result: ~1,870 routes (~1,787 international through DE + 83 DE cross-zone domestic)
2. For zone-split scenarios, copies all YAML files from baseline and modifies `FuelCost.yaml`:
   - Identifies German nodes by NUTS-2 code → North/South zone
   - Applies relative price adjustment to electricity entries only
   - Only modifies years ≥ `split_from_year`; earlier years stay at baseline
3. Writes `eem_scenario_metadata.yaml` with scenario parameters

**Run:**
```bash
cd examples/moving_loads_SM
python create_eem_scenarios.py
```

**Output:** 10 folders under `input_data/eem_scenarios/`:
```
input_data/eem_scenarios/
├── DE_unified/                    (baseline, copy of go_res)
├── DE_split_low_from2020/         (5% spread, from 2020)
├── DE_split_low_from2032/         (5% spread, from 2032)
├── DE_split_low_from2040/         (5% spread, from 2040)
├── DE_split_mid_from2020/         (10% spread, from 2020)
├── DE_split_mid_from2032/         (10% spread, from 2032)
├── DE_split_mid_from2040/         (10% spread, from 2040)
├── DE_split_high_from2020/        (20% spread, from 2020)
├── DE_split_high_from2032/        (20% spread, from 2032)
└── DE_split_high_from2040/        (20% spread, from 2040)
```

Each folder contains the full set of 22 YAML input files. Only `FuelCost.yaml` differs between scenarios (electricity prices for German nodes).

**Verification:** Check that the price modification is correct:
```bash
python plot_eem_electricity_prices_4panels.py
```
→ Produces `figures/eem_electricity_prices_by_country_4scenarios.png`

---

### Step 2: Run the optimization model (batch)

**Script:** `SM_batch_eem.jl`
**What it does** (for each of the 10 scenarios):
1. Reads 22 YAML input files → `get_input_data()`
2. Parses into data structures → `parse_data()`
3. Creates JuMP optimization model → `create_model()`
4. Adds constraints:
   - Mandatory breaks (driving time regulation)
   - Fueling demand (energy balance per route segment)
   - Fueling infrastructure (capacity ↔ demand)
   - Infrastructure expansion shift (investment timing)
   - SOC tracking and SOC max (battery state-of-charge along route)
   - Travel time tracking
   - Mode shift (road ↔ rail)
   - Max mode share (policy caps)
   - Demand coverage (all freight must be served)
   - Vehicle sizing, aging, stock shift (fleet dynamics)
   - No BEV purchases in 2020 (initial condition)
5. Adds objective function (minimize total system cost)
6. Solves with Gurobi (barrier method, 1% MIP gap)
7. Saves results to `results/eem_scenarios/<folder_name>/`

**Run:**
```bash
cd examples/moving_loads_SM
julia SM_batch_eem.jl
```

**⚠️ Before running:** The batch script (`SM_batch_eem.jl`) must be updated to reference the new folder names with timing suffixes. See Step 2a below.

**Expected runtime:** ~1–3 hours per scenario (depends on number of OD pairs and machine). Total: ~10–30 hours for all 10 scenarios.

**Output per scenario:**
```
results/eem_scenarios/<folder_name>/
├── eem_<case_name>_f_dict.yaml          # fuel consumption
├── eem_<case_name>_h_dict.yaml          # fleet stock (vehicles)
├── eem_<case_name>_h_plus_dict.yaml     # new vehicle purchases
├── eem_<case_name>_h_exist_dict.yaml    # existing vehicles
├── eem_<case_name>_h_minus_dict.yaml    # vehicle retirements
├── eem_<case_name>_s_dict.yaml          # charging energy by node
├── eem_<case_name>_soc_dict.yaml        # state of charge along routes
├── eem_<case_name>_q_fuel_infr_plus_dict.yaml          # infra capacity added
├── eem_<case_name>_q_fuel_infr_plus_by_route_dict.yaml # infra by route
├── eem_<case_name>_travel_time_dict.yaml               # travel times
├── eem_<case_name>_extra_break_time_dict.yaml          # extra break time
└── summary_stats.yaml                   # objective value, fleet by year
```

---

### Step 2a: Update SM_batch_eem.jl for new scenarios

The current `SM_batch_eem.jl` references the old folder names (without timing suffixes). It needs to be updated to list all 10 scenario folders with the new naming convention.

---

### Step 3: Post-processing and visualization

**Scripts** (to be created/updated after results are available):

#### 3a. Electricity price comparison figure
- **Script:** `plot_eem_electricity_prices_4panels.py`
- **Output:** `figures/eem_electricity_prices_by_country_4scenarios.pdf`
- Shows DE-North vs DE-South prices against a grey band of other corridor countries

#### 3b. BEV fleet development comparison
- Compare BEV adoption trajectories across the 10 scenarios
- Key metric: BEV share by year, BEV fleet size by year

#### 3c. Charging demand spatial redistribution
- Compare charging energy (variable `s`) by NUTS-2 region
- Key question: Does the zone split shift charging demand from South-DE to North-DE?
- Show as: latitude-based charging profile, or NUTS-2 heatmap delta

#### 3d. Infrastructure investment comparison
- Compare `q_fuel_infr_plus` across scenarios
- Where is new charging infrastructure built differently?

#### 3e. Cost comparison
- Compare objective function values (total system cost)
- Levelized cost per tkm by scenario

#### 3f. Timing effect analysis
- For each spread level: compare from2020 vs from2032 vs from2040
- Does early splitting lock in different infrastructure patterns?

---

## File Locations Summary

| Purpose | Path |
|---------|------|
| Scenario generation | `examples/moving_loads_SM/create_eem_scenarios.py` |
| Batch runner (Julia) | `examples/moving_loads_SM/SM_batch_eem.jl` |
| Input data | `examples/moving_loads_SM/input_data/eem_scenarios/` |
| Results | `examples/moving_loads_SM/results/eem_scenarios/` |
| Price plot | `examples/moving_loads_SM/plot_eem_electricity_prices_4panels.py` |
| Figures | `examples/moving_loads_SM/figures/eem_*.pdf` |
| EEM paper | `-EEM-Impact-of-.../conference_101719.tex` |
| Baseline data source | `examples/moving_loads_SM/input_data/genesys_scenarios/go_res/` |

---

## Key Assumptions

1. **Energy system scenario:** Go RES (GENeSYS-MOD) — high renewable ambition
2. **Charging markup:** 30% above wholesale price (CPO margin, grid fees)
3. **Zone split:** symmetric relative adjustment (North cheaper, South more expensive)
4. **Non-German prices:** unchanged across all scenarios (only DE is modified)
5. **Diesel + CO₂:** same across all scenarios (emission factor: 0.2664 kgCO₂/kWh diesel)
6. **Fleet:** no BEV purchases allowed in 2020 (initial year)
7. **Route network:** international NUTS-2 routes along SM corridor only
