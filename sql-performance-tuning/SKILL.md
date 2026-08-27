---
name: SQL Performance Tuning
description: Routine for analyzing and optimizing SQL Server stored procedures, functions, and views. Creates backups, generates tuning scripts, captures IO/TIME statistics via MCP, identifies anti-patterns, applies optimizations, and validates improvements.
inclusion: manual
---

# SQL Performance Tuning Routine

When the user specifies a SQL file for performance tuning, follow this routine step by step:

## Procedure

### 1. Create a backup copy

- Copy the original file to the same location with the suffix `.old` appended before the extension.
- Example: `dbo.aasi_my_procedure.sql` → `dbo.aasi_my_procedure.old.sql`
- This preserves the original version for comparison and rollback.

### 2. Create a tuning call script (Functions and Stored Procedures only)

If the SQL object is a Function or Stored Procedure, create a file in the same folder with the prefix `tuning_` followed by the original file name (e.g., `tuning_dbo.aasi_my_procedure.sql`).

This file contains a ready-to-execute call to the function/SP with representative parameter values. To generate it:

#### 2a. Identify parameters

- Read the parameter declarations from the SQL file (`@param_name <type>`).
- Cross-reference parameter types with the `.hbm.xml` mappings to understand what entity/column each parameter represents.
- Use the SQL Server MCP (`ExecuteQuery`) to sample real data for each parameter:

```sql
-- Example: get a representative value for @LegalEntityId
SELECT TOP 1 LegalEntityId FROM dbo.LegalEntity WHERE Active = 1
```

#### 2b. Generate the tuning script

The script must:

- Include `SET STATISTICS IO ON` and `SET STATISTICS TIME ON` at the top.
- Declare each parameter with a comment identifying its name and purpose.
- Use realistic values obtained from the MCP and the mappings.

**Example output** (`tuning_dbo.aasi_remittance_calculate_offering_level.sql`):

```sql
SET STATISTICS IO ON
SET STATISTICS TIME ON
GO

DECLARE
    @LegalEntityId INT = 42,           -- LegalEntityId: the accounting entity
    @PeriodStatusId INT = 15,          -- PeriodStatusId: fiscal period
    @FundId INT = 3,                   -- FundId: fund dimension
    @StartDate DATETIME = '2025-01-01' -- StartDate: beginning of range

EXEC dbo.aasi_remittance_calculate_offering_level
    @LegalEntityId = @LegalEntityId,
    @PeriodStatusId = @PeriodStatusId,
    @FundId = @FundId,
    @StartDate = @StartDate
```

For **Table-Valued Functions**, use a SELECT call:

```sql
SET STATISTICS IO ON
SET STATISTICS TIME ON
GO

-- @LegalEntityId: the accounting entity
-- @JournalId: the journal being allocated
SELECT *
FROM dbo.aasi_journal_item_allocation_all(
    42,    -- @LegalEntityId
    1001   -- @JournalId
)
```

This file serves as the reproducible test harness for baseline and post-optimization comparisons.

### 3. Analyze the SQL object

- Read the file completely.
- Identify the object type (Stored Procedure, Function, View, Trigger).
- Identify the tables involved by reading their `.hbm.xml` mappings under `DataLayer/` to understand column types, PKs, FKs, and indexes.
- Cross-reference with existing consumers in `*Data.cs` files and other scripts in `DatabaseScripts/`.

### 4. Gather runtime statistics using the SQL Server MCP

Before making any changes, capture baseline performance data using the MCP SQL Server tools:

#### 4a. Enable IO and TIME statistics

Execute via `ExecuteQuery` or `ExecuteStoredProcedure`:

```sql
SET STATISTICS IO ON
SET STATISTICS TIME ON
```

Then run the SQL object (or a representative call) to capture:

- **Logical reads** per table (the most important metric for tuning).
- **Physical reads** (indicates missing data in buffer pool).
- **CPU time** and **elapsed time**.
- **Scan count** (number of times each table was accessed).

#### 4b. Verify existing indexes

Use `GetTableInfo` or `ExecuteQuery` to check current indexes on the involved tables:

```sql
SELECT i.name AS IndexName, i.type_desc,
       STRING_AGG(c.name, ', ') WITHIN GROUP (ORDER BY ic.key_ordinal) AS Columns,
       i.is_unique, i.is_primary_key
FROM sys.indexes i
JOIN sys.index_columns ic ON i.object_id = ic.object_id AND i.index_id = ic.index_id
JOIN sys.columns c ON ic.object_id = c.object_id AND ic.column_id = c.column_id
WHERE i.object_id = OBJECT_ID('<table_name>')
GROUP BY i.name, i.type_desc, i.is_unique, i.is_primary_key
```

#### 4c. Check index usage statistics

```sql
SELECT OBJECT_NAME(s.object_id) AS TableName, i.name AS IndexName,
       s.user_seeks, s.user_scans, s.user_lookups, s.user_updates,
       s.last_user_seek, s.last_user_scan
FROM sys.dm_db_index_usage_stats s
JOIN sys.indexes i ON s.object_id = i.object_id AND s.index_id = i.index_id
WHERE s.object_id = OBJECT_ID('<table_name>')
```

#### 4d. Analyze the query execution plan

Use `AnalyzeQueryPerformance` with `includeExecutionPlan: true` and `includeIndexSuggestions: true` to get:

- The actual/estimated execution plan.
- Missing index suggestions from the optimizer.
- Costly operators (Key Lookups, Table Scans, Hash Joins on large sets, Spools).

#### 4e. Check column statistics

```sql
SELECT s.name AS StatName,
       STRING_AGG(c.name, ', ') WITHIN GROUP (ORDER BY sc.stats_column_id) AS Columns,
       STATS_DATE(s.object_id, s.stats_id) AS LastUpdated,
       s.auto_created, s.user_created
FROM sys.stats s
JOIN sys.stats_columns sc ON s.object_id = sc.object_id AND s.stats_id = sc.stats_id
JOIN sys.columns c ON sc.object_id = c.object_id AND sc.column_id = c.column_id
WHERE s.object_id = OBJECT_ID('<table_name>')
GROUP BY s.name, s.object_id, s.stats_id, s.auto_created, s.user_created
```

Record all baseline results for comparison after optimization.

### 5. Identify performance issues

Look for these common anti-patterns:

- **Non-sargable predicates**: `ISNULL(col, val)`, `CONVERT(col, ...)`, arithmetic on indexed columns, functions wrapping columns in WHERE/JOIN.
- **Implicit conversions**: mismatched types between parameters and columns (e.g., `nvarchar` param against `varchar` column).
- **Missing or suboptimal indexes**: high-selectivity columns in WHERE/JOIN without supporting indexes.
- **RBAR patterns**: cursors, WHILE loops, or scalar UDFs called per-row.
- **Multi-Statement TVFs (MSTVFs)**: replace with inline TVFs when possible.
- **Unnecessary DISTINCT or ORDER BY** in subqueries.
- **Missing SET NOCOUNT ON**.
- **Parameter sniffing vulnerability**: queries sensitive to parameter values without OPTION(RECOMPILE) or local variable copy.
- **Repeated subquery/TVF evaluation**: same expensive result set consumed multiple times without materialization into a #temp table.
- **Missing SCHEMABINDING** on inline TVFs/views (prevents unnecessary spools).
- **Excessive or missing statistics hints**.

### 6. Apply optimizations to the original file

Rewrite the SQL in-place (the original file path) applying the identified improvements. Follow existing project conventions:

- Keep the standard header: `SET QUOTED_IDENTIFIER ON` / `SET ANSI_NULLS ON` / DROP IF EXISTS / CREATE.
- Use `SET NOCOUNT ON` at the top of the body.
- Prefer inline TVFs (`RETURNS TABLE ... WITH SCHEMABINDING`) over MSTVFs.
- Materialize hot results into `#temp` with clustered PK when consumed multiple times.
- Bind parameters with correct types matching the column definition from `.hbm.xml`.
- Use `x = y OR (x IS NULL AND y IS NULL)` instead of `ISNULL(x,-1) = ISNULL(y,-1)`.
- Add query hints only when justified (document why in a comment).

### 7. Validate improvements using the SQL Server MCP

After applying optimizations, re-run the analysis to confirm improvements:

#### 7a. Re-capture IO and TIME statistics

Run the optimized version with `SET STATISTICS IO ON` / `SET STATISTICS TIME ON` and compare:

- Reduction in **logical reads** (target: significant reduction on the largest tables).
- Reduction in **CPU time** and **elapsed time**.
- Elimination of unnecessary **scan counts**.

#### 7b. Compare execution plans

Use `AnalyzeQueryPerformance` again on the new version:

- Confirm Key Lookups are eliminated or reduced.
- Confirm Table/Index Scans are replaced by Seeks where expected.
- Confirm estimated row counts are closer to actual (no cardinality estimation issues).
- Verify no new expensive operators were introduced (Sort spills, Hash spills).

#### 7c. Validate index recommendations were effective

If new indexes were recommended and created, verify they appear in the new plan and that `user_seeks` increases on subsequent executions.

Present a **before vs after** comparison table:

| Metric              | Before | After | Improvement |
| ------------------- | ------ | ----- | ----------- |
| Logical reads (X)   |        |       |             |
| CPU time (ms)       |        |       |             |
| Elapsed time (ms)   |        |       |             |
| Scan count          |        |       |             |
| Key Lookups         |        |       |             |

### 8. Document changes

Create or update a `README-<object_name>.md` file in the same folder as the SQL file. Include:

- **Date** of the tuning session.
- **Summary** of changes made.
- **Rationale** for each significant change.
- **Before/After** key snippets showing the most impactful rewrites.
- **Pending DBA actions** (if any): index creation scripts, statistics updates, etc.

### 9. Generate index recommendations (if applicable)

If new indexes are needed, provide `CREATE INDEX` statements as a separate code block in the README, clearly marked as pending DBA review. Follow the naming convention: `IX_<TableName>_<Column1>[_Column2]`.

## Output format

After completing the routine, **always** provide a summary to the user that includes:

1. List of issues found (categorized by severity: critical, moderate, minor).
2. List of changes applied.
3. Any recommended DBA actions (index creation, statistics updates, plan guide).
4. A diff-friendly description of the most impactful change.
5. **Execution time comparison** (mandatory, always show):

| Run        | CPU time (ms) | Elapsed time (ms) | Logical reads (total) |
| ---------- | ------------- | ------------------ | --------------------- |
| **Before** |               |                    |                       |
| **After**  |               |                    |                       |
| **Delta**  |               |                    |                       |

Include percentage improvement in the Delta row (e.g., `-45%`). If multiple executions were captured, show the average. Always execute both the original (from the `.old.sql` backup) and the optimized version under the same conditions to ensure a fair comparison.
