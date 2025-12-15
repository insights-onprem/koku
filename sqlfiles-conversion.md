# SQL Files Conversion - Trino to PostgreSQL

## Overview

This document tracks the conversion of Trino SQL files to PostgreSQL-compatible SQL files for the ONPREM deployment.

**Total files converted: 64 files**
- Root-level files: 10 files
- OpenShift files: 18 files
- AWS/Azure/GCP provider integration files: 32 files
- Subscription files: 8 files (not needed for on-prem per Luke Couzens, but converted for completeness)

**All SQL files have been successfully converted from Trino to PostgreSQL syntax.**

---

## Managed Table Creation Files

### Approach

Created parallel `postgres_sql/` folder structure alongside `trino_sql/` containing PostgreSQL-compatible versions of table creation SQL files.

**Decision**: Non-partitioned tables with indexes instead of native PostgreSQL partitioning for simplicity.

### Created Files

1. **`postgres_sql/reporting_ocpusagelineitem_daily_summary.sql`**
   - Source: `trino_sql/reporting_ocpusagelineitem_daily_summary.sql`
   - Table: `reporting_ocpusagelineitem_daily_summary`
   - Purpose: OCP usage daily summary

2. **`postgres_sql/aws/openshift/populate_daily_summary/0_prepare_daily_summary_tables.sql`**
   - Source: `trino_sql/aws/openshift/populate_daily_summary/0_prepare_daily_summary_tables.sql`
   - Tables:
     - `managed_aws_openshift_daily_temp`
     - `managed_reporting_ocpawscostlineitem_project_daily_summary_temp`
     - `managed_reporting_ocpawscostlineitem_project_daily_summary`
     - `managed_aws_openshift_disk_capacities_temp`
   - Purpose: OCP-on-AWS correlation and cost distribution

3. **`postgres_sql/azure/openshift/populate_daily_summary/0_prepare_daily_summary_tables.sql`**
   - Source: `trino_sql/azure/openshift/populate_daily_summary/0_prepare_daily_summary_tables.sql`
   - Tables:
     - `managed_azure_openshift_daily_temp`
     - `managed_azure_openshift_disk_capacities_temp`
     - `managed_reporting_ocpazurecostlineitem_project_daily_summary_temp`
     - `managed_reporting_ocpazurecostlineitem_project_daily_summary`
   - Purpose: OCP-on-Azure correlation and cost distribution

4. **`postgres_sql/gcp/openshift/populate_daily_summary/0_prepare_daily_summary_tables.sql`**
   - Source: `trino_sql/gcp/openshift/populate_daily_summary/0_prepare_daily_summary_tables.sql`
   - Tables:
     - `managed_gcp_openshift_daily_temp`
     - `managed_reporting_ocpgcpcostlineitem_project_daily_summary_temp`
     - `managed_reporting_ocpgcpcostlineitem_project_daily_summary`
   - Purpose: OCP-on-GCP correlation and cost distribution

---

## Data Type Mapping

### Trino to PostgreSQL Type Conversions

| Trino Type | PostgreSQL Type | Column Examples | Notes |
|------------|-----------------|-----------------|-------|
| `varchar` | `VARCHAR` | All string columns including `uuid`, `cluster_id`, `pod_labels`, `volume_labels`, `infrastructure_usage_cost` | Content (JSON or plain text) is irrelevant - type-to-type conversion |
| `double` | `FLOAT` | `pod_usage_cpu_core_hours`, `node_capacity_cpu_cores` | Double-precision floating point |
| `int` | `INTEGER` | `report_period_id`, `cost_category_id` | Integer values |
| `date` | `DATE` | `usage_start`, `usage_end` | Date values |

**Note:** We store JSON content as `VARCHAR` (text strings), matching Trino's approach. The third INSERT (Trino→Django PostgreSQL copy) handles conversion to JSONField, but that's not relevant for our Trino→PostgreSQL table conversion.

## Conversion Rules Applied

### 1. Table Name Format

**Trino**:
```sql
CREATE TABLE IF NOT EXISTS hive.{{schema | sqlsafe}}.table_name
```

**PostgreSQL**:
```sql
CREATE TABLE IF NOT EXISTS "{{schema}}".table_name
```

### 2. Data Type Conversions

| Trino Type | PostgreSQL Type | Notes |
|------------|----------------|-------|
| `varchar` | `VARCHAR` | No change |
| `double` | `FLOAT` | Matches Trino's double-precision floating point behavior |
| `int` | `INTEGER` | No change |
| `integer` | `INTEGER` | No change |
| `boolean` | `BOOLEAN` | No change |
| `timestamp(3)` | `TIMESTAMP` | Removed precision, PostgreSQL handles milliseconds |
| `timestamp` | `TIMESTAMP` | No change |
| `date` | `DATE` | No change |
| `varchar` (for large text) | `TEXT` | For columns like `tags`, `labels`, `pod_labels`, `volume_labels`, `aws_cost_category`, `system_labels` |

### 3. Partitioning Changes

**Trino** (removed):
```sql
WITH(format = 'PARQUET', partitioned_by=ARRAY['source', 'year', 'month', 'day'])
```

**PostgreSQL** (partition columns become regular columns):
```sql
-- No WITH clause
-- Columns source, year, month, day are regular columns
```

### 4. Indexes Added

For each table, added indexes on commonly queried columns:

```sql
-- Partition-like filtering (replaces partition pruning)
CREATE INDEX IF NOT EXISTS idx_tablename_source_year_month
    ON "{{schema}}".table_name (source, year, month);

-- Or for multi-source tables:
CREATE INDEX IF NOT EXISTS idx_tablename_source_year_month
    ON "{{schema}}".table_name (source, ocp_source, year, month);

-- Day filtering
CREATE INDEX IF NOT EXISTS idx_tablename_day
    ON "{{schema}}".table_name (day);

-- Temporal queries
CREATE INDEX IF NOT EXISTS idx_tablename_usage_start
    ON "{{schema}}".table_name (usage_start);
```

---

## Rationale for Decisions

### Why FLOAT instead of NUMERIC?

PostgreSQL's `FLOAT` (double precision) matches Trino's `double` behavior exactly. While `NUMERIC` provides arbitrary precision which is better for financial calculations, using `FLOAT` ensures:
1. **Same calculation results** as Trino
2. **Same precision/rounding behavior** as existing Trino queries
3. **Better performance** than NUMERIC
4. **Consistency** with reportdb_accessor_postgres implementation

### Why TEXT for large string fields?

Fields like `tags`, `labels`, `pod_labels`, etc. can contain JSON or large amounts of text. PostgreSQL's `TEXT` type is more appropriate than `VARCHAR` for these use cases.

### Why no partitioning?

1. **Simplicity**: No need to manage partition creation/deletion dynamically
2. **Smaller data volumes**: ONPREM deployments expected to have smaller data than multi-tenant SaaS
3. **Indexes provide adequate performance**: B-tree indexes on `(source, year, month)` provide good query performance
4. **Easy migration path**: Can add partitioning later if performance requires it

### Why indexes on source, year, month?

These columns were partition keys in Trino. Queries filter by these columns frequently, so indexes maintain similar query performance without the complexity of partitioning.

---

## Verification Against TRINO_MANAGED_TABLES

Compared created SQL files against `TRINO_MANAGED_TABLES` in `koku/reporting/models.py`:

**Result**: ✓ **MATCH** - All 12 tables are accounted for

### Tables Created (12 total):

1. `reporting_ocpusagelineitem_daily_summary` (source column: `source`)
2. `managed_aws_openshift_daily_temp` (source column: `ocp_source`)
3. `managed_aws_openshift_disk_capacities_temp` (source column: `ocp_source`)
4. `managed_reporting_ocpawscostlineitem_project_daily_summary_temp` (source column: `ocp_source`)
5. `managed_reporting_ocpawscostlineitem_project_daily_summary` (source column: `ocp_source`)
6. `managed_azure_openshift_daily_temp` (source column: `ocp_source`)
7. `managed_azure_openshift_disk_capacities_temp` (source column: `ocp_source`)
8. `managed_reporting_ocpazurecostlineitem_project_daily_summary_temp` (source column: `ocp_source`)
9. `managed_reporting_ocpazurecostlineitem_project_daily_summary` (source column: `ocp_source`)
10. `managed_gcp_openshift_daily_temp` (source column: `ocp_source`)
11. `managed_reporting_ocpgcpcostlineitem_project_daily_summary_temp` (source column: `ocp_source`)
12. `managed_reporting_ocpgcpcostlineitem_project_daily_summary` (source column: `ocp_source`)

**Note**: The source column mapping indicates which column is used for partition deletion when a source is removed. This is important for the `delete_hive_partitions_by_source()` method.

---

## Understanding WITH Clauses in Trino SQL

The `WITH` keyword serves two different purposes in the Trino SQL files:

### 1. Table Properties (in CREATE TABLE statements)

**Trino**:
```sql
CREATE TABLE IF NOT EXISTS hive.{{schema | sqlsafe}}.managed_aws_openshift_daily_temp
(
    ...
) WITH(format = 'PARQUET', partitioned_by=ARRAY['source', 'ocp_source', 'year', 'month', 'day']);
```

This `WITH` clause specifies **table properties** for Trino/Hive:
- `format = 'PARQUET'` - Store data in Parquet columnar format
- `partitioned_by=ARRAY[...]` - Define partition columns for physical data layout

**PostgreSQL**: ✅ Already removed - PostgreSQL doesn't use this syntax

### 2. Common Table Expressions (CTEs) in INSERT statements

**Trino**:
```sql
INSERT INTO hive.{{schema | sqlsafe}}.reporting_ocpusagelineitem_daily_summary (...)
WITH cte_pg_enabled_keys as (
    select array['vm_kubevirt_io_name'] || array_agg(key order by key) as keys
      from postgres.{{schema | sqlsafe}}.reporting_enabledtagkeys
     where enabled = true
     and provider_type = 'OCP'
),
cte_ocp_node_label_line_item_daily AS (
    SELECT date(nli.interval_start) as usage_start,
        ...
    FROM hive.{{schema | sqlsafe}}.openshift_node_labels_line_items_daily AS nli
    ...
)
SELECT ... FROM ...
```

This `WITH` clause defines **Common Table Expressions (CTEs)** - temporary named result sets that can be referenced in the main query.

**PostgreSQL**: ✅ **CTE syntax is standard SQL** - PostgreSQL supports WITH...AS for CTEs natively. The keyword itself needs no conversion.

### What Needs Conversion in INSERT Statements

While the `WITH` keyword and CTE structure work in PostgreSQL, the following Trino-specific features need conversion:

#### 1. Trino-Specific Functions

| Trino Function | PostgreSQL Equivalent | Notes |
|----------------|----------------------|-------|
| `map_filter(map, lambda)` | JSONB operators + `jsonb_object_agg()` | PostgreSQL uses JSONB instead of MAP type |
| `map_concat(map1, map2, ...)` | `jsonb1 \|\| jsonb2 \|\| ...` | JSONB concatenation operator |
| `json_parse(string)` | `string::jsonb` | Type casting |
| `json_format(json)` | `jsonb::text` or `to_json()` | Convert JSONB back to text |
| `date_add('day', n, date)` | `date + INTERVAL 'n days'` | Standard SQL interval arithmetic |
| `contains(array, value)` | `value = ANY(array)` | PostgreSQL array membership |
| `array_agg(key order by key)` | `array_agg(key ORDER BY key)` | Syntax difference (uppercase ORDER BY) |
| `cast(x as map(varchar, varchar))` | `x::jsonb` | PostgreSQL uses JSONB for key-value structures |

#### 2. Table References

| Trino | PostgreSQL |
|-------|------------|
| `hive.{{schema \| sqlsafe}}.table_name` | `"{{schema}}".table_name` |
| `postgres.{{schema \| sqlsafe}}.table_name` | `"{{schema}}".table_name` |

#### 3. Other Syntax Differences

- **Array literals**: `ARRAY['a', 'b']` → `ARRAY['a', 'b']` (same syntax, works in both)
- **Array concatenation**: `array1 \|\| array2` (same in both)
- **CROSS JOIN**: Works identically in both
- **LEFT JOIN**: Works identically in both
- **GROUP BY ordinals**: `GROUP BY 3, 6` (works in both, but PostgreSQL also supports column names)

---

## Complete Inventory of Trino SQL Files by Query Type

### Files with **CREATE TABLE** only (✅ Already Converted):
- `trino_sql/aws/openshift/populate_daily_summary/0_prepare_daily_summary_tables.sql`
- `trino_sql/azure/openshift/populate_daily_summary/0_prepare_daily_summary_tables.sql`
- `trino_sql/gcp/openshift/populate_daily_summary/0_prepare_daily_summary_tables.sql`

### Files with **DELETE + INSERT** (need conversion):

#### Resource Matching Files:
- `trino_sql/aws/openshift/populate_daily_summary/1_resource_matching_by_cluster.sql`
  - DELETE from `managed_aws_openshift_daily_temp`
  - INSERT with CTEs for resource matching (AWS resources to OCP nodes)
- `trino_sql/azure/openshift/populate_daily_summary/1_resource_matching_by_cluster.sql`
  - DELETE from `managed_azure_openshift_daily_temp`
  - INSERT with CTEs for resource matching
- `trino_sql/gcp/openshift/populate_daily_summary/1_resource_matching_by_cluster.sql`
  - DELETE from `managed_gcp_openshift_daily_temp`
  - INSERT with CTEs for resource matching

#### Data Summarization Files:
- `trino_sql/aws/openshift/populate_daily_summary/2_summarize_data_by_cluster.sql`
  - DELETE from `managed_aws_openshift_disk_capacities_temp` (conditional)
  - INSERT disk capacity calculations
  - DELETE from `managed_reporting_ocpawscostlineitem_project_daily_summary_temp`
  - INSERT cost summarization with CTEs
  - DELETE from `managed_reporting_ocpawscostlineitem_project_daily_summary`
  - INSERT final aggregated data
- `trino_sql/azure/openshift/populate_daily_summary/2_summarize_data_by_cluster.sql`
  - Similar pattern for Azure
- `trino_sql/gcp/openshift/populate_daily_summary/2_summarize_data_by_cluster.sql`
  - Similar pattern for GCP

### Files with **INSERT** only (need conversion):

#### OCP Usage Summary:
- `trino_sql/reporting_ocpusagelineitem_daily_summary.sql`
  - INSERT pod and storage usage (lines 51-450)
  - INSERT unallocated capacity (lines 461-581)
  - INSERT to PostgreSQL table (lines 583-667)

#### OCP-on-Cloud Daily Summary (Copy to PostgreSQL):
- `trino_sql/aws/openshift/populate_daily_summary/3_reporting_ocpawscostlineitem_project_daily_summary_p.sql`
  - INSERT from managed table to PostgreSQL partitioned table
- `trino_sql/azure/openshift/populate_daily_summary/3_reporting_ocpazurecostlineitem_project_daily_summary_p.sql`
- `trino_sql/gcp/openshift/populate_daily_summary/3_reporting_ocpgcpcostlineitem_project_daily_summary_p.sql`

#### OCP-on-Cloud Summary Tables (Copy to PostgreSQL):
- `trino_sql/aws/openshift/reporting_ocpawscostlineitem_project_daily_summary_p.sql`
- `trino_sql/azure/openshift/reporting_ocpazurecostlineitem_project_daily_summary_p.sql`
- `trino_sql/gcp/openshift/reporting_ocpgcpcostlineitem_project_daily_summary_p.sql`

#### OCP-on-Cloud Aggregated Summaries:
- `trino_sql/aws/openshift/reporting_ocpaws_cost_summary_p.sql` (aggregate by cluster/day)
- `trino_sql/aws/openshift/reporting_ocpaws_cost_summary_by_account_p.sql`
- `trino_sql/aws/openshift/reporting_ocpaws_cost_summary_by_region_p.sql`
- `trino_sql/aws/openshift/reporting_ocpaws_cost_summary_by_service_p.sql`
- `trino_sql/aws/openshift/reporting_ocpaws_compute_summary_p.sql`
- `trino_sql/aws/openshift/reporting_ocpaws_database_summary_p.sql`
- `trino_sql/aws/openshift/reporting_ocpaws_network_summary_p.sql`
- `trino_sql/aws/openshift/reporting_ocpaws_storage_summary_p.sql`
- Similar files for Azure and GCP (replace `aws` with `azure` or `gcp`)

#### Cloud Provider Daily Summaries (non-OCP):
- `trino_sql/reporting_awscostentrylineitem_daily_summary.sql`
- `trino_sql/reporting_azurecostentrylineitem_daily_summary.sql`
- `trino_sql/reporting_gcpcostentrylineitem_daily_summary.sql`

#### Other Summary Files:
- `trino_sql/reporting_awscostentrylineitem_summary_by_ec2_compute_p.sql`

### Files with **SELECT** only (queries that return data, no INSERT/DELETE):
- `trino_sql/reporting_ocpaws_matched_tags.sql`
  - Pure SELECT query with CTEs for tag matching
- `trino_sql/reporting_ocpazure_matched_tags.sql`
- `trino_sql/gcp/openshift/reporting_ocpgcp_matched_tags.sql`
- `trino_sql/ocp_special_matched_tags.sql`
- `trino_sql/aws/reporting_ocpinfrastructure_provider_map.sql`
  - SELECT query mapping OCP resources to AWS infrastructure
- `trino_sql/azure/reporting_ocpinfrastructure_provider_map.sql`
- `trino_sql/gcp/reporting_ocpinfrastructure_provider_map.sql`

### Cost Model Files (OCP VM costing):
- `trino_sql/openshift/cost_model/hourly_cost_virtual_machine.sql`
- `trino_sql/openshift/cost_model/hourly_cost_vm_tag_based.sql`
- `trino_sql/openshift/cost_model/hourly_vm_core.sql`
- `trino_sql/openshift/cost_model/hourly_vm_core_tag_based.sql`
- `trino_sql/openshift/cost_model/monthly_project_tag_based.sql`
- `trino_sql/openshift/cost_model/monthly_vm_core.sql`
- `trino_sql/openshift/cost_model/monthly_vm_core_tag_based.sql`
- `trino_sql/openshift/populate_vm_tmp_table.sql`
- `trino_sql/openshift/populate_vm_tmp_table_with_vm_report.sql`

### Test Files:
- `trino_sql/test/ocp/mimic_remove_disabled_tags.sql`
- `trino_sql/test/ocp/mimic_virt_ui.sql`

---

## Complete Table Inventory and Name Collision Analysis

### All Hive Tables (22 tables)

| Table Name | Type | Used In |
|------------|------|---------|
| `aws_line_items` | Line items | AWS daily processing |
| `aws_line_items_daily` | Line items | AWS daily processing |
| `azure_line_items` | Line items | Azure daily processing |
| `gcp_line_items_daily` | Line items | GCP daily processing |
| `openshift_namespace_labels_line_items_daily` | Line items | OCP processing |
| `openshift_node_labels_line_items_daily` | Line items | OCP processing |
| `openshift_pod_usage_line_items` | Line items | OCP processing |
| `openshift_pod_usage_line_items_daily` | Line items | OCP processing |
| `openshift_storage_usage_line_items_daily` | Line items | OCP processing |
| `openshift_vm_usage_line_items` | Line items | OCP VM processing |
| `openshift_vm_usage_line_items_daily` | Line items | OCP VM processing |
| `managed_aws_openshift_daily_temp` | Managed/temp | OCP-on-AWS correlation |
| `managed_aws_openshift_disk_capacities_temp` | Managed/temp | OCP-on-AWS storage |
| `managed_azure_openshift_daily_temp` | Managed/temp | OCP-on-Azure correlation |
| `managed_azure_openshift_disk_capacities_temp` | Managed/temp | OCP-on-Azure storage |
| `managed_gcp_openshift_daily_temp` | Managed/temp | OCP-on-GCP correlation |
| `managed_reporting_ocpawscostlineitem_project_daily_summary` | Managed/summary | OCP-on-AWS summary |
| `managed_reporting_ocpawscostlineitem_project_daily_summary_temp` | Managed/temp | OCP-on-AWS temp |
| `managed_reporting_ocpazurecostlineitem_project_daily_summary` | Managed/summary | OCP-on-Azure summary |
| `managed_reporting_ocpazurecostlineitem_project_daily_summary_temp` | Managed/temp | OCP-on-Azure temp |
| `managed_reporting_ocpgcpcostlineitem_project_daily_summary` | Managed/summary | OCP-on-GCP summary |
| `managed_reporting_ocpgcpcostlineitem_project_daily_summary_temp` | Managed/temp | OCP-on-GCP temp |
| **`reporting_ocpusagelineitem_daily_summary`** | **Managed/summary** | **OCP usage summary - NAME COLLISION!** |

### All PostgreSQL Tables Referenced in Trino SQL (32 tables)

| Table Name | Type | Used In |
|------------|------|---------|
| `reporting_awsaccountalias` | Lookup | AWS processing |
| `reporting_awscostentrylineitem_daily_summary` | Summary | AWS summaries |
| `reporting_awscostentrylineitem_summary_by_ec2_compute_p` | Summary | AWS EC2 |
| `reporting_awsorganizationalunit` | Lookup | AWS processing |
| `reporting_azurecostentrylineitem_daily_summary` | Summary | Azure summaries |
| `reporting_enabledtagkeys` | Lookup | Tag filtering |
| `reporting_gcpcostentrylineitem_daily_summary` | Summary | GCP summaries |
| `reporting_ocp_cost_category_namespace` | Lookup | OCP cost categories |
| `reporting_ocp_nodes` | Lookup | OCP node info |
| `reporting_ocpaws_compute_summary_p` | Summary | OCP-on-AWS compute |
| `reporting_ocpaws_cost_summary_by_account_p` | Summary | OCP-on-AWS by account |
| `reporting_ocpaws_cost_summary_by_region_p` | Summary | OCP-on-AWS by region |
| `reporting_ocpaws_cost_summary_by_service_p` | Summary | OCP-on-AWS by service |
| `reporting_ocpaws_cost_summary_p` | Summary | OCP-on-AWS cost |
| `reporting_ocpaws_database_summary_p` | Summary | OCP-on-AWS database |
| `reporting_ocpaws_network_summary_p` | Summary | OCP-on-AWS network |
| `reporting_ocpaws_storage_summary_p` | Summary | OCP-on-AWS storage |
| `reporting_ocpawscostlineitem_project_daily_summary_p` | Summary | OCP-on-AWS daily |
| `reporting_ocpazure_compute_summary_p` | Summary | OCP-on-Azure compute |
| `reporting_ocpazure_cost_summary_by_account_p` | Summary | OCP-on-Azure by account |
| `reporting_ocpazure_cost_summary_by_location_p` | Summary | OCP-on-Azure by location |
| `reporting_ocpazure_cost_summary_by_service_p` | Summary | OCP-on-Azure by service |
| `reporting_ocpazure_cost_summary_p` | Summary | OCP-on-Azure cost |
| `reporting_ocpazure_database_summary_p` | Summary | OCP-on-Azure database |
| `reporting_ocpazure_network_summary_p` | Summary | OCP-on-Azure network |
| `reporting_ocpazure_storage_summary_p` | Summary | OCP-on-Azure storage |
| `reporting_ocpazurecostlineitem_project_daily_summary_p` | Summary | OCP-on-Azure daily |
| `reporting_ocpgcp_compute_summary_p` | Summary | OCP-on-GCP compute |
| `reporting_ocpgcp_cost_summary_by_account_p` | Summary | OCP-on-GCP by account |
| `reporting_ocpgcp_cost_summary_by_gcp_project_p` | Summary | OCP-on-GCP by project |
| `reporting_ocpgcp_cost_summary_by_region_p` | Summary | OCP-on-GCP by region |
| `reporting_ocpgcp_cost_summary_by_service_p` | Summary | OCP-on-GCP by service |
| `reporting_ocpgcp_cost_summary_p` | Summary | OCP-on-GCP cost |
| `reporting_ocpgcp_database_summary_p` | Summary | OCP-on-GCP database |
| `reporting_ocpgcp_network_summary_p` | Summary | OCP-on-GCP network |
| `reporting_ocpgcp_storage_summary_p` | Summary | OCP-on-GCP storage |
| `reporting_ocpgcpcostlineitem_project_daily_summary_p` | Summary | OCP-on-GCP daily |
| **`reporting_ocpusagelineitem_daily_summary`** | **Summary** | **OCP usage summary - NAME COLLISION!** |
| `reporting_tenant_api_provider` | Lookup | Provider info |
| `tmp_virt_*` | Temp | VM processing |

### Name Collision Identified

**Only 1 collision found:**

| Table Name | Hive Reference | PostgreSQL Reference | Status |
|------------|----------------|---------------------|--------|
| `reporting_ocpusagelineitem_daily_summary` | `hive.{{schema}}.reporting_ocpusagelineitem_daily_summary` | `postgres.{{schema}}.reporting_ocpusagelineitem_daily_summary` | **COLLISION** |

**All other tables are unique** - Hive tables are different from PostgreSQL tables by name.

**Note on Schema:** All tables use `{{schema}}` template variable - no tables found in public/default schema in the SQL files.

---

## Table Renaming: Hive/Trino Managed Table

### Issue

In ONPREM mode, we use PostgreSQL for everything (both Django tables and managed tables). This creates a name collision:
- Django table: `"{schema}".reporting_ocpusagelineitem_daily_summary` (exists in PostgreSQL)
- Managed table (from Trino migration): `"{schema}".reporting_ocpusagelineitem_daily_summary` (would also be in PostgreSQL)

Same database, same schema, same name = COLLISION

### Solution

**Conditional table naming based on `settings.ONPREM`:**

- **Non-ONPREM (Trino)**: Use `reporting_ocpusagelineitem_daily_summary` (no suffix)
  - Trino SQL files keep original name (no changes to trino_sql folder)
  - No breaking changes to existing Trino deployments

- **ONPREM (PostgreSQL)**: Use `reporting_ocpusagelineitem_daily_summary_trino` (with suffix)
  - PostgreSQL SQL files use `_trino` suffix to avoid collision
  - Code looks for table with suffix when ONPREM=True

### Files Changed

#### SQL Files:

**Trino SQL files (7 files)**: ✅ **NO CHANGES** - Keep original name `reporting_ocpusagelineitem_daily_summary`
- `koku/masu/database/trino_sql/reporting_ocpusagelineitem_daily_summary.sql`
- `koku/masu/database/trino_sql/aws/openshift/populate_daily_summary/2_summarize_data_by_cluster.sql`
- `koku/masu/database/trino_sql/azure/openshift/populate_daily_summary/2_summarize_data_by_cluster.sql`
- `koku/masu/database/trino_sql/gcp/openshift/populate_daily_summary/2_summarize_data_by_cluster.sql`
- `koku/masu/database/trino_sql/reporting_ocpaws_matched_tags.sql`
- `koku/masu/database/trino_sql/reporting_ocpazure_matched_tags.sql`
- `koku/masu/database/trino_sql/gcp/openshift/reporting_ocpgcp_matched_tags.sql`

**PostgreSQL SQL file (1 file)**: ✅ **CREATED** with `_trino` suffix
- `koku/masu/database/postgres_sql/reporting_ocpusagelineitem_daily_summary.sql`
  - Table name: `reporting_ocpusagelineitem_daily_summary_trino`

#### Python Files:

**`koku/reporting/models.py`**: ✅ **UPDATED** with conditional table naming

Added conditional logic based on `settings.ONPREM`:

```python
from django.conf import settings

if getattr(settings, 'ONPREM', False):
    TRINO_MANAGED_TABLES = {
        "reporting_ocpusagelineitem_daily_summary_trino": "source",
        # ... rest of tables
    }
else:
    TRINO_MANAGED_TABLES = {
        "reporting_ocpusagelineitem_daily_summary": "source",  # no suffix for Trino
        # ... rest of tables
    }

if getattr(settings, 'ONPREM', False):
    EXPIRE_MANAGED_TABLES = {
        "reporting_ocpusagelineitem_daily_summary_trino": "source",
    }
else:
    EXPIRE_MANAGED_TABLES = {
        "reporting_ocpusagelineitem_daily_summary": "source",  # no suffix for Trino
    }
```

**Other files**: No changes needed - they import `TRINO_MANAGED_TABLES` dict, so changes propagate automatically:
- `koku/masu/celery/tasks.py`
- `koku/masu/processor/ocp/ocp_report_db_cleaner.py`
- `koku/masu/management/commands/migrate_trino_tables.py`
- `koku/masu/test/celery/test_tasks.py`

### Summary

**Total files modified: 2 files**
- ✅ 1 PostgreSQL SQL file created with `_trino` suffix
- ✅ 1 Python file updated with conditional table naming (`koku/reporting/models.py`)
- ✅ 0 Trino SQL files modified (no breaking changes to existing Trino deployments)

---

## Root-Level Files Converted

### Files Converted: ✅ **10 files completed**

1. `reporting_ocpusagelineitem_daily_summary.sql` - OCP usage summary (CREATE TABLE + 3 INSERTs)
2. `reporting_ocpaws_matched_tags.sql` - AWS/OCP tag matching (SELECT)
3. `reporting_ocpazure_matched_tags.sql` - Azure/OCP tag matching (SELECT)
4. `gcp/openshift/reporting_ocpgcp_matched_tags.sql` - GCP/OCP tag matching (SELECT)
5. `ocp_special_matched_tags.sql` - OCP special tags (SELECT)
6. `reporting_awscostentrylineitem_daily_summary.sql` - AWS daily summary (INSERT)
7. `reporting_azurecostentrylineitem_daily_summary.sql` - Azure daily summary (INSERT)
8. `reporting_gcpcostentrylineitem_daily_summary.sql` - GCP daily summary (INSERT)
9. `reporting_awscostentrylineitem_summary_by_ec2_compute_p.sql` - AWS EC2 compute summary (INSERT)
10. Table creation files: `aws/azure/gcp/openshift/populate_daily_summary/0_prepare_daily_summary_tables.sql`

### Matched Tags Query Conversion

#### File: `reporting_ocpaws_matched_tags.sql` ✅ CONVERTED

Pure SELECT query (58 lines) that finds matching tags between AWS and OCP resources.

**Conversions Applied:**

**CTE 1: `cte_enabled_tag_keys`**
- Removed `postgres.` prefix from table reference

**CTE 2: `cte_unnested_aws_tags`**
- **Important**: `UNNEST(cast(json_parse(resourcetags) as map(varchar, varchar)))` → `json_each_text(resourcetags::json)` with `LATERAL`
- **Important**: `any_match(etk.key_array, x->strpos(aws.resourcetags, x) != 0)` → `EXISTS (SELECT 1 FROM unnest(etk.key_array) AS enabled_key WHERE strpos(aws.resourcetags, enabled_key) != 0)`
- Converted `date_add('day', 1, {{end_date}})` → `{{end_date}} + INTERVAL '1 day'`
- Removed `hive.` prefix

**CTE 3: `cte_unnested_ocp_tags`**
- **Important**: Dual `UNNEST(map1, map2)` → Two separate `CROSS JOIN LATERAL json_each_text()` calls
  ```sql
  -- Trino: UNNEST(cast(json_parse(pod_labels) as map(...)), cast(json_parse(volume_labels) as map(...)))
  -- PostgreSQL:
  CROSS JOIN LATERAL json_each_text(COALESCE(pod_labels::json, '{}'::json)) AS pod_tags(...)
  CROSS JOIN LATERAL json_each_text(COALESCE(volume_labels::json, '{}'::json)) AS volume_tags(...)
  ```
- Added `COALESCE(..., '{}'::json)` for NULL handling
- `any_match()` with OR → `EXISTS` with OR condition
- Removed `hive.` prefix

**Final SELECT:**
- No changes needed (standard SQL)

**Key Learning: LATERAL and json_each_text()**

- `LATERAL` allows functions/subqueries in FROM clause to reference columns from earlier tables
- `json_each_text(json_column::json)` expands JSON objects to rows in PostgreSQL (replaces Trino's UNNEST with maps)
- `EXISTS (SELECT 1 FROM unnest(array))` replaces Trino's `any_match()` lambda function

#### File: `reporting_ocpazure_matched_tags.sql` ✅ CONVERTED

Pure SELECT query (50 lines) that finds matching tags between Azure and OCP resources. Simpler than AWS version - no `any_match()` lambda.

**Conversions Applied:**

**CTE 1: `cte_unnested_azure_tags`**
- Removed `hive.` prefix
- `UNNEST(cast(json_parse(tags) as map(varchar, varchar)))` → `json_each_text(azure.tags::json)` with `LATERAL`
- Converted `date_add('day', 1, {{end_date}})` → `{{end_date}} + INTERVAL '1 day'`

**CTE 2: `cte_unnested_ocp_tags`**
- Same as AWS version - dual UNNEST to two LATERAL json_each_text calls with COALESCE
- Removed `hive.` prefix

**Final SELECT:**
- Removed `postgres.` prefix from two JOINs to `reporting_enabledtagkeys`

#### File: `gcp/openshift/reporting_ocpgcp_matched_tags.sql` ✅ CONVERTED

Pure SELECT query (50 lines) that finds matching tags between GCP and OCP resources. Identical pattern to Azure version.

**Conversions Applied:**

**CTE 1: `cte_unnested_gcp_tags`**
- Removed `hive.` prefix
- `UNNEST(cast(json_parse(labels) as map(varchar, varchar)))` → `json_each_text(gcp.labels::json)` with `LATERAL`
- Converted `date_add('day', 1, {{start_date}})` → `{{start_date}} + INTERVAL '1 day'`

**CTE 2: `cte_unnested_ocp_tags`**
- Same as AWS/Azure versions - dual UNNEST to two LATERAL json_each_text calls with COALESCE
- Removed `hive.` prefix

**Final SELECT:**
- Removed `postgres.` prefix from two JOINs to `reporting_enabledtagkeys`
- Provider type: 'GCP'

#### File: `ocp_special_matched_tags.sql` ✅ CONVERTED

Pure SELECT query (45 lines) that builds OCP-specific tags (cluster, node, project/namespace tags).

**Conversions Applied:**

**CTE 1: `cte_array_agg_nodes`**
- Removed `hive.` prefix
- Converted `date_add('day', 1, {{end_date}})` → `{{end_date}} + INTERVAL '1 day'`

**CTE 2: `cte_cluster_info`**
- **Important**: Converted `json_extract_scalar(auth.credentials, '$.cluster_id')` → `auth.credentials::json->>'cluster_id'`
- Removed `postgres.public.` prefix, used `public.` directly
- Converted `CAST({{ocp_provider_uuid}} as UUID)` → `{{ocp_provider_uuid}}::uuid`
- Kept `format()` function (works in both Trino and PostgreSQL)

**CTE 3: `cte_tag_matches`**
- 5 SELECT queries combined with UNION (no syntax change needed for UNION)
- Removed `hive.` prefix from last SELECT
- Converted `date_add('day', 1, {{end_date}})` → `{{end_date}} + INTERVAL '1 day'`

**Final SELECT:**
- No changes needed (`array_agg()` works identically)

**Key Learning: PostgreSQL JSON operators**

- Trino: `json_extract_scalar(json_col, '$.key')`
- PostgreSQL: `json_col::json->>'key'` (no `$.` prefix needed)

#### File: `reporting_awscostentrylineitem_daily_summary.sql` ✅ CONVERTED

INSERT query (156 lines) that aggregates AWS daily line items and inserts into PostgreSQL summary table.

**Conversions Applied:**

**INSERT statement:**
- Removed `postgres.` prefix: `INSERT INTO postgres.{{schema}}` → `INSERT INTO {{schema}}`

**CTE: `cte_pg_enabled_keys`**
- Removed `postgres.` prefix from table reference

**Main SELECT conversions:**
- `uuid()` → `uuid_generate_v4()`
- **Important**: `cast(map_filter(cast(json_parse(tags) as map(varchar, varchar)), (k,v) -> contains(pek.keys, k)) as json)` → `{{schema}}.filter_json_by_keys(tags, pek.keys)::json`
- `json_parse(costcategory)` → `costcategory::json`
- `UUID '{{source_uuid}}'` → `'{{source_uuid}}'::uuid` (2 occurrences)
- `TIMESTAMP '{{start_date}}'` → `'{{start_date}}'::timestamp`
- `date_add('day', 1, TIMESTAMP '{{end_date}}')` → `'{{end_date}}'::timestamp + INTERVAL '1 day'`

**Subquery (FROM clause):**
- Removed `hive.` prefix from `aws_line_items_daily`
- All other SQL (CASE, COALESCE, GROUP BY, aggregations) works identically

**JOINs:**
- Removed `postgres.` prefix from `reporting_awsaccountalias` and `reporting_awsorganizationalunit`

#### File: `reporting_azurecostentrylineitem_daily_summary.sql` ✅ CONVERTED

INSERT query (99 lines) that aggregates Azure daily line items and inserts into PostgreSQL summary table.

**Conversions Applied:**

**INSERT statement:**
- Removed `postgres.` prefix

**CTE 1: `cte_line_items`**
- **Important**: `json_extract_scalar(json_parse(additionalinfo), '$.ServiceType')` → `(additionalinfo::json->>'ServiceType')`
- **Important**: `regexp_like(split_part(...), '^\d+(\.\d+)?$')` → `split_part(...) ~ '^\d+(\.\d+)?$'` (PostgreSQL regex operator)
- `json_parse(tags)` → `tags::json`
- `cast(source as UUID)` → `source::uuid`
- `TIMESTAMP '{{start_date}}'` → `'{{start_date}}'::timestamp`
- `date_add('day', 1, TIMESTAMP '{{end_date}}')` → `'{{end_date}}'::timestamp + INTERVAL '1 day'`
- Removed `hive.` prefix from `azure_line_items`
- **Note**: `split_part()` function works identically in both Trino and PostgreSQL!

**CTE 2: `cte_pg_enabled_keys`**
- Removed `postgres.` prefix

**Main SELECT:**
- `uuid()` → `uuid_generate_v4()`
- `cast(map_filter(cast(li.tags as map(...)), ...) as json)` → `{{schema}}.filter_json_by_keys(li.tags::text, pek.keys)::json`

**Key Learning: PostgreSQL regex operators**

- Trino: `regexp_like(text, pattern)`
- PostgreSQL: `text ~ pattern` (tilde operator for regex matching)

#### File: `reporting_gcpcostentrylineitem_daily_summary.sql` ✅ CONVERTED

INSERT query (78 lines) that aggregates GCP daily line items and inserts into PostgreSQL summary table.

**Conversions Applied:**

**INSERT statement:**
- Removed `postgres.` prefix

**CTE: `cte_pg_enabled_keys`**
- Removed `postgres.` prefix

**Main SELECT:**
- `uuid()` → `uuid_generate_v4()`
- **Important**: `json_extract_scalar(json_parse(system_labels), '$["compute.googleapis.com/machine_spec"]')` → `system_labels::json->>'compute.googleapis.com/machine_spec'`
  - Used twice: in SELECT and in GROUP BY
- **Important**: `json_extract_scalar(json_parse(credits), '$["amount"]')` → `credits::json->>'amount'`
- `map_filter(cast(json_parse(labels) as map(...)), ...)` → `{{schema}}.filter_json_by_keys(labels, pek.keys)::json`
- `UUID '{{source_uuid}}'` → `'{{source_uuid}}'::uuid`
- `TIMESTAMP '{{start_date}}'` → `'{{start_date}}'::timestamp`
- `date_add('day', 1, TIMESTAMP '{{end_date}}')` → `'{{end_date}}'::timestamp + INTERVAL '1 day'`
- Removed `hive.` prefix from table reference

**Key Learning: PostgreSQL JSON path extraction**

- Trino: `json_extract_scalar(json_parse(col), '$["key.with.dots"]')` or `json_extract_scalar(json_parse(col), '$.key')`
- PostgreSQL: `col::json->>'key.with.dots'` or `col::json->>'key'`
- No need for `$` prefix or array notation in PostgreSQL

#### File: `reporting_awscostentrylineitem_summary_by_ec2_compute_p.sql` ✅ CONVERTED

INSERT query (182 lines) that aggregates AWS EC2 compute instances and inserts into PostgreSQL summary table.

**Conversions Applied:**

**INSERT statement:**
- Removed `postgres.` prefix

**CTE 1: `cte_pg_enabled_keys`**
- Removed `postgres.` prefix

**CTE 2: `cte_latest_values`**
- `json_extract_scalar(json_parse(resourcetags), '$.Name')` → `resourcetags::json->>'Name'`
- Removed `hive.` prefix from two table references (outer query and correlated subquery)

**Main SELECT:**
- `uuid()` → `uuid_generate_v4()`
- `map_filter(cast(json_parse(cte_l.tags) as map(...)), ...)` → `{{schema}}.filter_json_by_keys(cte_l.tags, pek.keys)::json`
- `cast(json_parse(cte_l.cost_category) as json)` → `cte_l.cost_category::json`
- `UUID '{{source_uuid}}'` → `'{{source_uuid}}'::uuid`

**Subquery:**
- Removed `hive.` prefix from `aws_line_items_daily`

**JOINs:**
- Removed `postgres.` prefix from `reporting_awsaccountalias`

### PostgreSQL INSERT Query Conversion Progress

#### File: `reporting_ocpusagelineitem_daily_summary.sql`

Converting the first INSERT statement (Pod and Storage usage aggregation) from Trino to PostgreSQL.

**CTEs Converted:**

1. **`cte_pg_enabled_keys`**
   - Minor changes: Removed `postgres.` prefix from table reference

2. **`cte_ocp_node_label_line_item_daily`**
   - **Important**: Replaced `map_filter()` with `filter_json_by_keys()` helper function
   - Semi-important: Converted `date_add('day', 1, {{end_date}})` to `{{end_date}} + INTERVAL '1 day'`

3. **`cte_ocp_namespace_label_line_item_daily`**
   - **Important**: Replaced `map_filter()` with `filter_json_by_keys()` helper function

4. **`cte_ocp_node_capacity`** - No changes required

5. **`cte_ocp_cluster_capacity`** - No changes required

6. **`cte_volume_nodes`** (conditional: `{% if storage_exists %}`) - No changes required

7. **`cte_shared_volume_node_count`** (conditional) - No changes required

**Main SELECT Statements Converted:**

**POD Section:**
- **Important**: Converted `map_concat()` with NULL handling to PostgreSQL `||` operator with nested `COALESCE()`
  ```sql
  -- Trino: map_concat(cast(coalesce(...), ...), ...)
  -- PostgreSQL: COALESCE(COALESCE(nli.node_labels, '{}'::json) || COALESCE(nsli.namespace_labels, '{}'::json) || filter_json_by_keys(...), '{}'::json)
  ```
- **Important**: Added `COALESCE()` for LEFT JOIN capacity fields to prevent NULL issues
  - `max(nc.node_capacity_cpu_core_seconds)` → `COALESCE(max(nc.node_capacity_cpu_core_seconds), 0)`
  - Same for memory and cluster capacity fields
- **Important**: Converted `json_format(cast(x as json))` to `x::text`
- **Important**: Converted `year()/month()/day()` to `EXTRACT(YEAR/MONTH/DAY FROM x)::text`

**STORAGE Section:**
- **Important**: Converted `map_concat()` with 4 inputs (node_labels, namespace_labels, persistentvolume_labels, persistentvolumeclaim_labels)
- **Important**: Converted `last_day_of_month()` to `DATE_TRUNC('month', x) + INTERVAL '1 month - 1 day'`
- **Important**: Added `COALESCE(max(nc.node_count), 1)` to prevent division by zero when volume has no nodes

**Second INSERT: Unallocated Capacity**

Reads from `reporting_ocpusagelineitem_daily_summary` (Pod data only) and writes back new rows with unallocated capacity calculations.

**CTEs:**
1. **`cte_node_role`** - No changes required
2. **`cte_unallocated_capacity`**:
   - **Important**: Converted `year()/month()/day()` to `EXTRACT(YEAR/MONTH/DAY FROM x)::text`
   - Removed `hive.` prefix from table references
   - Removed `postgres.` prefix from table references

**Final SELECT:**
- Removed `postgres.` prefix from `reporting_ocp_cost_category_namespace`
- CASE statement works identically (no conversion needed)
- lpad() works identically (no conversion needed)

**Third INSERT: Copy Data (Trino-specific conversions)**

Reads from the same table and writes back with type conversions (originally Hive→PostgreSQL copy).

**Conversions:**
- **Important**: `uuid()` → `uuid_generate_v4()` (requires uuid-ossp extension)
- **Important**: `json_parse(x)` → `x::json` (for pod_labels, volume_labels, infrastructure_usage_cost)
- **Important**: `map_concat()` with complex casting → `||` operator
  ```sql
  -- Trino: json_parse(json_format(cast(map_concat(...) as json)))
  -- PostgreSQL: COALESCE(pod_labels::json, '{}'::json) || COALESCE(volume_labels::json, '{}'::json)
  ```
- **Important**: `cast(source_uuid as UUID)` → `source_uuid::uuid`
- Removed `hive.` and `postgres.` prefixes from table references
- lpad() and IN clause work identically (no conversion needed)

---

## OpenShift Files Converted

**Total: 18 files**

### Overview

The OpenShift SQL files handle VM cost models and cloud provider integration for OpenShift clusters. This includes:
1. **VM Cost Models** (9 files): Calculate costs for virtual machines running on OpenShift based on CPU core usage
2. **Cloud Provider Integration** (9 files): Match and correlate OpenShift resources with AWS, Azure, and GCP infrastructure

All files use standard Trino→PostgreSQL conversion patterns.

### VM Cost Model Files (9 files)

#### File: `openshift/cost_model/hourly_vm_core.sql` (36 lines)
**Purpose:** Hourly cost calculation for VM cores with standard cost model pricing

**Conversions:**
- `date_trunc('hour', x)` → Works identically in both
- `json_extract_scalar(lower(system_labels), '$.kubevirt_io_vmi_name')` → `lower(system_labels)::json->>'kubevirt_io_vmi_name'`
- `postgres.{{schema}}.reporting_ocp_nodes` → `{{schema}}.reporting_ocp_nodes`
- `hive.{{schema}}.openshift_vm_usage_line_items` → `{{schema}}.openshift_vm_usage_line_items_trino`
- Removed `FORMAT_DATETIME()` - Using PostgreSQL's standard timestamp functions

#### File: `openshift/cost_model/monthly_vm_core.sql` (30 lines)
**Purpose:** Monthly cost summary for VM cores

**Conversions:** Same as hourly_vm_core.sql plus:
- `DATE_TRUNC('month', x)` → Works identically
- No additional conversions required

#### File: `openshift/cost_model/hourly_vm_core_tag_based.sql` (35 lines)
**Purpose:** Hourly VM cost calculation using tag-based pricing

**Conversions:** Same as hourly_vm_core.sql plus:
- `json_parse(node_labels)` → `node_labels::json`
- Uses tag-based rate lookup with LATERAL join

#### File: `openshift/cost_model/monthly_vm_core_tag_based.sql` (30 lines)
**Purpose:** Monthly VM cost summary using tag-based pricing

**Conversions:** Same as hourly_vm_core_tag_based.sql

#### File: `openshift/cost_model/hourly_cost_virtual_machine.sql` (49 lines)
**Purpose:** Hourly cost for entire VMs (includes memory + CPU calculations)

**Conversions:**
- `json_extract_scalar(lower(system_labels), '$.kubevirt_io_vmi_name')` → `lower(system_labels)::json->>'kubevirt_io_vmi_name'`
- `json_extract_scalar(lower(pod_labels), '$.vm_kubevirt_io_name')` → `lower(pod_labels)::json->>'vm_kubevirt_io_name'`
- `hive.{{schema}}.openshift_vm_usage_line_items` → `{{schema}}.openshift_vm_usage_line_items_trino`
- `hive.{{schema}}.openshift_pod_usage_line_items` → `{{schema}}.openshift_pod_usage_line_items_trino`
- Complex CASE expressions work identically

#### File: `openshift/cost_model/hourly_cost_vm_tag_based.sql` (52 lines)
**Purpose:** Hourly VM cost with tag-based rates (memory + CPU)

**Conversions:** Same as hourly_cost_virtual_machine.sql plus tag-based rate lookup

#### File: `openshift/cost_model/monthly_project_tag_based.sql` (34 lines)
**Purpose:** Monthly cost summary by project using tag-based pricing

**Conversions:**
- `json_parse(pod_labels)` → `pod_labels::json`
- `json_parse(volume_labels)` → `volume_labels::json`
- `json_parse(infrastructure_usage_cost)` → `infrastructure_usage_cost::json`
- `hive.{{schema}}.reporting_ocpusagelineitem_daily_summary` → `{{schema}}.reporting_ocpusagelineitem_daily_summary_trino`
- All other functions (COALESCE, CAST) work identically

#### File: `openshift/populate_vm_tmp_table.sql` (33 lines)
**Purpose:** Creates temp table for VM metrics from pod usage data

**Conversions:**
- `json_extract_scalar(lower(pod_labels), '$.vm_kubevirt_io_name')` → `lower(pod_labels)::json->>'vm_kubevirt_io_name'`
- `hive.{{schema}}.openshift_pod_usage_line_items` → `{{schema}}.openshift_pod_usage_line_items_trino`
- All temp table operations work identically

#### File: `openshift/populate_vm_tmp_table_with_vm_report.sql` (39 lines)
**Purpose:** Populates temp table using VM-specific reports instead of pod reports

**Conversions:**
- `json_extract_scalar(lower(system_labels), '$.kubevirt_io_vmi_name')` → `lower(system_labels)::json->>'kubevirt_io_vmi_name'`
- `hive.{{schema}}.openshift_vm_usage_line_items_daily` → `{{schema}}.openshift_vm_usage_line_items_daily_trino`

### Cloud Provider Integration Files (9 files)

#### File: `aws/reporting_ocpinfrastructure_provider_map.sql` (62 lines)
**Purpose:** Maps OCP clusters to AWS infrastructure providers by matching resource IDs

**Conversions:**
- `hive.{{schema}}.aws_line_items_daily` → `{{schema}}.aws_line_items_daily_trino`
- `hive.{{schema}}.openshift_pod_usage_line_items_daily` → `{{schema}}.openshift_pod_usage_line_items_daily_trino`
- `date_add('day', 1, {{end_date}})` → `{{end_date}} + INTERVAL '1 day'`
- `cast(x as varchar)` → `x::varchar`
- `postgres.public.api_provider` → `public.api_provider`
- `postgres.{{schema}}.reporting_tenant_api_provider` → `{{schema}}.reporting_tenant_api_provider`

#### File: `azure/reporting_ocpinfrastructure_provider_map.sql` (62 lines)
**Purpose:** Maps OCP clusters to Azure infrastructure providers

**Conversions:**
- `hive.{{schema}}.azure_line_items` → `{{schema}}.azure_line_items_trino`
- `hive.{{schema}}.openshift_pod_usage_line_items_daily` → `{{schema}}.openshift_pod_usage_line_items_daily_trino`
- `date_add('day', 1, {{end_date}})` → `{{end_date}} + INTERVAL '1 day'`
- `split_part()` - already PostgreSQL compatible
- Extracts instance name using `split_part(resourceid, '/', 9)`

#### File: `gcp/reporting_ocpinfrastructure_provider_map.sql` (61 lines)
**Purpose:** Maps OCP clusters to GCP infrastructure providers

**Conversions:**
- `hive.{{schema}}.gcp_line_items_daily` → `{{schema}}.gcp_line_items_daily_trino`
- `hive.{{schema}}.openshift_pod_usage_line_items_daily` → `{{schema}}.openshift_pod_usage_line_items_daily_trino`
- `date_add('day', 1, {{end_date}})` → `{{end_date}} + INTERVAL '1 day'`
- Uses `strpos(gcp.resource_name, ocp.node) != 0` for substring matching

#### Files: `aws/openshift/populate_daily_summary/1_resource_matching_by_cluster.sql` (141 lines)
**Purpose:** Matches AWS resources to OCP nodes at cluster level

**Conversions:**
- All `hive.{{schema}}.table` → `{{schema}}.table_trino`
- `date_add('day', 1, {{end_date}})` → `{{end_date}} + INTERVAL '1 day'`
- `cast(x as varchar)` → `x::varchar`
- Multiple CTEs with standard SQL (no conversion needed)

#### Files: `azure/openshift/populate_daily_summary/1_resource_matching_by_cluster.sql` (159 lines)
**Purpose:** Matches Azure resources to OCP nodes

**Conversions:**
- Same as AWS version
- Uses `split_part()` for Azure resource ID parsing

#### Files: `gcp/openshift/populate_daily_summary/1_resource_matching_by_cluster.sql` (140 lines)
**Purpose:** Matches GCP resources to OCP nodes

**Conversions:**
- Same as AWS version
- Uses GCP-specific resource matching logic

#### Files: `aws/openshift/populate_daily_summary/2_summarize_data_by_cluster.sql` (303 lines)
**Purpose:** Summarizes OCP-on-AWS cost data by cluster

**Conversions:**
- Complex multi-CTE query with disk capacity calculations
- `hive.{{schema}}.table` → `{{schema}}.table_trino`
- `date_add()` → INTERVAL arithmetic
- All standard aggregations work identically

#### Files: `azure/openshift/populate_daily_summary/2_summarize_data_by_cluster.sql` (295 lines)
**Purpose:** Summarizes OCP-on-Azure cost data

**Conversions:**
- Same pattern as AWS version
- Azure-specific fields (subscriptionid, resourcegroup)

#### Files: `gcp/openshift/populate_daily_summary/2_summarize_data_by_cluster.sql` (299 lines)
**Purpose:** Summarizes OCP-on-GCP cost data

**Conversions:**
- Same pattern as AWS/Azure versions
- GCP-specific fields (project_id, service_id, invoice_month)

### Summary

**All OpenShift files use standard conversion patterns:**
- `hive.{{schema}}.table` → `{{schema}}.table_trino`
- `postgres.{{schema}}.table` → `{{schema}}.table`
- `json_extract_scalar()` → `::json->>'key'`
- `date_add()` → INTERVAL arithmetic
- `json_parse()` → `::json`

**No new conversion patterns introduced.**

---

## AWS/Azure/GCP Provider Integration Files

**Total: 32 files**

### AWS Files (10 files)

#### Root Level

##### reporting_ocpinfrastructure_provider_map.sql (62 lines)

**Purpose:** Maps OCP (OpenShift) clusters to AWS infrastructure providers by matching resource IDs between AWS line items and OCP pod usage.

**Conversions:**
- `hive.{{schema}}.aws_line_items_daily` → `{{schema}}.aws_line_items_daily_trino`
- `hive.{{schema}}.openshift_pod_usage_line_items_daily` → `{{schema}}.openshift_pod_usage_line_items_daily_trino`
- `date_add('day', 1, {{end_date}})` → `{{end_date}} + INTERVAL '1 day'`
- `cast(x as varchar)` → `x::varchar`
- `postgres.public.api_provider` → `public.api_provider`
- `postgres.{{schema}}.reporting_tenant_api_provider` → `{{schema}}.reporting_tenant_api_provider`

#### aws/openshift/ Summary Files

All summary files aggregate data from the managed table `managed_reporting_ocpawscostlineitem_project_daily_summary_trino` to create provider-specific summary views.

**Common Conversions (all files):**
- `uuid()` → `uuid_generate_v4()`
- `cast({{aws_source_uuid}} as uuid)` → `{{aws_source_uuid}}::uuid`
- `date_add('day', 1, {{end_date}})` → `{{end_date}} + INTERVAL '1 day'`
- `hive.{{schema}}.managed_*` → `{{schema}}.managed_*_trino`
- `postgres.{{schema}}.table` → `{{schema}}.table`

##### reporting_ocpaws_compute_summary_p.sql (55 lines)

**Purpose:** Aggregates compute instance costs by usage_start, account, instance_type, and resource_id.

**Groups by:** usage_start, usage_account_id, account_alias_id, instance_type, resource_id

**Filters:** instance_type IS NOT NULL

##### reporting_ocpaws_cost_summary_by_account_p.sql (46 lines)

**Purpose:** Aggregates total costs by AWS account.

**Groups by:** usage_start, usage_account_id, account_alias_id

##### reporting_ocpaws_cost_summary_by_region_p.sql (50 lines)

**Purpose:** Aggregates costs by AWS region and availability zone.

**Groups by:** usage_start, usage_account_id, account_alias_id, region, availability_zone

##### reporting_ocpaws_cost_summary_by_service_p.sql (50 lines)

**Purpose:** Aggregates costs by AWS product code and product family.

**Groups by:** usage_start, usage_account_id, account_alias_id, product_code, product_family

##### reporting_ocpaws_cost_summary_p.sql (44 lines)

**Purpose:** Aggregates total costs at the cluster level with cost category support.

**Groups by:** usage_start

##### reporting_ocpaws_database_summary_p.sql (53 lines)

**Purpose:** Aggregates database service costs (RDS, DynamoDB, ElastiCache, Neptune, Redshift, DocumentDB).

**Groups by:** usage_start, usage_account_id, account_alias_id, product_code

**Filters:** product_code IN ('AmazonRDS','AmazonDynamoDB','AmazonElastiCache','AmazonNeptune','AmazonRedshift','AmazonDocumentDB')

##### reporting_ocpaws_network_summary_p.sql (53 lines)

**Purpose:** Aggregates network service costs (VPC, CloudFront, Route53, API Gateway).

**Groups by:** usage_start, usage_account_id, account_alias_id, product_code

**Filters:** product_code IN ('AmazonVPC','AmazonCloudFront','AmazonRoute53','AmazonAPIGateway')

##### reporting_ocpaws_storage_summary_p.sql (54 lines)

**Purpose:** Aggregates storage costs for products with 'Storage' in product family and GB-Mo unit.

**Groups by:** usage_start, usage_account_id, account_alias_id, product_family

**Filters:** product_family LIKE '%%Storage%%' AND unit = 'GB-Mo'

##### reporting_ocpawscostlineitem_project_daily_summary_p.sql (110 lines)

**Purpose:** Copies data from managed Trino table to final PostgreSQL partitioned table with tag filtering and data transfer direction mapping.

**Additional Conversions:**
- `json_parse(pod_labels)` → `pod_labels::json`
- `map_filter(cast(json_parse(tags) as map(...)), (k,v) -> contains(pek.keys, k))` → `{{schema}}.filter_json_by_keys(tags, pek.keys)::json`
- `json_parse(aws_cost_category)` → `aws_cost_category::json`
- `cast(source as UUID)` → `source::UUID`

**Features:**
- Filters tags based on enabled keys for AWS and OCP providers
- Maps data transfer direction: 'IN' and 'OUT'
- Uses CTE for enabled tag keys

### Azure Files (10 files)

#### Root Level

##### reporting_ocpinfrastructure_provider_map.sql (62 lines)

**Purpose:** Maps OCP clusters to Azure infrastructure providers by matching instance names (extracted from Azure resource IDs) with OCP node names.

**Conversions:**
- `hive.{{schema}}.azure_line_items` → `{{schema}}.azure_line_items_trino`
- `hive.{{schema}}.openshift_pod_usage_line_items_daily` → `{{schema}}.openshift_pod_usage_line_items_daily_trino`
- `date_add('day', 1, {{end_date}})` → `{{end_date}} + INTERVAL '1 day'`
- `cast(x as varchar)` → `x::varchar`
- `split_part()` - already PostgreSQL compatible

**Features:**
- Extracts instance name using `split_part(resourceid, '/', 9)`
- Matches Azure instances to OCP nodes by exact name match

#### azure/openshift/ Summary Files

All summary files aggregate data from the managed table `managed_reporting_ocpazurecostlineitem_project_daily_summary_trino`.

**Common Conversions (all files):**
- `uuid()` → `uuid_generate_v4()`
- `cast({{azure_source_uuid}} as uuid)` → `{{azure_source_uuid}}::uuid`
- `date_add('day', 1, {{end_date}})` → `{{end_date}} + INTERVAL '1 day'`
- `hive.{{schema}}.managed_*` → `{{schema}}.managed_*_trino`
- `postgres.{{schema}}.table` → `{{schema}}.table`

##### reporting_ocpazure_compute_summary_p.sql (44 lines)

**Purpose:** Aggregates compute instance costs by subscription, instance_type, and resource_id.

**Groups by:** usage_start, subscription_guid, instance_type, resource_id

**Filters:** instance_type IS NOT NULL AND unit_of_measure = 'Hrs'

##### reporting_ocpazure_cost_summary_by_account_p.sql (34 lines)

**Purpose:** Aggregates total costs by Azure subscription.

**Groups by:** usage_start, subscription_guid

##### reporting_ocpazure_cost_summary_by_location_p.sql (36 lines)

**Purpose:** Aggregates costs by Azure resource location.

**Groups by:** usage_start, subscription_guid, resource_location

##### reporting_ocpazure_cost_summary_by_service_p.sql (36 lines)

**Purpose:** Aggregates costs by Azure service name.

**Groups by:** usage_start, subscription_guid, service_name

##### reporting_ocpazure_cost_summary_p.sql (32 lines)

**Purpose:** Aggregates total costs at the cluster level with cost category support.

**Groups by:** usage_start

##### reporting_ocpazure_database_summary_p.sql (44 lines)

**Purpose:** Aggregates database service costs (Cosmos DB, Cache for Redis, or services with 'database' in name).

**Groups by:** usage_start, subscription_guid, service_name

**Filters:** service_name IN ('Cosmos DB','Cache for Redis') OR lower(service_name) LIKE '%%database%%'

##### reporting_ocpazure_network_summary_p.sql (41 lines)

**Purpose:** Aggregates network service costs (Virtual Network, VPN, DNS, Traffic Manager, ExpressRoute, Load Balancer, Application Gateway).

**Groups by:** usage_start, subscription_guid, service_name

**Filters:** service_name IN ('Virtual Network','VPN','DNS','Traffic Manager','ExpressRoute','Load Balancer','Application Gateway')

##### reporting_ocpazure_storage_summary_p.sql (42 lines)

**Purpose:** Aggregates storage costs for services with 'Storage' in service name and GB-Mo unit.

**Groups by:** usage_start, subscription_guid, service_name

**Filters:** service_name LIKE '%%Storage%%' AND unit_of_measure = 'GB-Mo'

##### reporting_ocpazurecostlineitem_project_daily_summary_p.sql (97 lines)

**Purpose:** Copies data from managed Trino table to final PostgreSQL partitioned table with tag filtering and data transfer direction mapping.

**Additional Conversions:**
- `json_parse(pod_labels)` → `pod_labels::json`
- `map_filter(cast(json_parse(tags) as map(...)), (k,v) -> contains(pek.keys, k))` → `{{schema}}.filter_json_by_keys(tags, pek.keys)::json`
- `cast(source as UUID)` → `source::UUID`

**Features:**
- Filters tags based on enabled keys for Azure and OCP providers
- Maps data transfer direction: 'datatrin' → 'IN', 'datatrout' → 'OUT'
- Uses CTE for enabled tag keys

### GCP Files (12 files)

#### Root Level

##### reporting_ocpinfrastructure_provider_map.sql (61 lines)

**Purpose:** Maps OCP clusters to GCP infrastructure providers by matching GCP resource names with OCP node names using substring matching.

**Conversions:**
- `hive.{{schema}}.gcp_line_items_daily` → `{{schema}}.gcp_line_items_daily_trino`
- `hive.{{schema}}.openshift_pod_usage_line_items_daily` → `{{schema}}.openshift_pod_usage_line_items_daily_trino`
- `date_add('day', 1, {{end_date}})` → `{{end_date}} + INTERVAL '1 day'`
- `cast(x as varchar)` → `x::varchar`

**Features:**
- Uses `strpos(gcp.resource_name, ocp.node) != 0` for substring matching

#### gcp/openshift/ Summary Files

All summary files aggregate data from the managed table `managed_reporting_ocpgcpcostlineitem_project_daily_summary_trino`.

**Common Conversions (all files):**
- `uuid()` → `uuid_generate_v4()`
- `cast({{gcp_source_uuid}} as uuid)` → `{{gcp_source_uuid}}::uuid`
- `date_add('day', 1, {{end_date}})` → `{{end_date}} + INTERVAL '1 day'`
- `hive.{{schema}}.managed_*` → `{{schema}}.managed_*_trino`
- `postgres.{{schema}}.table` → `{{schema}}.table`

**GCP-Specific Features:**
- All tables include `credit_amount` and `invoice_month` fields for GCP billing credits

##### reporting_ocpgcp_compute_summary_p.sql (49 lines)

**Purpose:** Aggregates compute instance costs by account, cluster, and instance_type.

**Groups by:** cluster_id, account_id, cluster_alias, usage_start, usage_end, instance_type, invoice_month

##### reporting_ocpgcp_cost_summary_by_account_p.sql (42 lines)

**Purpose:** Aggregates total costs by GCP billing account.

**Groups by:** cluster_id, cluster_alias, usage_start, usage_end, account_id, invoice_month

##### reporting_ocpgcp_cost_summary_by_gcp_project_p.sql (45 lines)

**Purpose:** Aggregates costs by GCP project.

**Groups by:** cluster_id, cluster_alias, usage_start, usage_end, project_id, project_name, invoice_month

##### reporting_ocpgcp_cost_summary_by_region_p.sql (45 lines)

**Purpose:** Aggregates costs by GCP region.

**Groups by:** cluster_id, cluster_alias, usage_start, usage_end, account_id, region, invoice_month

##### reporting_ocpgcp_cost_summary_by_service_p.sql (48 lines)

**Purpose:** Aggregates costs by GCP service.

**Groups by:** cluster_id, cluster_alias, usage_start, usage_end, account_id, service_id, service_alias, invoice_month

##### reporting_ocpgcp_cost_summary_p.sql (41 lines)

**Purpose:** Aggregates total costs at the cluster level with cost category support.

**Groups by:** cluster_id, cluster_alias, usage_start, usage_end, invoice_month

##### reporting_ocpgcp_database_summary_p.sql (59 lines)

**Purpose:** Aggregates database service costs (SQL, Spanner, Bigtable, Firestore, Firebase, Memorystore, MongoDB).

**Groups by:** cluster_id, cluster_alias, usage_start, usage_end, account_id, service_id, service_alias, invoice_month

**Filters:** service_alias LIKE patterns for database services

##### reporting_ocpgcp_network_summary_p.sql (65 lines)

**Purpose:** Aggregates network service costs (Network, VPC, Firewall, Route, IP, DNS, CDN, NAT, Traffic Director, etc.).

**Groups by:** cluster_id, cluster_alias, usage_start, usage_end, account_id, service_id, service_alias, invoice_month

**Filters:** service_alias LIKE patterns for network services

##### reporting_ocpgcp_storage_summary_p.sql (54 lines)

**Purpose:** Aggregates storage costs (Filestore, Storage, Cloud Storage, Data Transfer).

**Groups by:** cluster_id, cluster_alias, usage_start, usage_end, account_id, service_id, service_alias, invoice_month

**Filters:** service_alias IN ('Filestore', 'Storage', 'Cloud Storage', 'Data Transfer')

##### reporting_ocpgcpcostlineitem_project_daily_summary_p.sql (114 lines)

**Purpose:** Copies data from managed Trino table to final PostgreSQL partitioned table with tag filtering and gibibyte-to-gigabyte conversion for data transfer.

**Additional Conversions:**
- `json_parse(pod_labels)` → `pod_labels::json`
- `map_filter(cast(json_parse(tags) as map(...)), (k,v) -> contains(pek.keys, k))` → `{{schema}}.filter_json_by_keys(tags, pek.keys)::json`
- `cast(source as UUID)` → `source::UUID`

**Features:**
- Filters tags based on enabled keys for GCP and OCP providers
- Converts gibibyte to gigabyte for data transfer: `usage_amount * 1.07374` when `unit = 'gibibyte'`
- Maps data transfer direction: 'IN' and 'OUT'
- Uses CTE for enabled tag keys

##### reporting_ocpgcp_matched_tags.sql (48 lines)

**Purpose:** Finds matching tags between GCP labels and OCP pod/volume labels for tag-based cost allocation.

**Conversions:**
- `hive.{{schema}}.gcp_line_items_daily` → `{{schema}}.gcp_line_items_daily_trino`
- `UNNEST(cast(json_parse(labels) as map(...)))` → `LATERAL json_each_text(labels::json)`
- `UNNEST(map1, map2)` → Separate CROSS JOIN LATERAL for each map
- `postgres.{{schema}}.reporting_enabledtagkeys` → `{{schema}}.reporting_enabledtagkeys`

**Features:**
- Uses LATERAL json_each_text for JSON expansion
- Joins GCP and OCP tags with case-insensitive matching
- Filters by enabled tag keys for both GCP and OCP providers
- Returns matching tags as JSON strings

### Summary Statistics

**Total Files Converted:** 32

**AWS:**
- 1 root-level file
- 9 openshift summary files
- **Total:** 10 files

**Azure:**
- 1 root-level file
- 9 openshift summary files
- **Total:** 10 files

**GCP:**
- 1 root-level file
- 11 openshift summary files
- **Total:** 12 files

**All conversions use standard Trino→PostgreSQL patterns established in previous files.**

---

## Subscription (SUBS) SQL Files Conversion

### Important Note

**According to Luke Couzens: These subscription files are NOT needed for on-prem deployment.**

This conversion is included for completeness but may not be required for the ONPREM use case.

### Overview

The subscription SQL files in `koku/subs/trino_sql/` are used for Red Hat Enterprise Linux (RHEL) subscription metering on AWS and Azure cloud instances. These files track RHEL usage, extract subscription metadata from resource tags, and upload data to S3 for subscription billing purposes.

This is separate from cost reporting - it's specifically for tracking Red Hat subscription usage on cloud instances.

### Files Converted

**Total: 8 files** (4 AWS + 4 Azure, no GCP)

#### AWS Files (4 files)

##### determine_ids_for_provider.sql (15 lines)
**Purpose:** Find AWS usage accounts that have EC2 instances with RHEL tags.

**Conversions:**
- `hive.{{schema}}.aws_line_items` → `{{schema}}.aws_line_items_trino`
- `postgres.{{schema}}.reporting_subs_id_map` → `{{schema}}.reporting_subs_id_map`
- All other functions already PostgreSQL compatible (`strpos`, `lower`)

##### determine_resource_ids_for_usage_account.sql (20 lines)
**Purpose:** Get resource IDs and their latest timestamps for a specific usage account.

**Conversions:**
- `hive.{{schema}}.aws_line_items` → `{{schema}}.aws_line_items_trino`
- `postgres.{{schema}}.reporting_subs_last_processed_time` → `{{schema}}.reporting_subs_last_processed_time`

##### subs_row_count.sql (25 lines)
**Purpose:** Count how many RHEL records need to be processed for given resource IDs and time ranges.

**Conversions:**
- `hive.{{schema}}.aws_line_items` → `{{schema}}.aws_line_items_trino`

##### subs_summary.sql (89 lines)
**Purpose:** Extract and transform RHEL subscription data from AWS EC2 instances with detailed role, usage, SLA, and version mappings.

**Conversions:**
- `split(identity_timeinterval, '/')` → `string_to_array(identity_timeinterval, '/')`
- `transform_keys(map_filter(...), (k,v) -> lower(k))` → Subquery with `jsonb_object_agg(lower(key), value)` and `jsonb_each_text()`
- `map_filter(cast(json_parse(resourcetags) as map(...)), (k,v) -> contains(ARRAY[...], lower(k)))` → `WHERE lower(key) = ANY(ARRAY[...])`
- `json_extract_scalar(tags, '$.key')` → `tags::json->>'key'`
- `hive.{{schema}}.aws_line_items` → `{{schema}}.aws_line_items_trino`

**Key Pattern:** Complex lambda functions converted to PostgreSQL LATERAL subquery with jsonb_object_agg:
```sql
-- Trino:
cast(transform_keys(map_filter(..., (k,v) -> contains(...)), (k,v) -> lower(k)) as json)

-- PostgreSQL:
(SELECT jsonb_object_agg(lower(key), value)
 FROM jsonb_each_text(resourcetags::jsonb)
 WHERE lower(key) = ANY(ARRAY[...]))::text
```

#### Azure Files (4 files)

##### determine_ids_for_provider.sql (16 lines)
**Purpose:** Find Azure subscription IDs that have Virtual Machines with RHEL tags.

**Conversions:**
- `hive.{{schema}}.azure_line_items` → `{{schema}}.azure_line_items_trino`
- `json_extract_scalar(lower(additionalinfo), '$.vcpus')` → `lower(additionalinfo)::json->>'vcpus'`
- `json_extract_scalar(lower(tags), '$.com_redhat_rhel')` → `lower(tags)::json->>'com_redhat_rhel'`
- `postgres.{{schema}}.reporting_subs_id_map` → `{{schema}}.reporting_subs_id_map`

##### determine_resource_ids_for_usage_account.sql (36 lines)
**Purpose:** Get resource IDs with instance keys (for handling scalesets) and calculated end times.

**Conversions:**
- `hive.{{schema}}.azure_line_items` → `{{schema}}.azure_line_items_trino`
- `json_extract_scalar(lower(additionalinfo), '$.vmname')` → `lower(additionalinfo)::json->>'vmname'`
- `date_add('day', -2, current_date)` → `current_date - INTERVAL '2 days'`
- `date_add('day', -1, max_date)` → `max_date - INTERVAL '1 day'`
- `regexp_extract(resourceid, '([^/]+$)')` → `(regexp_match(resourceid, '([^/]+$)'))[1]`
- `postgres.{{schema}}.reporting_subs_last_processed_time` → `{{schema}}.reporting_subs_last_processed_time`

##### subs_row_count.sql (27 lines)
**Purpose:** Count how many RHEL VM records need to be processed.

**Conversions:**
- `hive.{{schema}}.azure_line_items` → `{{schema}}.azure_line_items_trino`
- `json_extract_scalar(lower(additionalinfo), '$.vcpus')` → `lower(additionalinfo)::json->>'vcpus'`
- `json_extract_scalar(lower(lower(tags)), '$.com_redhat_rhel')` → `lower(lower(tags))::json->>'com_redhat_rhel'`

##### subs_summary.sql (69 lines)
**Purpose:** Extract and transform RHEL subscription data from Azure VMs with detailed role, usage, SLA, and version mappings.

**Conversions:**
- `with_timezone(date, 'UTC')` → `date AT TIME ZONE 'UTC'`
- `date_add('day', 1, date)` → `date + INTERVAL '1 day'`
- `json_extract_scalar(lower(additionalinfo), '$.vcpus')` → `lower(additionalinfo)::json->>'vcpus'`
- `json_extract_scalar(lower(tags), '$.key')` → `lower(tags)::json->>'key'`
- `regexp_extract(resourceid, '([^/]+$)')` → `(regexp_match(resourceid, '([^/]+$)'))[1]`
- `hive.{{schema}}.azure_line_items` → `{{schema}}.azure_line_items_trino`
- All CASE statements and COALESCE work identically

### Conversion Summary

**Complexity: MODERATE - ALL FILES CONVERTED**

All 8 files have been successfully converted. The simple files (6) use standard conversions already established in previous files:
- `hive.{{schema}}.table` → `{{schema}}.table_trino`
- `postgres.{{schema}}.table` → `{{schema}}.table`
- `json_extract_scalar()` → `::json->>'key'`
- `date_add('day', N, date)` → `date + INTERVAL 'N days'` or `date - INTERVAL 'N days'`
- `regexp_extract(str, pattern)` → `(regexp_match(str, pattern))[1]`

The complex files (2) introduced new conversion patterns:

**AWS subs_summary.sql:**
- `split()` → `string_to_array()`
- `transform_keys(map_filter(...), lambda)` → LATERAL subquery with `jsonb_object_agg()` and `jsonb_each_text()`
- Lambda functions with `contains()` → `WHERE ... = ANY(ARRAY[...])`

**Azure subs_summary.sql:**
- `with_timezone(date, 'UTC')` → `date AT TIME ZONE 'UTC'`

**Note:** While subscription functionality is not required for ONPREM deployment per Luke Couzens, all files have been converted for completeness.

### File Locations

**Source (Trino):** `koku/subs/trino_sql/`
**Target (PostgreSQL):** `koku/subs/postgres_sql/`

**Code Usage:** `koku/subs/subs_data_extractor.py` loads these files using:
```python
sql_file = f"trino_sql/{self.provider_type.lower()}/filename.sql"
sql = pkgutil.get_data("subs", sql_file)
```

---

## Trino Infrastructure Analysis

See **[TRINO_INFRASTRUCTURE_ANALYSIS.md](./TRINO_INFRASTRUCTURE_ANALYSIS.md)** for comprehensive analysis of:
- Database connection abstraction layer (already implemented!)
- Files using Trino infrastructure (all using abstraction correctly ✅)
- Error handling considerations for ONPREM
- Test files that may need ONPREM guards
- Deployment considerations (smoke_test.sh)

**Key Finding:** The codebase already has a complete abstraction layer (`get_report_db_accessor()`) that switches between Trino and PostgreSQL based on `settings.ONPREM`. All production code uses this abstraction correctly!

## Hardcoded SQL Analysis - CRITICAL BUG DISCOVERED

⚠️ **CRITICAL**: See **[CRITICAL_BUG_HARDCODED_HIVE_SQL.md](./CRITICAL_BUG_HARDCODED_HIVE_SQL.md)** for urgent fix requirements!

See **[HARDCODED_SQL_ANALYSIS.md](./HARDCODED_SQL_ANALYSIS.md)** for comprehensive analysis of:
- All hardcoded SQL in Python files (analyzed ALL files including `masu/database/report_db_accessor*.py`)
- Critical bug: hardcoded `hive.` prefix SQL will FAIL in PostgreSQL
- Files requiring fixes to support ONPREM mode
- Recommended fix approach (extend abstraction layer)

**CRITICAL FINDINGS:**
- ❌ **5 files with hardcoded `hive.` prefix** that WILL FAIL in PostgreSQL:
  - `report_db_accessor_base.py`
  - `ocp_report_db_accessor.py`
  - `aws_report_db_accessor.py`
  - `azure_report_db_accessor.py`
  - `gcp_report_db_accessor.py`
- ⚠️ **1 additional file needs ONPREM guard**: `migrate_trino_tables.py`
- 🔧 **Fix Required**: Extend abstraction layer to generate database-specific SQL for DELETE/SELECT operations
- ❌ **Initial "guard pattern" analysis was WRONG** - guards don't prevent execution, they enable it!

---

## Next Steps

### SQL Conversion (Completed ✅)

1. ✅ Create PostgreSQL table creation SQL files
2. ✅ Verify all tables in TRINO_MANAGED_TABLES are created
3. ✅ Convert root-level SQL files (10 files)
4. ✅ Convert OpenShift SQL files (18 files)
5. ✅ Convert AWS/Azure/GCP provider integration SQL files (32 files)
6. ✅ Convert subscription SQL files (8 files - ALL files converted)
7. ✅ Add `get_sql_folder_name()` method to base class for dynamic folder selection
8. ✅ Update all hardcoded "trino_sql" references to use `get_sql_folder_name()` (34 occurrences updated across 6 files)
   - `report_db_accessor_base.py`: 1 reference
   - `aws_report_db_accessor.py`: 5 references
   - `azure_report_db_accessor.py`: 4 references
   - `gcp_report_db_accessor.py`: 4 references
   - `ocp_report_db_accessor.py`: 15 references
   - `subs_data_extractor.py`: 4 references (inherits from ReportDBAccessorBase)
9. ✅ Update subscription code (`subs_data_extractor.py`) to use dynamic folder selection
10. ✅ Verify Trino infrastructure uses abstraction layer (see TRINO_INFRASTRUCTURE_ANALYSIS.md)

### Testing & Deployment (Remaining)

11. ✅ Add skip decorators to Trino-specific tests (test_trino_db_utils.py)
    - Added `@unittest.skipIf(getattr(settings, 'ONPREM', False), ...)` to 4 test methods/classes
    - Tests now skip gracefully when ONPREM=True
12. ✅ Update smoke_test.sh to skip Trino deployment when ONPREM=True
    - Added conditional logic: `if [ "${ONPREM:-false}" = "true" ]; then COMPONENTS="koku"; else COMPONENTS="koku trino"; fi`
    - Trino-specific parameters only added when not in ONPREM mode
    - Usage: `ONPREM=true ./smoke_test.sh` for ONPREM deployment
13. ✅ Analyze all hardcoded SQL in Python files (see HARDCODED_SQL_ANALYSIS.md and CRITICAL_BUG_HARDCODED_HIVE_SQL.md)
    - Searched ALL Python files including `masu/database/report_db_accessor*.py`
    - Excluded only `koku/reportdb_accessor*.py` (abstraction layer implementations)
    - Found 8 files with hardcoded Trino SQL:
      - test_trino_db_utils.py: ✅ Skip decorators (no fix needed)
      - data_validation.py: ✅ Trino/PostgreSQL abstraction (no fix needed)
      - report_db_accessor_base.py: ❌ CRITICAL BUG - hardcoded `hive.` prefix
      - ocp_report_db_accessor.py: ❌ CRITICAL BUG - hardcoded `hive.` prefix
      - aws_report_db_accessor.py: ❌ CRITICAL BUG - hardcoded `hive.` prefix
      - azure_report_db_accessor.py: ❌ CRITICAL BUG - hardcoded `hive.` prefix
      - gcp_report_db_accessor.py: ❌ CRITICAL BUG - hardcoded `hive.` prefix
      - migrate_trino_tables.py: ⚠️ Needs ONPREM guard (see item 17)
    - **CRITICAL**: Initial analysis was wrong - `hive.` prefix will cause PostgreSQL syntax errors!
14. 🔄 **IN PROGRESS: Fix hardcoded `hive.` prefix SQL in 5 files** (see CRITICAL_BUG_HARDCODED_HIVE_SQL.md)
    - **Status**: 1 of 5 files fixed ✅
    - **Files affected**:
      - report_db_accessor_base.py: ✅ FIXED - created `get_delete_by_month_sql()` abstraction
      - ocp_report_db_accessor.py: ⬜ TODO - 3 methods need abstraction
      - aws_report_db_accessor.py: ⬜ TODO - 1 method needs abstraction
      - azure_report_db_accessor.py: ⬜ TODO - 1 method needs abstraction
      - gcp_report_db_accessor.py: ⬜ TODO - 2 methods need abstraction
    - **Approach**: Extend abstraction layer in `koku/reportdb_accessor*.py`
    - **Implemented method**: `get_delete_by_month_sql()` - generates DELETE with correct catalog/schema syntax
    - **Remaining methods needed**:
      - `get_delete_by_day_sql()` - for day-level partition deletion
      - `get_delete_by_source_sql()` - for source deletion
      - `get_select_tag_matching_sql()` - for tag matching queries
      - `get_delete_date_range_sql()` - for date range deletion
    - **Trino syntax**: `DELETE FROM hive.{schema}.{table}`
    - **PostgreSQL syntax**: `DELETE FROM "{schema}".{table}` (no `hive.` catalog)
    - **Impact**: Without complete fix, partition deletion and tag matching will FAIL in ONPREM mode
15. ⬜ Test table creation with actual PostgreSQL database
16. ⬜ Verify indexes are created correctly
17. ⬜ Performance test queries with indexes vs without
18. ⬜ Add ONPREM guard to migrate_trino_tables.py management command
    - File: koku/masu/management/commands/migrate_trino_tables.py
    - Add early return at start of handle() method when ONPREM=True
    - This is a Trino-specific command for table migrations (not needed for ONPREM)
    - See HARDCODED_SQL_ANALYSIS.md for implementation details
19. ⬜ Add integration tests with settings.ONPREM=True
20. ⬜ Verify error handling works for both Trino and PostgreSQL exceptions
