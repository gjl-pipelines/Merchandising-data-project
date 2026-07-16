# Merch Data Platform — Cloud Rebuild

A cloud-native, medallion-architecture data platform that reproduces a real-world
merchandising reporting system using synthetic data. Built to demonstrate
end-to-end data engineering: ingestion of relational (CDC) and API sources,
dimensional modelling with historical tracking, and a business-facing BI layer.

> **The story:** the source system couldn't answer the business's questions —
> no price/cost history, no stock history, no cost captured at point of sale.
> This platform engineers that history and grain into the model so accurate
> margin and stock reporting becomes possible.

---

## Architecture

```
[SQL Server: synthetic merch OLTP + CDC]  ─┐
                                            ├─▶ Bronze ─────▶ Silver ─────▶ Gold ─▶ Semantic model ─▶ Power BI
[API source: enrichment (TBD)]  ───────────┘        (raw)  └── dbt ──┘  (dimensional)  (shared measures)
                                              ADF/dlt owns this    dbt owns this
```

- **Cloud:** Azure (ADLS Gen2, Data Factory / notebooks, Databricks or Fabric)
- **Ingestion:** CDC from SQL Server for incremental/changed-data-only loads; scheduled API pulls
- **Transformation:** **dbt** owns silver → gold (SQL models, tests, lineage, docs). Ingestion/bronze landing is *not* dbt — dbt is transform-after-load only.
- **Engine/adapter:** Databricks (`dbt-databricks`) — decision recorded in the progress log
- **Modelling:** star schema with SCD Type 2 and point-in-time snapshots
- **Semantic layer:** Power BI semantic model — certified shared measures (e.g. one canonical Margin definition) so every department reports off the same source of truth
- **Serve:** Power BI on the gold layer
- **Orchestration:** scheduled end-to-end run

---

## Flagship features (the hard problems)

These are the reason the project exists. Each gets its own issue and a section in the README.

| # | Real-world problem | Engineering solution to build |
|---|--------------------|-------------------------------|
| 1 | No cost / retail price history in source | **SCD Type 2** dimension tables tracking price & cost changes over time |
| 2 | No stock-level history in source | **Periodic snapshot** fact table capturing stock levels over time |
| 3 | Cost not written to sales at point of sale (legacy report did "gymnastics" for margin) | **Point-in-time cost lookup** — reference cost-history SCD in the model to capture cost as at date of sale |
| 4 | Aggregation/grain issues; SKUs with differing costs inside the same grouping | Correct **grain design** + accurate cost roll-ups that respect SKU-level cost variance |

---

## Milestones & task tracker

Tick tasks as you go. Each milestone → a GitHub Milestone; each unchecked box → a candidate Issue.

### M0 — Repo & planning  ⬜
- [ ] Create repo, commit this ROADMAP
- [ ] Add architecture diagram (draw.io / Excalidraw) to `/docs`
- [ ] Set up GitHub Milestones (M1–M6) and a Projects board
- [ ] Write the data model / grain design in `/docs` before building

### M1 — Source system: synthetic merch OLTP  ⬜  *(Weeks 1–2)*
- [ ] Design the OLTP schema (products/SKUs, stores, sales, stock, cost/price)
- [ ] Generate synthetic transactional data (Python + Faker) with realistic volume
- [ ] Stand up SQL Server (local or Azure SQL) and load it
- [ ] **Enable CDC** on the relevant tables — verify changed-data capture works
- [ ] Script incremental data changes (new sales, price/cost updates) to demo CDC

### M2 — Azure foundations & bronze ingestion  ⬜  *(Weeks 3–4)*
- [ ] Azure account + ADLS Gen2 storage set up
- [ ] Ingest CDC changes → **bronze** (changed-data-only, watermark/high-water-mark logic)
- [ ] Decide & document API source; ingest it to bronze on a schedule
- [ ] (Optional) integrate **dlt** for one of the ingestion paths
- [ ] Start DP-900 as scaffolding credential

### M3 — dbt project + silver: clean & conform  ⬜  *(Week 5)*
- [ ] `dbt init`, connect adapter, get `dbt debug` passing
- [ ] Declare bronze tables as dbt **sources** with freshness checks
- [ ] Staging models: deduplicate, type-cast, standardise keys across sources
- [ ] Conform product/store dimensions from raw
- [ ] `dbt test` — unique/not_null/relationships on every staging model
- [ ] Habit: `dbt compile` and read the rendered SQL before every `dbt run`

### M4 — Gold: the flagship modelling  ⬜  *(Weeks 6–8)*
- [ ] **Feature 1** — SCD Type 2 price & cost dims as dbt **incremental models built from the CDC log** (not `dbt snapshot` — snapshots miss intra-run changes; see progress log)
- [ ] **Feature 2** — stock snapshot fact table (dbt incremental, one row per SKU per location per day)
- [ ] **Feature 3** — point-in-time cost joined into the sales fact for margin
- [ ] **Feature 4** — correct grain + SKU-level cost roll-ups
- [ ] Build the star schema (fact + conformed dimensions)
- [ ] Custom dbt tests: no overlapping SCD2 validity windows, no gaps, exactly one current row per SKU
- [ ] **Validation a stakeholder can run**: a single readable SQL query reconciling margin to a hand-calculated sample

### M5 — Semantic layer, serve & orchestrate  ⬜  *(Weeks 9–10)*
- [ ] Build the Power BI **semantic model** over gold: relationships, hierarchies, measures
- [ ] Define **certified/shared measures** (canonical Margin, stock value, etc.) so departments share one definition
- [ ] Power BI report on the semantic model (margin, stock, trend views)
- [ ] Schedule the full pipeline end to end
- [ ] Verify a source change flows all the way to the report

### M6 — Polish for portfolio  ⬜
- [ ] README: problem → architecture → each flagship feature → screenshots
- [ ] Architecture diagram + Power BI screenshots embedded
- [ ] Clean commit history / meaningful messages
- [ ] Pin the repo; add it to CV and LinkedIn
- [ ] (Optional) DP-203 / Fabric cert

---

## Progress log

Keep a running note of decisions and blockers — great raw material for interview stories.

| Date | Milestone | Note / decision |
|------|-----------|-----------------|
| 2026-07-16 | Arch | **dbt owns silver→gold, not PySpark.** Reasons: dbt is what job ads filter on; its tests are the sign-off evidence I never get at work; and it's SQL, so my 8hrs/week go on modelling not on learning an API. Boundary: dbt is transform-after-load only — ingestion stays ADF/dlt. |
| 2026-07-16 | Arch | **No "plain SQL first, migrate later" phase.** A dbt model *is* a SELECT in a .sql file; the migration is swapping a hardcoded table name for `ref()`. Nothing to learn in the interim phase. Discipline kept instead via `dbt compile` → read rendered SQL → trace by hand → `dbt run`. |
| 2026-07-16 | Arch | **Engine = Databricks (`dbt-databricks`).** MERGE-based incrementals by default (what SCD2 + snapshot facts need), most mature adapter, strongest CV recognition. Fabric (`dbt-fabric`) was the alternative — tighter Power BI story, but warehouse-only. |
| 2026-07-16 | M4 / F1 | **SCD2 built from the CDC log, not `dbt snapshot`.** dbt snapshots diff the source at run time, so two price changes between runs = one captured, one silently lost. CDC already has every change with an LSN/timestamp. More work, but correct — and it's the reason CDC is in the architecture at all. |
