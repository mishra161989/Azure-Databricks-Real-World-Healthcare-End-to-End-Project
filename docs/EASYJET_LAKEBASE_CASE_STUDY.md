# easyJet on Databricks — Lakebase E2E Architecture (Simple View)

> Source: [easyJet customer story](https://www.databricks.com/customers/easyjet/lakebase)
> A 1-page reference to understand the architecture and how it maps to a Resident Solutions Architect (RSA) role.

---

## What easyJet Implemented (Summary)

easyJet rebuilt their airline **Revenue Management System (RMS)** on the Databricks platform, replacing a decade-old .NET desktop app and an overloaded SQL Server estate. They moved historical and analytical data into **Delta Lake** (bronze → silver → gold) for forecasting, reporting, and competitor benchmarking, and introduced **Lakebase** (managed Postgres) to handle live pricing transactions that must reach booking channels quickly. Trading teams now use a modern web hub built with **Databricks Apps** — a **Python API** plus a **Node.js UI** — so they can analyse fares, decide strategy, and push price changes in one place instead of juggling legacy tools. **Unity Catalog** centralised permissions, lineage, and audit, replacing uncontrolled direct database access with governed APIs and service principals. **Lakeflow Jobs** and **Lakeflow Declarative Pipelines** automated fare refreshes, demand checks, and data ingestion, consolidating **100+ scattered pipeline repos into 2** standard codebases deployed via **GitHub, Repos, and Asset Bundles**. **Databricks SQL** dashboards were added on the same gold tables for BI, replacing legacy SQL Server and Tableau-style reporting. The result: app delivery dropped from **6–9 months to ~3–4 months**, with near real-time pricing decisions and a foundation for future agentic AI assistants.

---

## 1. The Problem (Why they moved)

easyJet's Revenue Management System (RMS) ran on:
- A decade-old **.NET desktop app**.
- One of Europe's largest **SQL Server** databases — opened up to every team = no governance, slow performance, high cost.
- **100+ Git repos**, huge code sprawl.
- Migrating any feature took **6–9 months**.

Goal: one web hub where trading teams **analyse → decide → act** on pricing, in near real time.

---

## 2. The Architecture (End-to-End Flow)

```
 Pricing actions / bookings (live operational data)
            │
            ▼
   ┌───────────────────┐        ┌───────────────────────────┐
   │     LAKEBASE       │        │        DELTA LAKE          │
   │ (Managed Postgres) │        │  Bronze → Silver → Gold    │
   │ Transactions / OLTP│◄──────►│  Analytics / OLAP / history│
   └───────────────────┘        └───────────────────────────┘
            ▲    ▲                          ▲
            │    │                          │
   ┌────────┴────┴──────────────────────────┴───────────┐
   │              UNITY CATALOG (governance)             │
   │     permissions • lineage • audit • secure APIs     │
   └─────────────────────────────────────────────────────┘
            ▲                                  ▲
            │                                  │
   ┌────────┴─────────┐              ┌─────────┴──────────┐
   │  DATABRICKS APPS │              │   DATABRICKS SQL   │
   │ 1) Python API    │              │  BI dashboards on  │
   │ 2) Node UI (hub) │              │  gold tables       │
   └──────────────────┘              └────────────────────┘

   Orchestrated by: LAKEFLOW JOBS + DECLARATIVE PIPELINES
   Shipped by: GitHub + Databricks Repos + Asset Bundles (CI/CD)
```

### The components, plainly:

| Layer | Product | What it does here |
|-------|---------|-------------------|
| **Transactional (OLTP)** | **Lakebase** (managed Postgres) | Captures live pricing/booking activity, feeds updates back to booking channels. The "operational heartbeat." |
| **Analytical (OLAP)** | **Delta Lake** | Medallion: bronze (raw) → silver (refined) → gold (business-ready). Forecasting, reporting, competitor benchmarking. |
| **Governance** | **Unity Catalog** | One place for permissions, lineage, audit. Only governed APIs/service principals touch data (replaces SQL Server free-for-all). |
| **Orchestration** | **Lakeflow Jobs + Declarative Pipelines** | Auto-refresh fares near real time, run scheduled demand/competitor checks, no manual pipeline babysitting. |
| **Applications** | **Databricks Apps** | (1) Python API app linking Lakebase + Delta Lake; (2) Node UI app = the trading team's single hub. No separate CI/CD or infra. |
| **BI** | **Databricks SQL** | Dashboards on the same gold tables, replacing Tableau/SQL Server reports. |
| **CI/CD** | **Repos + Asset Bundles** | 100+ repos → 2; safe deploys, fast rollbacks. |

---

## 3. The Results

- App migration time: **6–9 months → 3–4 months**.
- **100+ Git repos → 2**.
- Governance/security centralised; uncontrolled SQL Server access eliminated.
- Near real-time decisions flowing straight into booking channels.
- Next vision: **agentic RMS** + GenAI conversational pricing.

---

## 4. Why This Matters For You as an RSA

A Resident Solutions Architect lands inside a customer and turns their goals into working Databricks architecture. This story is the *exact* template you'll repeat:

1. **Discovery → Pain mapping.** They had legacy OLTP (SQL Server), sprawl, slow delivery. As RSA you start by naming the pain in business terms (cost, time-to-market, governance), not just tech.

2. **The "Lakebase + Delta Lake" pattern is your hero pattern.** Memorize it: **Lakebase = operational/transactional**, **Delta Lake = analytical**, joined through **governed APIs**. Most modern data-app customers need exactly this OLTP+OLAP unification — you'll pitch it constantly.

3. **Lead with governance early (Unity Catalog).** The biggest mess was uncontrolled access. RSAs win trust by designing governance up front, not bolting it on.

4. **Sell outcomes, not features.** Notice every product ties to a metric (9 months → 3, 100 repos → 2). As an RSA, always frame the architecture around measurable business value.

5. **Push the full platform story.** One customer, one architecture, used Delta + Lakebase + Unity Catalog + Lakeflow + Apps + DBSQL together. RSAs drive adoption *across* the platform, not single products.

6. **Show the maturity path.** They ended pointing at agentic AI. Always leave the customer with a "what's next" roadmap.

### Your 30-second pitch to practice
> "We put live operational data in Lakebase (managed Postgres) and analytics in Delta Lake's medallion layers, govern both through Unity Catalog, automate refreshes with Lakeflow, and surface it all in Databricks Apps and SQL — so business users go from insight to action in near real time, and you ship apps in months instead of years."

---

## 5. Map to *This* Project (Healthcare E2E)

Your repo already uses the same building blocks — note the parallels:

| easyJet | This healthcare project |
|---------|-------------------------|
| Bronze/Silver/Gold | `bronze/`, `silver/`, `gold/` notebooks |
| Lakebase (Postgres) | See `docs/LAKEBASE_POSTGRES_NOTES.md` |
| Asset Bundles | `databricks.yml` |
| Unity Catalog governance | catalog/schema setup in spec |

Practicing this project = practicing the easyJet pattern end-to-end.
