# CEIR Analytics

Analytics pipeline for Uganda's Central Equipment Identity Register (CEIR) data: detecting **fake**, **cloned**, and **genuine** mobile devices (IMEIs) from telecom network data, and publishing aggregated summaries for downstream reporting/BI.

> **Keep this file up to date.** When notebooks, pipeline stages, schemas, or environment requirements change, update the relevant section below in the same change.

## 1. Overview

The project ingests device (IMEI), subscriber (IMSI/MSISDN), and device-registry (GSMA TAC) data from a ClickHouse data warehouse, enriches it with KYC and country/operator lookups, classifies devices as fake / genuine / cloned, and republishes aggregated tables back to ClickHouse each month for dashboards and reporting. Regulator context: cell/call data references are sourced under UCC (Uganda Communications Commission) feeds from MTN, Airtel, and Lyca.

## 2. Data Pipeline

Three notebooks form the active monthly pipeline and are meant to be run **in numeric order**:

```
ClickHouse (ceir_gold, source layer)
        │
        ▼
1. etl_fake_clone_genuine.ipynb   — extract + enrich → monthly Parquet
        │
        ▼
   External volume: /Volumes/E$/CEIR/Clean Dumps/{Fake,Genuine,Cloned}/*.parquet
        │
        ├──────────────────────────────┐
        ▼                               ▼
2. eda_fake_clone_genuine.ipynb    3. export_fake_clone_gen.ipynb
   (exploratory analysis,             (aggregate one month → publish)
    reads full history)                       │
                                               ▼
                                  ClickHouse (ceir, published summary tables)
                                               │
                                               ▼
                                     Downstream dashboards / BI
```

### 1. `1. etl_fake_clone_genuine.ipynb` — Extract, Transform, Load

- Pulls `imeis_fake_v`, `imeis_genuine_v`, and `cloned_imeis_v` from ClickHouse database `ceir_gold` for a single target month.
- Enriches records with:
  - **KYC** (National ID) data — read from Parquet dumps on an external volume (`/Volumes/E$/KYC/Merged Clean Dumps/<year>/`), joined on `msisdn`.
  - **MNO attribution** — derived from the IMSI prefix when missing (MTN, Airtel, Hamilton, Talkio).
  - **Country** — derived from the MCC (first 3 digits of IMSI) via `clean_dumps/MCC_Each_country.csv`.
  - **GSMA device metadata** — TAC-based brand/model/OS/network-generation lookup (genuine devices only; fake IMEIs' TACs are typically unallocated, so they're intentionally *not* merged with GSMA).
- Writes one Parquet file per category per month to `/Volumes/E$/CEIR/Clean Dumps/{Fake,Genuine,Cloned}/<category>_<YYYY-MM>.parquet`.
- **Manual step required each run:** update the `MONTH = pd.Period("YYYY-MM")` parameter cell. The notebook must be re-run for every month back to August 2025 whenever ClickHouse gold tables are truncated/rebuilt — see the warning banner in the notebook.
- Connects directly to ClickHouse with a hardcoded host (`192.168.1.95`) rather than `.env` — inconsistent with notebook 3 (see [§5 Configuration](#5-configuration)).

### 2. `2. eda_fake_clone_genuine.ipynb` — Exploratory Data Analysis

- Loads and concatenates **all** historical Parquet dumps (not just one month) from the external volume for Fake, Genuine, and Cloned categories.
- Derives time-based features (year/month/hour, device age brackets) and produces breakdowns/visualizations per category (matplotlib).
- Read-only / exploratory — does not write data anywhere.

### 3. `3. export_fake_clone_gen.ipynb` — Aggregate & Publish

- Loads a single month's Fake / Genuine / Cloned Parquet dumps (`YEAR` / `MONTH` parameters at the top of the notebook).
- Builds long-format summary tables: monthly totals, breakdowns by MNO / gender / age bracket / district / country / device type / brand & model, and top-N IMEI tables.
- Publishes each table to ClickHouse database `ceir` via `clickhouse-connect`, then runs `OPTIMIZE TABLE ... FINAL` so the `ReplacingMergeTree` engine collapses duplicate rows from re-runs of the same month.
- Uses `.env` (via `python-dotenv`) for ClickHouse credentials — this is the intended pattern; see [§5 Configuration](#5-configuration).

## 3. Repository Structure

```
ceir/
├── 1. etl_fake_clone_genuine.ipynb   # Stage 1: extract/enrich/dump
├── 2. eda_fake_clone_genuine.ipynb   # Stage 2: exploratory analysis
├── 3. export_fake_clone_gen.ipynb    # Stage 3: aggregate & publish
├── clean_dumps/                      # Local reference data (gitignored)
│   ├── MCC_Each_Country.csv          #   MCC → country lookup used by all notebooks
│   └── *_90_day_call_locations.csv   #   ad hoc call-location extracts
├── dumps/                            # Raw CSV extracts (gitignored, e.g. cell files)
├── exports/                          # Scratch output folder (gitignored)
├── lib/                              # Bundled JS assets for offline pyvis/vis-network
│   ├── bindings/utils.js             #   graph rendering helpers
│   ├── tom-select/                   #   select-box widget used in exported HTML
│   └── vis-9.1.2/                    #   network graph rendering library
├── pyproject.toml                    # Python deps (managed with uv)
├── uv.lock                           # Locked dependency versions (gitignored)
├── .python-version                   # Python 3.12
└── .env                              # ClickHouse credentials (gitignored, not committed)
```

**Note on data locations:** the *monthly Parquet pipeline data* (`Fake/Genuine/Cloned` dumps and KYC dumps) lives on an external volume (`/Volumes/E$/...`), **not** in this repository. The repo's own `clean_dumps/` and `dumps/` folders hold smaller, ad hoc reference/lookup CSVs (MCC codes, cell files, call-location extracts) and are gitignored because of file size and sensitivity.

`lib/` is a leftover asset bundle for `pyvis`-rendered interactive network graphs (used by earlier network-graph notebooks). None of the three active pipeline notebooks currently import `pyvis`, `networkx`, or `geopandas` — those remain in `pyproject.toml` as declared dependencies but aren't exercised by the current pipeline.

## 4. Setup

Requires Python 3.12 and [`uv`](https://docs.astral.sh/uv/).

```bash
uv sync                 # installs dependencies from pyproject.toml / uv.lock
uv run jupyter lab       # launch JupyterLab to work with the notebooks
```

## 5. Configuration

ClickHouse connection settings are expected in a local `.env` file (gitignored, never commit this):

```
CH_HOST=
CH_PORT=
CH_DATABASE=
CH_USER=
CH_PASSWORD=
```

`3. export_fake_clone_gen.ipynb` reads these via `python-dotenv` and falls back to `localhost` / `default` if unset. `1. etl_fake_clone_genuine.ipynb` currently hardcodes its ClickHouse host/port/user in-notebook instead of using `.env` — worth aligning to the same pattern as notebook 3 if the connection details ever change.

Notebook 1 also hardcodes several filesystem paths that assume a specific machine layout (external volume mount points, KYC sibling-project location). These must be updated per-environment:

| Variable | Purpose | Example |
|---|---|---|
| `KYC_DIR` | Monthly KYC Parquet dumps | `/Volumes/E$/KYC/Merged Clean Dumps/2026` |
| `OUT_DIR` | Destination for pipeline Parquet output | `/Volumes/E$/CEIR/Clean Dumps` |
| `MCC_CSV` | MCC → country lookup | `clean_dumps/MCC_Each_country.csv` (in-repo) |
| `BASE_DIR` (notebook 3) | Source for monthly Parquet reads | `/Volumes/E$/CEIR/Clean Dumps` |

## 6. Running the Monthly Pipeline

1. Open `1. etl_fake_clone_genuine.ipynb`, set `MONTH` to the target period, and run all cells. Repeat per month if ClickHouse gold tables were truncated/rebuilt.
2. (Optional) Open `2. eda_fake_clone_genuine.ipynb` to explore trends across all months collected so far.
3. Open `3. export_fake_clone_gen.ipynb`, set `YEAR` / `MONTH` to the month you want published, and run all cells to push aggregated tables into ClickHouse `ceir`.

## 7. Conventions & Gotchas

- IMSI → MNO prefix mapping is hardcoded in both notebooks 1 and 3 (`64110`→MTN, `64101`/`64122`→Airtel, `64120`→Hamilton, `64108`→Talkio). Update both places if operator prefixes change.
- MSISDN normalization keeps only local-format numbers (10 digits, leading `0`) before converting to `256...` international format — international-format rows already present are deliberately dropped at that step in notebook 1.
- Notebook 3's `ReplacingMergeTree` + `OPTIMIZE TABLE ... FINAL` pattern means re-running the export for the same month is safe (idempotent), but relies on `updated_at` versioning — don't remove that column from the publish manifest.
