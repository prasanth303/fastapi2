# adusa_eds_po_databricks_poc

Databricks Asset Bundle that deploys a set of Delta Live Tables ingestion pipelines for purchase order files from the SEGA and BICEPS source systems.

---

## High-level flow

```
Volume (CSV files)
    └── Raw (Auto Loader)             → header_sega_raw, line_sega_raw, header_biceps_raw, line_biceps_raw
        └── Staging                   → *_staging_base / *_staging_clean / *_staging_error
            └── CDC (SCD2)            → header, line
                ├── Canonical         → canonical_kafka  (forEachBatch, fixed cluster)
                │   └── Kafka Sink    → Confluent Kafka topic
                └── Materialized View → canonical_mv  (DLT MV, serverless)
```

Reconciliation runs daily as a separate job and writes results to `recon_audit_source_to_raw`.

---

## Directory guide

| Directory | What it contains |
|---|---|
| `src/transformations/common_raw/` | Shared Auto Loader ingestion code |
| `src/transformations/staging/` | Staging transforms, quarantine logic, clean/error views |
| `src/transformations/cdc/` | SCD2 CDC flows producing header and line tables |
| `src/transformations/po_canon/` | forEachBatch canonical assembly → canonical_kafka |
| `src/transformations/mv/` | DLT materialized view → canonical_mv |
| `src/transformations/sinks/` | Confluent Kafka sink (Avro serialisation) |
| `src/utils/` | read_metadata utilities and trim_all helper |
| `src/recon/` | Source-to-raw row-count reconciliation job |
| `src/integration_tests/` | Job triggered integration tests |
| `metadata_transformations/` | YAML column-mapping files for each source/type |
| `resources/` | Bundle resource definitions (pipelines, jobs, alerts, schema) |

---

## Release target — all tables

All tables live in `eden_dev.po.*` for the release target.

### Raw

| Table | Description |
|---|---|
| `header_sega_raw` | Raw SEGA header rows |
| `header_biceps_raw` | Raw BICEPS header rows |
| `line_sega_raw` | Raw SEGA line rows |
| `line_biceps_raw` | Raw BICEPS line rows |

### Staging

| Table | Description |
|---|---|
| `header_sega_staging_base` | SEGA header staging (partitioned by `_is_quarantine`) |
| `header_sega_staging_clean` | View — clean SEGA header rows |
| `header_sega_staging_error` | View — quarantined SEGA header rows |
| `header_biceps_staging_base` | BICEPS header staging |
| `header_biceps_staging_clean` | View — clean BICEPS header rows |
| `header_biceps_staging_error` | View — quarantined BICEPS header rows |
| `line_sega_staging_base` | SEGA line staging |
| `line_sega_staging_clean` | View — clean SEGA line rows |
| `line_sega_staging_error` | View — quarantined SEGA line rows |
| `line_biceps_staging_base` | BICEPS line staging |
| `line_biceps_staging_clean` | View — clean BICEPS line rows |
| `line_biceps_staging_error` | View — quarantined BICEPS line rows |

### CDC / Canonical

| Table | Description |
|---|---|
| `header` | SCD2 canonical header (CDF enabled, CLUSTER BY AUTO) |
| `line` | SCD2 canonical line (CDF enabled, CLUSTER BY AUTO) |
| `canonical_kafka` | Nested canonical PO records (append-only Delta) |
| `canonical_mv` | Nested canonical PO records (DLT materialized view) |
| `recon_audit_source_to_raw` | Per-file source-to-raw reconciliation results |

### Event logs

| Table | Description |
|---|---|
| `event_log_sega_header_raw` | DLT event log — SEGA header raw pipeline |
| `event_log_sega_line_raw` | DLT event log — SEGA line raw pipeline |
| `event_log_biceps_header_raw` | DLT event log — BICEPS header raw pipeline |
| `event_log_biceps_line_raw` | DLT event log — BICEPS line raw pipeline |
| `event_log_header_canon` | DLT event log — header CDC pipeline |
| `event_log_line_canon` | DLT event log — line CDC pipeline |
| `event_log_canonical` | DLT event log — canonical combined pipeline |
| `event_log_canonical_mv` | DLT event log — materialized view pipeline |
| `event_log_sinks` | DLT event log — Kafka sink pipeline |

---

## Deploying

Requires the Databricks CLI and an authenticated profile.

```bash
# Validate
databricks bundle validate --target personal

# Deploy to your personal development schema
databricks bundle deploy --target personal

# Deploy to release (production-style)
databricks bundle deploy --target release
```

Or use the Makefile shortcuts (set `PROFILE` to your CLI profile name, e.g. `dev`):

```bash
make validate-personal PROFILE=dev
make deploy-personal   PROFILE=dev
make deploy-release    PROFILE=dev
```

> **Note:** `personal` uses `mode: development` — Databricks pauses all schedules automatically. Start pipelines manually in the UI after deploying.

---

## Source data directories per target

| Target | `base_file_dir` |
|---|---|
| `personal` | `/Volumes/devops/eden_test/test_data/short_csv/MODIFIED/` |
| `qa` | `/Volumes/devops/eden_test/test_data/QA/` |
| `release` | `/Volumes/devops/eden_test/retailspine_landingzone_dev_sc_po_files_volume/MODIFIED/` |

---

## Inspect pipeline configuration

```bash
databricks bundle validate --profile dev --target personal --output json \
  | jq '.. | objects | select(has("configuration")) | .configuration'
```

---

## Remote testing

See Integration Tests README.

---

## Local testing

Prerequisites: Databricks CLI installed and a `[DEFAULT]` profile set in `~/.databrickscfg`:

```ini
[DEFAULT]
host      = adb-705736162196606.6.azuredatabricks.net
auth_type = databricks-cli
```

```bash
uv sync
uv run pytest
```

See `tests/TESTING.md` for additional details.

### Windows — winutils setup

```bash
mkdir C:\hadoop\bin
curl -o C:\hadoop\bin\winutils.exe https://raw.githubusercontent.com/cdarlint/winutils/master/hadoop-3.3.6/bin/winutils.exe
curl -o C:\hadoop\bin\hadoop.dll   https://raw.githubusercontent.com/cdarlint/winutils/master/hadoop-3.3.6/bin/hadoop.dll
```

Set environment variable `HADOOP_HOME=C:\hadoop` and add `C:\hadoop\bin` to `PATH`, then:

```bash
TEST_MODE=local uv run --only-group local --python 3.11 pytest tests/ -v
```
