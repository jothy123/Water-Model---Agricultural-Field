# 💧 WFLDB Spatial Water Accounting Model

> A spatially explicit Python model for computing all five ISO 14046 water flows in agricultural systems, following the **World Food LCA Database (WFLDB) v3.10** methodological guidelines.

---

## 📖 Overview

This model produces **five spatial water output layers** (GeoTIFF rasters) for any crop and region, directly implementing the WFLDB v3.10 water use methodology. It supports blue, green, and grey water accounting at field to national scales, with outputs normalized per tonne of harvested crop [m³/t] and per hectare [m³/ha].

The five output flows follow the **ISO 14046:2014** water footprint standard:

| # | Output Flow | Type | Unit |
|---|---|---|---|
| 1 | Water withdrawal (surface + groundwater) | Input from biosphere | m³/ha |
| 2 | Water emitted to air (evapotranspiration) | Output to biosphere | m³/ha |
| 3 | Water emitted to surface water (return flow) | Output to biosphere | m³/ha |
| 4 | Water emitted to groundwater (percolation) | Output to biosphere | m³/ha |
| 5 | Wastewater sent to treatment | Output to technosphere | m³/ha |
| + | Grey water volume (Hoekstra method) | Supplementary | m³/ha |

---

## 🧮 Core Equations

All equations are implemented directly from **WFLDB v3.10, Section 4.4** (Lévová et al. 2012):

```
# Irrigation efficiency (FAO 1989 / ICID 2021)
EFirr = Σ (share_i × EFirr_i)       where i ∈ {surface, sprinkler, drip}

# Flow 1 — Water withdrawal
I_withdrawal = ETirr / EFirr         [m³/t]

# Flow 2 — Water to air (blue water footprint)
W_air = ETirr                        [m³/t]

# Flow 3 — Return flow to surface water
W_surface = 0.8 × ((ETirr / EFirr) − ETirr)   [m³/t]

# Flow 4 — Return flow to groundwater
W_ground  = 0.2 × ((ETirr / EFirr) − ETirr)   [m³/t]

# Mass balance check (must hold per pixel)
I_withdrawal = W_air + W_surface + W_ground    ✓

# Flow 5 — Wastewater (animal production)
W_ww = 0.17 × W_livestock_use        [m³/ha]   (Shaffer 2008)

# Supplementary — Grey water (Hoekstra et al. 2011)
GW = max(L_i / (c_max_i − c_nat_i))  [m³/t]   for i ∈ {NO₃, PO₄, pesticides}
```

---

## 🗂️ Repository Structure

```
wfldb-water-model/
│
├── wfldb_water_model.py        # Main model — all 5 flows + grey water
│
├── data/                       # Input data directory (not tracked by git)
│   ├── crop_mask.tif           # Harvested area raster [ha/pixel] (e.g. SPAM2020)
│   ├── yield_tha.tif           # Yield raster [t/ha]
│   ├── no3_load_kg_per_t.tif   # Nitrate load [kg NO₃/t crop]
│   ├── po4_load_kg_per_t.tif   # Phosphate load [kg PO₄/t crop]
│   ├── pest_load_kg_per_t.tif  # Pesticide load [kg a.i./t crop] (optional)
│   ├── livestock_water_use.tif # Livestock water use [m³/ha]
│   ├── countries_iso3.shp      # Admin boundaries with ISO3 codes
│   ├── etirr_pfister2011.csv   # ETirr lookup (Pfister et al. 2011)
│   ├── irrigation_efficiency_icid2021.csv  # Tech shares (ICID 2021)
│   └── water_origin_siebert2010.csv        # SW/GW shares (Siebert et al. 2010)
│
├── outputs/                    # Generated GeoTIFF outputs (not tracked)
│   ├── flow1_withdrawal_total_m3_ha.tif
│   ├── flow1a_withdrawal_surface_m3_ha.tif
│   ├── flow1b_withdrawal_ground_m3_ha.tif
│   ├── flow2_water_to_air_m3_ha.tif
│   ├── flow3_water_to_surface_m3_ha.tif
│   ├── flow4_water_to_ground_m3_ha.tif
│   ├── flow5_wastewater_m3_ha.tif
│   └── grey_water_m3_ha.tif
│
├── requirements.txt            # Python dependencies
├── .gitignore
└── README.md
```

---

## ⚙️ Installation

**Requirements:** Python 3.9+

```bash
# Clone the repository
git clone https://github.com/your-username/wfldb-water-model.git
cd wfldb-water-model

# Create a virtual environment (recommended)
python -m venv venv
source venv/bin/activate        # Linux/macOS
venv\Scripts\activate           # Windows

# Install dependencies
pip install -r requirements.txt
```

**`requirements.txt`**
```
numpy>=1.24
pandas>=2.0
geopandas>=0.14
rasterio>=1.3
xarray>=2023.6
scipy>=1.11
matplotlib>=3.7
```

---

## 🚀 Quick Start

### Demo mode (no data files needed)

Test the equations and workflow immediately using synthetic rasters:

```bash
python wfldb_water_model.py --demo
```

Expected output:
```
============================================================
WFLDB Water Model — DEMO MODE (synthetic data)
============================================================

  EFirr (60% surface, 25% sprinkler, 15% drip): 0.5775

  Per-pixel results (centre pixel, [m³/t]):
    ETirr input                  : 412.37 m³/t
    EFirr                        : 0.5775

    Flow 1 — Withdrawal (total)  : 714.06 m³/t  →  2499.21 m³/ha
    Flow 2 — Water to air        : 412.37 m³/t  →  1443.30 m³/ha
    Flow 3 — Water to surface    : 241.34 m³/t  →  844.68 m³/ha
    Flow 4 — Water to ground     : 60.34  m³/t  →  211.19 m³/ha
    Flow 5 — Wastewater          : 8.72   m³/ha (livestock input)
    Grey water (supplementary)   : 31.40  m³/t  →  109.90 m³/ha

    Mass balance: PASSED ✓  (max err: 1.19e-07)
```

### Full model with real rasters

1. **Edit the `Config` class** in `wfldb_water_model.py` to point to your data files.
2. **Run the model:**

```bash
python wfldb_water_model.py
```

Outputs are saved as GeoTIFF files to `outputs/water_flows/`.

---

## 📥 Input Data Sources

| Parameter | Dataset | Source | Resolution |
|---|---|---|---|
| ETirr [m³/t] | Blue water footprint per crop | [Pfister et al. 2011](https://doi.org/10.1021/es903116j) | Country × crop |
| Crop harvested area | SPAM2020 | [IFPRI / HarvestChoice](https://www.mapspam.info) | 10 km |
| Yield [t/ha] | SPAM2020 or FAOSTAT gridded | [FAOSTAT](https://www.fao.org/faostat) | Country |
| Irrigation tech shares | ICID 2021 | [ICID](https://icid-ciid.org) | Country |
| Water origin (SW/GW) | Global irrigated areas | [Siebert et al. 2010](https://doi.org/10.5194/hess-14-1863-2010) | 5 arcmin |
| Admin boundaries | Natural Earth / GAUL | [Natural Earth](https://www.naturalearthdata.com) | — |
| NO₃ / PO₄ loads | WFLDB sect. 5.7–5.8 | Computed from SALCA-NO3 / P model | Derived |
| Livestock water use | FAO GLW3 | [FAO GLW](https://www.fao.org/livestock-systems/global-distributions) | 10 km |

---

## 🔬 Methodology

This model implements **WFLDB v3.10 Section 4.4** exactly:

- **Blue water** (Flows 1–4): Based on consumed water (ETirr) from Pfister et al. (2011), defined as the arithmetic mean of full-irrigation and deficit-irrigation water consumption. Irrigation losses are partitioned 80/20 between surface water and groundwater return flows.
- **Green water**: Not modelled within WFLDB scope (per section 4.4.2.1). Can be added externally as `min(ET_crop, P_effective) × area`.
- **Grey water** (supplementary): Computed using the Hoekstra et al. (2011) formula from pollutant loads derived from WFLDB Sections 5.7 (NO₃) and 5.8 (PO₄), using WHO drinking water thresholds.
- **Irrigation efficiency** (EFirr): Weighted average of surface (0.45), sprinkler (0.75), and drip (0.90) efficiency values from FAO (1989), weighted by country-level technology shares from ICID (2021).

### Irrigation efficiency values (WFLDB Table 3)

| Technique | Field efficiency (Ea) | Conveyance (Ec) | EFirr |
|---|---|---|---|
| Surface irrigation | 0.60 | 0.75 | **0.45** |
| Sprinkler irrigation | 0.75 | 1.00 | **0.75** |
| Drip / micro-irrigation | 0.90 | 1.00 | **0.90** |

---

## ✅ Model Validation

The model includes a built-in mass balance check per pixel:

```
I_withdrawal = W_air + W_surface + W_ground
```

This check is printed at runtime. A maximum error below `1e-6 m³/t` is expected from floating-point arithmetic.

---

## 📊 Output Description

All spatial outputs are GeoTIFF rasters in the same CRS and resolution as your input rasters.

| Output file | Description | Unit |
|---|---|---|
| `flow1_withdrawal_total_m3_ha.tif` | Total irrigation water extracted | m³/ha |
| `flow1a_withdrawal_surface_m3_ha.tif` | Surface water abstraction share | m³/ha |
| `flow1b_withdrawal_ground_m3_ha.tif` | Groundwater abstraction share | m³/ha |
| `flow2_water_to_air_m3_ha.tif` | Evapotranspiration (blue WF) | m³/ha |
| `flow3_water_to_surface_m3_ha.tif` | Return flow to rivers/canals | m³/ha |
| `flow4_water_to_ground_m3_ha.tif` | Percolation to aquifers | m³/ha |
| `flow5_wastewater_m3_ha.tif` | Wastewater to treatment | m³/ha |
| `grey_water_m3_ha.tif` | Grey water volume (most limiting pollutant) | m³/ha |

---

## 📚 References

- **WFLDB v3.10**: Nemecek T. et al. (2024). *Methodological Guidelines for the Life Cycle Inventory of Agricultural Products.* Version 3.10. Quantis and Agroscope, Lausanne and Zurich, Switzerland.
- **Pfister et al. (2011)**: Pfister S., Bayer P., Koehler A., Hellweg S. (2011). Projected water consumption in future global agriculture: Scenarios and related impacts. *Science of the Total Environment*.
- **Hoekstra et al. (2011)**: Hoekstra A.Y. et al. (2011). *The Water Footprint Assessment Manual.* Earthscan, London.
- **Siebert et al. (2010)**: Siebert S. et al. (2010). Groundwater use for irrigation – a global inventory. *Hydrology and Earth System Sciences*, 14, 1863–1880.
- **Lévová et al. (2012)**: Lévová T. et al. (2012). *Good practice for life cycle inventories: modelling of water use.* Ecoinvent V3.0 guidelines.
- **ISO 14046:2014**: Environmental management — Water footprint — Principles, requirements and guidelines.
- **FAO (1989)**: Irrigation water management: Irrigation scheduling. FAO Training Manual No. 4.
- **ICID (2021)**: World irrigated area with sprinkler and micro-irrigation. ICID data portal.

---

## 🤝 Contributing

Contributions are welcome. Please open an issue to discuss proposed changes before submitting a pull request.

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/add-green-water`)
3. Commit your changes (`git commit -m 'Add green water layer'`)
4. Push to the branch (`git push origin feature/add-green-water`)
5. Open a Pull Request

---

## 📄 License

This project is licensed under the MIT License. See `LICENSE` for details.

The WFLDB methodological guidelines are © Quantis and Agroscope. This model implements the published methodology for research purposes.

---

## 📬 Contact

For questions about the spatial model, open a GitHub Issue.
For questions about the WFLDB methodology, refer to the [official WFLDB documentation](https://quantis.com/who-we-guide/our-impact/our-databases/world-food-lca-database-wfldb/).
