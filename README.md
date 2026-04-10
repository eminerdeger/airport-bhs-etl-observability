# Airport BHS — Local BI & Observability Platform

A production-grade, Splunk-like data pipeline built to extract live operational data from an Oracle production database, transform and transfer it to a local Ubuntu VM, and visualise it through real-time Grafana dashboards. Deployed and validated at a major international airport Baggage Handling System (BHS) operation.

---

## Architecture

```
Oracle Production DB
        │
        ▼
extract.py — Python extraction script (Windows workstation)
  • Incremental query every 5 min (timestamp-based state files)
  • UTC → local timezone correction
  • Smart catch-up on restart (configurable lookback window)
  • gzip-compressed CSV output
        │
        ▼ SCP via paramiko
Ubuntu VM (VirtualBox) — csv_inbox/
        │
        ▼
ingest.py — Ingestion service (systemd)
  • Auto-schema creation from config.json
  • UNIQUE KEY deduplication (INSERT IGNORE)
  • Automatic row aging
  • Config hot-reload (no restart needed)
        │
        ▼
MySQL 8 — operational database
        │
        ▼
Grafana — 3 dashboards, 20+ panels (auto-refresh 30s)
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| Extraction | Python 3.13, oracledb 3.4 |
| Transfer | SCP via paramiko |
| VM | VirtualBox (headless), Ubuntu 24.04 LTS |
| Database | MySQL 8 Community |
| Ingestion service | Python 3, mysql-connector, systemd |
| Visualisation | Grafana (latest) |
| Automation | Windows Startup BAT, systemd |

---

## Data Sources

Four production tables from the Oracle database are monitored in real-time. Each table is fully configured through `config.json` — columns, computed expressions, filters, intervals, and aging rules. No code change is required to add a new table.

| Table type | Purpose |
|---|---|
| Deregistration events | Tracks baggage deregistration — identifies MISSING vs END-OF-TRACING outcomes by area |
| Registration events | Tracks baggage registration — identifies START-OF-TRACING and UNEXPECTED events by area |
| Message log | Parses structured messages — extracts type, category, flight info, and vendor group |
| Application log | System health events — WARNING, ERROR, FATAL for error pattern monitoring |

---

## Key Engineering Challenges Solved

### 1. Timezone correction
Oracle timestamps are stored as UTC. The extraction script applies a timezone offset at query time. The ingestion service converts timestamps to local time before writing to MySQL — ensuring Grafana time filters work correctly without any data misalignment.

### 2. Mixed line endings in raw message columns
Some Oracle columns contain both Windows (`\r\n`) and Unix (`\n`) line endings within the same field. All pattern-matching filters use `OR` conditions covering both variants to prevent silent misclassification.

### 3. Smart catch-up without Grafana spikes
On restart after an outage, the pipeline replays all missed 5-minute intervals in sequence. Catch-up data is inserted with the **original Oracle timestamp** — not the current timestamp — so historical data lands in its correct time slot in Grafana trend charts rather than creating a spike at restart time.

### 4. Config-driven extensibility
Adding a new Oracle table requires only a `config.json` entry (columns, filters, computed expressions, aging rules). No changes to extraction or ingestion scripts are needed. Hot-reload picks up config changes within 30 seconds without restarting any service.

### 5. Computed columns at extraction time
Some tables store structured data in a single raw TEXT column. Derived attributes are computed via SQL expressions defined in `config.json` at query time — avoiding any post-load transformation step and keeping the ingestion layer stateless.

---

## Grafana Dashboards

### Dashboard 1 — Registration & Deregistration Monitor
- Stat panels: Total registration, deregistration, and missing counts
- Donut charts: Top 15 areas by registration, deregistration, and missing counts
- Time series: Volume trends over time (colour-coded by event type)

### Dashboard 2 — Message Log Monitor
- Stat panel: Total message count (filterable by message type dropdown)
- Donut charts: Category breakdown, vendor group breakdown
- Time series: Message volume over time
- Bar chart: Top 20 items by message count
- Variable filter: filters all panels simultaneously

### Dashboard 3 — Application Log Monitor
- Stat panels: Total ERROR, WARNING, FATAL counts (colour-coded)
- Time series: ERROR / WARNING / FATAL volume over time
- Bar chart: Most frequent ERROR/FATAL message texts
- Donut chart: Error distribution by host server

---

## Automation & Resilience

| Component | Mechanism | Auto-restart |
|---|---|---|
| VirtualBox VM | Windows Startup BAT (headless) | On PC start |
| extract.py | Startup BAT with `:loop / goto loop` | Within 10s |
| ingest.py | systemd — `Restart=always, RestartSec=10` | Within 10s |
| Grafana | systemd — enabled at install | Automatic |
| MySQL | systemd — enabled at install | Automatic |

No manual steps required after PC startup. The full pipeline starts, recovers from crashes, and resumes after outages automatically.

---

## Data Accuracy Validation

Validated by running equivalent time-range queries directly against the Oracle source and comparing with Grafana panel values across all four monitored tables.

| Result | Detail |
|---|---|
| Overall accuracy | **97%+** across all tables |
| Validation method | Direct SQL comparison — Oracle source vs Grafana panel values |
| Row integrity | No drops, no duplicates confirmed |
| Residual delta (~3%) | Reflects few-second lag between Oracle query time and Grafana read — both are live systems |

---

## Configuration Reference

```json
{
  "interval_minutes": 5,
  "commit_batch_size": 5000,
  "aging_hours": 10,
  "max_lookback_hours": 24,
  "csv": { "aging_days": 7 },
  "tables": {
    "example_table": {
      "columns": ["ID", "INSERTTS", "MESSAGE", "VENDOR"],
      "computed_columns": {
        "MSG_TYPE":   "CASE WHEN ...",
        "CATEGORY":   "CASE WHEN ...",
        "ITEM_NO":    "SUBSTR(...)",
        "VENDOR_GRP": "SUBSTR(VENDOR, 1, 2)"
      },
      "where": "INSERTTS > :last_time"
    }
  }
}
```

---

## Production Recommendations

| Area | Current (PoC) | Recommended next step |
|---|---|---|
| VM lifecycle | VirtualBox on workstation | Dedicated always-on Linux server |
| Database | MySQL on VM | PostgreSQL or managed MySQL |
| Security | Credentials in config.json | Vault / environment variables |
| Scalability | Single machine, 4 tables | Containerise with Docker |
| Monitoring | Grafana dashboards | Add Grafana alerting + notification channels |
| Catch-up window | 24h max (configurable) | Extend based on SLA requirements |

---

## Context

Commissioned by management at a major international airport BHS operation to provide real-time operational visibility without impacting the production Oracle database. Designed, built, and documented end-to-end by a single engineer. Technical summary submitted to operations HQ following successful PoC validation.

**Role:** High Level Control Engineer  
**Environment:** Air-gapped production infrastructure  
**Status:** PoC validated — ready for production migration
