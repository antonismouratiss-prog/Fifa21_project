# FIFA 21 Player Data — ELT Pipeline on Databricks
![Databricks](https://img.shields.io/badge/Databricks-FF3621?style=flat&logo=databricks&logoColor=white)
![Delta Lake](https://img.shields.io/badge/Delta_Lake-003366?style=flat&logo=delta&logoColor=white)
![PySpark](https://img.shields.io/badge/PySpark-E25A1C?style=flat&logo=apachespark&logoColor=white)
![Python](https://img.shields.io/badge/Python-3776AB?style=flat&logo=python&logoColor=white)

End-to-end ELT pipeline for FIFA 21 player data built on Databricks using the Medallion Architecture (Bronze → Silver → Gold). The pipeline ingests raw messy data, applies a series of cleaning and transformation steps, and produces analytical aggregations visualised in a Databricks SQL Dashboard.

---

## Architecture

```
Kaggle Source (CSV)
      │
      ▼
┌─────────────┐
│   BRONZE    │  Raw ingestion — data landed as-is into Delta table
│             │  bronze_raw
└──────┬──────┘
       │
       ▼
┌─────────────┐
│   SILVER    │  Cleaning, type conversions, contract parsing
│             │  silver_cleaned
└──────┬──────┘
       │
       ▼
┌─────────────┐
│    GOLD     │  Business-level aggregations
│             │  gold_wage_value_scatter · gold_age_distribution
│             │  gold_wage_distribution · gold_value_distribution
│             │  gold_long_stay_players · gold_position_stats
└─────────────┘
```

---

## Dataset

| Property | Value |
|---|---|
| Source | [FIFA 21 Messy Raw Dataset — Kaggle](https://www.kaggle.com/datasets/yagunnersya/fifa-21-messy-raw-dataset-for-cleaning-exploring/data) |
| Format | CSV (multiline, embedded newlines) |
| Players | ~18,000 |

---

## Project Structure

```
fifa21-ELT-databricks/
│
├── README.md
├── notebooks/
│   └── fifa21_pipeline.ipynb
└── images/
    ├── wage_vs_value.png
    ├── wage_distribution.png
    ├── market_value_distribution.png
    └── age_distribution.png
```

---

## Pipeline

The pipeline runs as a single Databricks Delta Live Tables notebook using `@dp.materialized_view` decorators. Each function defines one layer of the medallion architecture and is orchestrated via Databricks Pipelines.

### Bronze — Raw Ingestion
- Reads the raw CSV using `multiLine=True` and `escape='"'` to handle embedded newlines and quoted fields
- Sanitises column names immediately to comply with Delta Lake naming requirements
- Returns the raw DataFrame untransformed

### Silver — Cleaning & Transformation

| Step | Transformation |
|---|---|
| Newline removal | Embedded `\n` characters removed from all string columns |
| Column drops | `photoUrl`, `playerUrl`, `LongName` dropped |
| Contract parsing | `Team_Contract` parsed into `TypeOfContract`, `TeamName`, `StartYear`, `EndYear` |
| Height conversion | Feet/inches (e.g. `5'11"`) → metres |
| Weight conversion | Pounds (e.g. `175lbs`) → kilograms |
| Star ratings | `★` symbol stripped from `IR`, `W_F`, `SM` and cast to integer |
| Money columns | `€XK`/`€XM` format parsed to integer for `Wage`, `Value`, `Release_Clause` |
| Type casting | `Joined` parsed to year integer, `Hits` cast to integer |

### Gold — Aggregations

| Table | Description |
|---|---|
| `gold_wage_value_scatter` | Weekly wage vs market value per player |
| `gold_age_distribution` | Player count by age |
| `gold_wage_distribution` | Player count binned by €5K wage bands |
| `gold_value_distribution` | Player count binned by €1M value bands |
| `gold_long_stay_players` | Players with 10+ years at their club |
| `gold_position_stats` | Avg wage, value and overall rating by position |

---

## Dashboard

Charts built in Databricks SQL Dashboard directly from the Gold layer tables.

### Wage vs Value
A clear non-linear relationship — the vast majority of players earn modest wages and hold low market values, while a small elite commands disproportionately high figures in both.

![Wage vs Value](images/wage_vs_value.png)

### Age Distribution
Player count peaks between ages 20–24 and follows a right-skewed distribution, with very few players active beyond 40.

![Age Distribution](images/age_distribution.png)

### Wage Distribution
Heavily right-skewed — the majority of players earn under €10K per week, with a sharp drop-off beyond that threshold.

![Wage Distribution](images/wage_distribution.png)

### Market Value Distribution
Similar skew to wages — most players are valued under €1M, with a long tail of elite players reaching beyond €100M.

![Market Value Distribution](images/market_value_distribution.png)

---

## Tech Stack

| Tool | Purpose |
|---|---|
| Databricks (Community Edition) | Pipeline orchestration and notebook environment |
| Delta Live Tables | Materialized view pipeline via `@dp.materialized_view` |
| PySpark | Distributed data processing and transformations |
| Delta Lake | ACID-compliant table format |
| Unity Catalog | Data governance and table management |
| Databricks SQL Dashboard | Analytical visualisations |
| Python | Pipeline logic |

---

## How to Run

1. Upload `fifa21_raw_data.csv` to a Unity Catalog Volume at `/Volumes/fifa21/default/fifa21_volume/`
2. Create the catalog:
```sql
CREATE CATALOG fifa21;
```
3. Import `fifa21_pipeline.ipynb` into your Databricks workspace
4. Create a Delta Live Tables pipeline pointing to the notebook
5. Click **Start** to run the full pipeline
