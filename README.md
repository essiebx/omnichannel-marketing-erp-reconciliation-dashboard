# Omnichannel Marketing & ERP Reconciliation Dashboard

Reconciles **Google Ads, Meta Ads and Klaviyo** performance claims against **verified Shopify and Microsoft Dynamics ERP** revenue for a simulated Nairobi omnichannel retailer ("Sandstorm"). It resolves duplicate conversion claims and timing lags, and shows **Platform-Reported ROAS vs Verified ERP ROAS**, **Customer Acquisition Cost (CAC)** and **Repeat Purchase Rate**.

> **All data in this repository is synthetic.** Results illustrate the method and the mechanisms behind attribution gaps, not real business performance.

![status](https://img.shields.io/badge/status-specification%20complete-blue)
![data](https://img.shields.io/badge/data-synthetic-lightgrey)
![license](https://img.shields.io/badge/license-MIT-green)

---

## Project status

| Area | Status |
|------|--------|
| Critique, PRD, architecture, data dictionary, logic and metric specs | Done (see `docs/`) |
| Synthetic data generator | Planned (Phase 1) |
| SQL models (staging, intermediate, marts) | Planned (Phases 2 to 4) |
| Validation against ground truth | Planned (Phase 5) |
| Dashboard (Looker Studio) | Planned (Phase 6) |
| Attribution & Data Gaps Memo | Drafted with placeholders; filled after the first full run |

The roadmap and milestones are in [docs/PROJECT_PLAN.md](docs/PROJECT_PLAN.md). Commands in "How to run" describe the target workflow and become runnable as each phase lands.

## The problem

| Source | Says | Reality |
|--------|------|---------|
| Meta, Google, Klaviyo | "We drove this order" (each with its own window) | Several platforms claim the same order |
| Shopify | Order placed | Not always fulfilled; sometimes duplicated |
| Dynamics ERP | Invoice posted; returns credited | The only verified revenue record |
| Nairobi store | Customer clicked an email Monday, bought Thursday | Invisible to digital tracking without identity capture |

## What this project delivers

1. A reproducible **synthetic data generator** with documented anomalies and a hidden ground truth.
2. A **layered SQL model** (raw, staging, intermediate, marts) on DuckDB.
3. A written, testable **attribution waterfall** with confidence labels and overlap flags.
4. A **reconciliation bridge** from platform-claimed revenue to verified net revenue that must tie out to 0 KES.
5. An **interactive dashboard** (Looker Studio, with a Power BI model spec).
6. A **one-page memo** on fixing Shopify-to-Dynamics tracking gaps.
7. A **validation report** scoring the method against the known truth.

## Documentation map

| Document | Purpose |
|----------|---------|
| [docs/PRD.md](docs/PRD.md) | Requirements, scope, acceptance criteria |
| [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) | Layers, tools, grain, identity, time, currency |
| [docs/DATA_DICTIONARY.md](docs/DATA_DICTIONARY.md) | Every table and column, source to mart |
| [docs/SYNTHETIC_DATA_SPEC.md](docs/SYNTHETIC_DATA_SPEC.md) | How data is simulated; injected anomalies |
| [docs/RECONCILIATION_LOGIC.md](docs/RECONCILIATION_LOGIC.md) | Linking, waterfall, flags, lag handling |
| [docs/METRICS_DEFINITIONS.md](docs/METRICS_DEFINITIONS.md) | ROAS, CAC, repeat rate, quality metrics |
| [docs/DASHBOARD_SPEC.md](docs/DASHBOARD_SPEC.md) | Pages, visuals, filters, measures |
| [docs/TEST_PLAN.md](docs/TEST_PLAN.md) | Data quality, scenario and validation tests |
| [docs/ATTRIBUTION_AND_DATA_GAPS_MEMO.md](docs/ATTRIBUTION_AND_DATA_GAPS_MEMO.md) | One-page executive memo |
| [docs/PROJECT_PLAN.md](docs/PROJECT_PLAN.md) | Phases, backlog, demo script |
| [CONTRIBUTING.md](CONTRIBUTING.md) | Working conventions |
| [CHANGELOG.md](CHANGELOG.md) | Version history |

## Architecture at a glance

```
Generator (Python/pandas, seeded)
   -> RAW           4 sources + reference tables
   -> STAGING       clean, normalise, hash identity, FX, VAT
   -> INTERMEDIATE  link orders, build touches, waterfall
   -> MARTS         order attribution, bridge, channel performance, customers, DQ issues
   -> BI            Looker Studio (primary) / Power BI (optional)

Ground truth -> validation report only (never read by the pipeline)
```

## Key design decisions

- The **ERP is the source of truth**; platforms are claims to verify.
- **One transformation engine** (SQL on DuckDB). Python is for generation and orchestration only.
- **Never force a match.** "Unmatched" is a reported outcome.
- **KES, net of VAT, after returns** for every verified comparison.
- **Klaviyo is reported as contribution** (verified revenue, cost per order), not as a ROAS peer to paid media.
- **B2B invoices are excluded** from ROAS and CAC but kept in the revenue tie-out.
- **No causal claims.** Incrementality testing is a listed next step.

## Core metrics

| Metric | Definition |
|--------|-----------|
| Platform-Reported ROAS | Platform conversion value / spend, as the platform shows it |
| Verified ERP ROAS | ERP-verified attributed net revenue after returns / spend |
| CAC | Paid acquisition spend / verified new customers (first ERP purchase) |
| Repeat Purchase Rate (90d) | Customers with a second fulfilled order within 90 days / matured first-order customers |

Full formulas: [docs/METRICS_DEFINITIONS.md](docs/METRICS_DEFINITIONS.md).

## Repository layout

```
.
├── README.md
├── LICENSE
├── CHANGELOG.md
├── CONTRIBUTING.md
├── requirements.txt
├── config/
│   └── default.yaml        parameters: windows, tolerances, VAT, anomaly rates
├── docs/                   specification set (this README's documentation map)
├── generator/              synthetic data simulation
├── sql/
│   ├── staging/
│   ├── intermediate/
│   ├── marts/
│   └── tests/
├── data/
│   ├── raw/                generated locally (git-ignored)
│   └── marts/              BI exports (small samples committed)
├── bi/
│   ├── looker_studio/      screenshots, field mapping, link
│   └── power_bi/           model spec, measure definitions
├── excel/                  optional Power Query parity workbook
├── reports/                audit report, validation report, memo PDF
├── tests/
└── .github/                issue and pull request templates
```

## How to run (target workflow)

Prerequisites: Python 3.11+, a Google account for Looker Studio.

```bash
# 1. Install dependencies
pip install -r requirements.txt

# 2. Generate synthetic data (seeded, reproducible)
python -m generator.run --config config/default.yaml

# 3. Build models and run blocking tests
python -m pipeline.run --config config/default.yaml

# 4. Produce audit and validation reports
python -m pipeline.report

# 5. Export marts for the BI tool
python -m pipeline.export --target looker
```

Outputs land in `data/marts/` and `reports/`. Re-running with the same seed reproduces identical headline metrics.

## Dashboard

Link: _to be added after publishing (Looker Studio, viewer access set to anyone with the link)_.
Screenshots and a PDF export will live in `bi/looker_studio/`.

## Assumptions and limitations

- Synthetic data; distributions are plausible, not empirical.
- Verified ROAS corrects counting, not causality.
- Store purchases without captured identity cannot be attributed.
- Discount-code and UTM signals are imperfect by nature.
- Orphan claims are estimated because platforms do not expose order IDs.
- Real deployments must comply with Kenya's Data Protection Act, 2019. Identity is hashed in staging and raw personal data never reaches marts.

## Roadmap

- Live connectors (Meta, Google Ads, Klaviyo, Shopify, Dynamics 365).
- Migration to BigQuery or Snowflake with dbt-style orchestration.
- Geo or audience holdout incrementality framework.
- Margin-based ROAS and customer lifetime value.
- Probabilistic identity resolution for anonymous store purchases.

## Author

Esther Naisimoi · [GitHub](https://github.com/essiebx)

## Licence

MIT. See [LICENSE](LICENSE).
