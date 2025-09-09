# Dialogue Analytics Layer (dbt + DuckDB)

This tasks built with dbt. It focuses on **clean staging**, **incremental dimensions**, an **SCD2 snapshot** on member province, and a **fact table for events** used to compute metrics like Monthly Utilization Rate.

> For screenshots, validation SQL, and deeper notes, see the Notion doc:  
> **Project Progress and Data Verification Notes** — https://www.notion.so/Project-Progress-and-Data-Verification-Notes-2685228f921f8022a211cd5b5ac94d17?source=copy_link

---

## Quick Start

**Prereqs**
- Python 3.10+
- dbt-core 1.9.x, dbt-duckdb 1.9.x
- DuckDB database initialized with `data_warehouse`

**Setup**
```bash
# (optional) create venv
python -m venv .venv
# activate it
source .venv/bin/activate
# install project deps (pip/poetry as applicable)
# pip install -r requirements.txt  # or: poetry install
```

**Run**
```bash
# verify connection + profiles
dbt debug

# build staging → snapshot → dims → facts
dbt run -s stg
dbt snapshot
dbt run -s dim_organizations
dbt run -s fct_events
```

# tests
A manual testing has been performed which can be found in the Notion doucment with screenshots.

> Detailed test steps and sample SQL are documented in Notion (§3.4 **Testing**).

---

## Data Layout

- **Warehouse**: `data_warehouse`
- **Schemas**: `raw`, `stg`, `dim`, `fct`
- **Raw tables**: `raw.organizations`, `raw.members`, `raw.events`

> Context and row samples are captured in Notion (§1–2 **Context / About Data**).

---

## Model Overview

### Staging (`models/stg/*.sql`)
- **`stg_organizations`** — Adds `is_active` from `churned_at`; filters non-null `organization_id`.
- **`stg_members`** — Filters out null `member_id`; preserves attributes for downstream joins.
- **`stg_events`** — Extracts JSON fields; renames `timestamp` → `event_timestamp`.

> Transformation details and examples in Notion (§3.1 **Created staging models**).

### Dimensions / Snapshot
- **`dim_organizations`** — **Incremental** on `loaded_at`; keeps latest state efficiently.
- **`dim_member_snapshot`** — **SCD2 snapshot** by `province_of_residence` to preserve history for attribution.

> Rationale, example (Ontario → Quebec), and config in Notion (§3.2 **Dimensions / Snapshot**).

### Fact
- **`fct_events`** — Joins events to the valid member snapshot record (time-bounded) to attribute events to the **right org** and **province** at the event time.

> Join logic, incremental pattern, and validation queries in Notion (§3.3 **Fact models**).

---

## Metrics & Known Caveats

- **Members linked to multiple organizations** can duplicate events in `fct_events`, affecting org-level rates.
- **Event-to-program mapping gap**: events don’t directly indicate program; requires mapping to org-subscribed and member-eligible programs.

> See Notion (§4–6 **Metric Calculation Challenges / Conclusion**).

---

## Useful Commands

```bash
# Rebuild a specific model from scratch
dbt run -s dim_organizations --full-refresh

# Run only facts
dbt run -s fct_events

# Run + test a focused selection
dbt run -s stg_events+ && dbt test -s stg_events
```

---

## Project Hygiene (Best Practices)

- **Naming**: use `stg_`, `dim_`, `fct_` prefixes.
- **Incrementals & snapshots**: guard with `is_incremental()`; use reliable `loaded_at`/surrogate keys.
- **Tests**: add `unique`/`not_null` on keys; relationship tests between fact and dimensions.
- **Docs**: keep rich context (screens, SQL checks) centralized in Notion; keep this README minimal and actionable.

## Limitations & Next Steps

The **Monthly Utilization Rate** metric could not be fully completed due to data integrity issues:

- **Issue**: A single member is associated with multiple organizations.  
- **Impact**: This creates duplicate enrollments, which inflate or distort the denominator in the utilization calculation.  
- **Result**: The metric cannot reliably reflect true utilization rates until member–organization mappings are resolved.

> For background, validation checks, and discussion of these issues, see the Notion doc:  
> [Project Progress and Data Verification Notes](https://www.notion.so/Project-Progress-and-Data-Verification-Notes-2685228f921f8022a211cd5b5ac94d17?source=copy_link)

