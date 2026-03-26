# PostgreSQL Internal Feature Extraction & ML Cost Injection

This repository is a PostgreSQL 14.5 fork that extends the query optimizer with two capabilities:

1. **Internal feature export** — extract raw optimizer state (row estimates, costs, selectivity, index metadata, etc.) during query planning, for use as ML training data
2. **ML estimate injection** — override the planner's cardinality and cost estimates with predictions from external ML models

Based on [lplm (dbis-ukon)](https://github.com/dbis-ukon/lplm).

---

## Components

| Directory | Role |
|-----------|------|
| `postgresql-14.5-selectivity-injection-main/` | Main fork — all optimizer changes live here |
| `pg_hint_plan-REL14_1_4_3/` | Extension for manual plan hints via SQL comments (fallback for edge cases) |

---

## Modified Files

The core changes are in the PostgreSQL planner:

| File | Changes |
|------|---------|
| `src/backend/optimizer/path/costsize.c` | Feature export functions, ML cardinality injection |
| `src/backend/optimizer/path/allpaths.c` | ML cost inference via HTTP, feature serialization |
| `src/backend/commands/explain.c` | Extended `EXPLAIN` output with `nn_*` features and ML cost predictions per plan node |
| `src/include/nodes/pathnodes.h` | `Path` struct extended with 100+ `nn_*` feature fields and ML cost fields |
| `src/backend/utils/misc/guc.c` | New GUC parameters to control ML and export behavior |

---

## GUC Parameters

These parameters are set in `postgresql.conf` or via `SET` command.

### Cardinality injection

| Parameter | Type | Default | Purpose |
|-----------|------|---------|---------|
| `ml_cardest_enabled` | bool | off | Use ML estimates for base-relation (single-table) row counts |
| `ml_joinest_enabled` | bool | off | Use ML estimates for join row counts |
| `ml_cardest_fname` | string | — | Path to file with pre-computed base-relation estimates (one float per line) |
| `ml_joinest_fname` | string | — | Path to file with pre-computed join estimates |
| `debug_card_est` | bool | off | Write old vs. new estimate pairs to `old_new_single_est.txt` / `old_new_join_est.txt` |

### Cost inference via ML server

| Parameter | Type | Default | Purpose |
|-----------|------|---------|---------|
| `use_cost_ml_server` | bool | off | POST path features to an ML server and use predicted costs |
| `use_join_ml_only` | bool | off | Only call ML server for join nodes (skip scan nodes) |
| `addr_cost_ml_server` | string | `http://172.17.0.1:8000/test` | URL of the ML inference server |
| `ml_lambda_worst_case_minimization` | double | 0.0 | Alpha for worst-case cost formula: `mu + alpha * sigma` |

### Feature export (training data)

| Parameter | Type | Default | Purpose |
|-----------|------|---------|---------|
| `print_single_tbl_queries` | bool | off | Export single-table query features to `single_tbl_est_record.txt` |
| `print_sub_queries` | bool | off | Export join query features to `join_est_record_job.txt` |
| `print_plan_opt_path` | bool | off | Enable optimizer path printing |

---

## EXPLAIN Output Extensions (`explain.c`)

Two GUC parameters extend the output of `EXPLAIN` (and `EXPLAIN ANALYZE`) to surface internal optimizer features for every plan node in the query plan tree. These are the primary way to inspect what the planner computed for each node after execution planning.

### GUC Parameters

| Parameter | Type | Default | Purpose |
|-----------|------|---------|---------|
| `print_ml_cost_features` | bool | off | Append ML-predicted costs and uncertainty estimates to each plan node |
| `print_nn_cost_features` | bool | off | Append all raw `nn_*` internal cost features to each plan node |

Both are declared in `guc.c` and exposed via `explain.h`.

---

### `print_ml_cost_features`

When enabled, each node in `EXPLAIN` output gains the following fields showing what the ML cost model predicted for that node:

**Per-node ML predictions:**

| Field | Description |
|-------|-------------|
| `ml_node_startup_cost` | ML-predicted startup cost for this node only |
| `ml_node_total_cost` | ML-predicted total cost for this node only |
| `ml_node_startup_std_cost` | Uncertainty (std dev) of the startup prediction |
| `ml_node_total_std_cost` | Uncertainty (std dev) of the total prediction |

**Aggregated ML costs (up the plan tree):**

| Field | Description |
|-------|-------------|
| `ml_agg_startup_cost` | Cumulative ML startup cost from this node to leaves |
| `ml_agg_total_cost` | Cumulative ML total cost from this node to leaves |
| `ml_agg_startup_std_cost` | Cumulative startup uncertainty |
| `ml_agg_total_std_cost` | Cumulative total uncertainty |

**Join-specific ML internals:**

| Field | Description |
|-------|-------------|
| `ml_inner_run_cost` | ML-predicted run cost for the inner side of a join |
| `ml_inner_rescan_run_cost` | ML-predicted rescan run cost (NestedLoop inner rescans) |
| `ml_inner_rescan_start_cost` | ML-predicted rescan startup cost |

**MergeJoin sort costs:**

| Field | Description |
|-------|-------------|
| `ml_sort_left_cost` | ML-predicted sort cost for the left (outer) input |
| `ml_sort_left_std_cost` | Uncertainty on left sort cost |
| `ml_sort_right_cost` | ML-predicted sort cost for the right (inner) input |
| `ml_sort_right_std_cost` | Uncertainty on right sort cost |

Example output (JSON format):
```json
{
  "Node Type": "Hash Join",
  "Startup Cost": 0.00,
  "Total Cost": 12345.67,
  "ml_node_startup_cost": 0.00000000,
  "ml_node_total_cost": 11800.12345678,
  "ml_node_startup_std_cost": 0.00000000,
  "ml_node_total_std_cost": 420.50000000,
  "ml_agg_startup_cost": 0.00000000,
  "ml_agg_total_cost": 14200.00000000,
  ...
}
```

These fields allow direct comparison between PostgreSQL's built-in cost estimates and ML predictions at each node, which is useful for evaluating model accuracy and diagnosing plan regressions.

---

### `print_nn_cost_features`

When enabled, each plan node in `EXPLAIN` output gains all raw `nn_*` fields that were computed during cost estimation in `costsize.c` and stored on the `Plan` struct. These are the exact feature values that `inference_ml_cost()` sends to the ML server — seeing them in `EXPLAIN` output lets you inspect what the model received as input for each node.

**Row counts and join cardinality:**

| Field | Description |
|-------|-------------|
| `nn_outer_path_rows` | Estimated rows from the outer input path |
| `nn_inner_path_rows` | Estimated rows from the inner input path |
| `nn_path_rows` | Total estimated output rows for this node |
| `nn_per_tuple` | Per-tuple processing cost |
| `nn_outer_matched_rows` | Outer rows that found a join match |
| `nn_outer_unmatched_rows` | Outer rows with no join match |
| `nn_inner_scan_frac` | Fraction of inner relation scanned per outer row |
| `nn_ntuples` | Number of tuples processed |
| `nn_outer_rows` / `nn_inner_rows` | Raw outer/inner row counts |
| `nn_outer_skip_rows` / `nn_inner_skip_rows` | Rows skipped before first match |

**Selectivity and I/O:**

| Field | Description |
|-------|-------------|
| `nn_selectivity` | Overall predicate selectivity for this node |
| `nn_indexselectivity` | Index-specific selectivity |
| `nn_indexcorrelation` | Physical-to-logical order correlation of the index |
| `nn_pages` | Number of heap pages for this relation |
| `nn_spc_seq_page_cost` | Sequential page I/O cost (from tablespace) |
| `nn_spc_random_page_cost` | Random page I/O cost (from tablespace) |
| `nn_pages_fetched` | Pages actually fetched (bitmap scan) |
| `nn_max_pages_fetched` / `nn_min_pages_fetched` | Best/worst-case page fetch counts |
| `nn_max_IO_cost` / `nn_min_IO_cost` | Best/worst-case I/O costs |

**Qualification costs:**

| Field | Description |
|-------|-------------|
| `nn_scan_qual_cost_startup` / `nn_scan_qual_cost_per_tuple` | Cost of evaluating scan predicates |
| `nn_merge_qual_cost_startup` / `nn_merge_qual_cost_per_tuple` | Cost of evaluating merge join conditions |
| `nn_qp_qual_cost_startup` / `nn_qp_qual_cost_per_tuple` | Cost of post-join qualification |
| `nn_hash_qual_cost_startup` / `nn_hash_qual_cost_per_tuple` | Cost of hash join probe predicates |
| `nn_restrict_qual_cost_startup` | Startup cost of restriction qual evaluation |
| `nn_pathtarget_cost_startup` / `nn_pathtarget_cost_per_tuple` | Target list projection costs |

**MergeJoin specifics:**

| Field | Description |
|-------|-------------|
| `nn_mergejointuples` | Estimated tuples produced by merge join |
| `nn_rescannedtuples` | Tuples rescanned during merge join |
| `nn_rescanratio` | Ratio of rescanned to total inner tuples |
| `nn_outerstartsel` / `nn_outerendsel` | Selectivity range for outer merge key |
| `nn_innerstartsel` / `nn_innerendsel` | Selectivity range for inner merge key |
| `nn_skip_mark_restore` | Whether mark/restore is skipped |
| `nn_materialize_inner` | Whether the inner side is materialized |
| `nn_sort_left` / `nn_sort_right` | Whether outer/inner inputs need sorting |

**HashJoin specifics:**

| Field | Description |
|-------|-------------|
| `nn_numbuckets` | Number of hash buckets |
| `nn_numbatches` | Number of hash batches (spill batches) |
| `nn_num_hashclauses` | Number of hash join clauses |
| `nn_outerpages` / `nn_innerpages` | Pages in outer/inner relations |
| `nn_inner_path_rows_total` | Total inner rows including all loops |
| `nn_hashjointuples` | Estimated join output tuples |
| `nn_virtualbuckets` | Virtual bucket count for skew optimization |
| `nn_innerbucketsize` | Average tuples per bucket |
| `nn_innermcvfreq` | Most-common-value frequency for the inner join key |
| `nn_hashentrysize` | Size of each hash table entry in bytes |
| `nn_nbatches` | Final number of batches after estimation |
| `nn_pages_written` / `nn_pages_read` | Spill pages written/read to disk |

**Index specifics:**

| Field | Description |
|-------|-------------|
| `nn_indexStartupCost` / `nn_indexTotalCost` | Cost of the index scan itself |
| `nn_indexpage` | Number of index pages visited |
| `nn_index_pages` | Total pages in the index |
| `nn_index_tuples` | Total tuples in the index |
| `nn_index_tree_height` | B-tree height |
| `nn_index_ncolumns` / `nn_index_nkeycolumns` | Number of index columns |
| `nn_index_unique` | Whether the index enforces uniqueness |
| `nn_index_StartupCost` / `nn_index_indexTotalCost` | Alternate index cost fields |
| `nn_index_numIndexPages` / `nn_index_numIndexTuples` | Index page/tuple counts |
| `nn_index_spc_random_page_cost` | Random I/O cost specific to the index's tablespace |
| `nn_index_num_sa_scans` | Number of ScalarArray scan iterations |
| `nn_indexonly` | Whether this is an index-only scan |
| `nn_loopcount` | Number of outer loops driving the index scan |
| `nn_iscurrentof` | Whether this is a `WHERE CURRENT OF` scan |
| `nn_baserel_pages` / `nn_baserel_allvisfrac` | Base relation pages and visibility fraction |

**Join metadata:**

| Field | Description |
|-------|-------------|
| `nn_jointype` | Join type (integer code: inner, left, semi, anti, etc.) |
| `nn_inner_unique` | Whether the inner side produces at most one match per outer row |
| `nn_has_indexed_join_quals` | Whether join quals can be pushed into an index |
| `nn_match_count` | Expected number of matches per outer row |
| `nn_parallel_worker` | Number of parallel workers for this node |

**Aggregation and grouping:**

| Field | Description |
|-------|-------------|
| `nn_ngroup` | Number of groups (Group node) |
| `nn_group_startup_cost` / `nn_group_run_cost` / `nn_group_input_run_cost` | Group node cost components |
| `nn_aggstrategy` | Aggregation strategy (plain, sorted, hashed, mixed) |
| `nn_numGroups` | Estimated number of output groups |
| `nn_ngroups` / `nn_ngroupcols` | Group counts and number of grouping columns |
| `nn_numgroupcols` / `nn_numpartcols` / `nn_numordercols` | Column counts for grouping/partitioning/ordering |
| `nn_ngroups_limit` | Upper bound on number of groups |
| `nn_num_partitions` | Number of hash partitions for parallel hash agg |
| `nn_depth` | Recursion depth for recursive CTEs |
| `nn_agg_transcost_startup` / `nn_agg_transcost_per_tuple` | Transition function costs |
| `nn_agg_finalcost_startup` / `nn_agg_finalcost_per_tuple` | Final function costs |

**Other node features:**

| Field | Description |
|-------|-------------|
| `nn_nstream` | Number of input streams (Append/MergeAppend) |
| `nn_material_nbytes` | Estimated bytes for Materialize node |
| `nn_comparison_cost` | Per-comparison cost (Sort, MergeJoin) |
| `nn_parallel_tuple_cost` | Cost to pass a tuple to a parallel worker |
| `nn_num_workers` | Number of workers for Gather/GatherMerge |
| `nn_input_width` | Average input tuple width in bytes |
| `nn_output_tables` | Number of output tables (ModifyTable) |
| `nn_partial_path` | Whether this is a partial path for parallelism |

**Pre-computed equation features (`nn_eq_1` through `nn_eq_14`):**

Mathematical relationships derived from the above features, pre-computed before being sent to the ML server. These capture non-linear interactions (e.g., pages × cost, rows × selectivity) that the model uses as engineered inputs rather than computing internally.

---

### How these features flow through the system

```
costsize.c cost_*() functions
    │
    ├─ Compute all nn_* values during cost estimation
    ├─ Store them on the Path struct fields (nn_* prefix)
    └─ [if use_cost_ml_server] POST them to inference server
            │
            └─ ML server returns predicted costs
               → stored as ml_node_*, ml_agg_*, ml_sort_*, ml_inner_*

createplan.c
    │
    └─ Path struct → Plan struct (nn_* and ml_* fields copied over)

explain.c ExplainNode()
    │
    ├─ [if print_ml_cost_features] emit ml_* fields in EXPLAIN output
    └─ [if print_nn_cost_features] emit all nn_* fields in EXPLAIN output
```

The same `nn_*` values that `inference_ml_cost()` sends to the remote server are also the values printed by `print_nn_cost_features` — so you can verify exactly what features the model received for any query by running `EXPLAIN` with this flag on.

---

## Feature Export Functions (`costsize.c`)

These functions serialize internal optimizer state to flat files, which are used as training data for ML models.

### `print_basic_rel(FILE *fp, PlannerInfo *root, RelOptInfo *rel)`

Writes a human-readable summary of a relation to an open file handle.

Output format:
```
RELOPTINFO (table_name): rows=1000 width=100
	baserestrictinfo: col1 > 5, col2 = 'value'
```

Includes:
- Table name(s) covered by the relation
- Estimated row count and average row width
- All base restriction clauses (WHERE predicates on this relation)

### `print_restrictclauses_fp(FILE *fp, PlannerInfo *root, List *clauses)`

Writes a formatted list of restriction clauses (WHERE/JOIN conditions) to a file handle. Iterates over each clause and delegates expression serialization to `fprint_expr()`.

### `fprint_expr(FILE *fp, const Node *expr, const List *rtable)`

Recursively serializes a PostgreSQL expression tree node to text. Handles:

- **Columns** (`Var`) — written as `table.column`
- **Constants** (`Const`) — type-aware formatting:
  - Strings → `'value'`
  - Numbers → raw digits
  - Dates/timestamps → `'2020-01-01'`
  - Intervals → interval syntax
  - Arrays → split and recursively processed
- **Operators** (`OpExpr`) — written as `left op right`
- **Function calls**, **boolean expressions**, **sub-links**

This produces a SQL-like text representation of each predicate, used as the feature string for ML training.

### `print_single_rel(PlannerInfo *root, RelOptInfo *rel)`

Called from `set_baserel_size_estimates()` when `print_single_tbl_queries = on`.

Appends to `single_tbl_est_record.txt`:
```
query: 42
RELOPTINFO (orders): rows=15000 width=64
	baserestrictinfo: o_totalprice > 1000.0
```

Each call corresponds to one base-relation sizing event during planning. The query number (`query_no`) is a global counter that increments with each call, matching the order of estimates in the cardinality injection files.

### `print_join_rel(PlannerInfo *root, RelOptInfo *rel1, RelOptInfo *rel2)`

Called from `calc_joinrel_size_estimate()` when `print_sub_queries = on`.

Appends to `join_est_record_job.txt`:
```
query: 7
==================inner_rel==================:
RELOPTINFO (lineitem): rows=6001215 width=72
	baserestrictinfo: l_shipdate < '1998-09-01'
==================outer_rel==================:
RELOPTINFO (orders): rows=1500000 width=48
	baserestrictinfo: o_orderstatus = 'F'
```

Captures both sides of each join considered by the planner, enabling offline training of join cardinality models.

### `print_query_no(const char *func_name)`

Appends a timestamped entry to `costsize.log`:
```
2024/03/26 17:30:45: pid[12345] in [set_baserel_size_estimates]: query num: 0
```

Used to correlate log entries with query counters when debugging estimate injection.

### `print_est_card(const char *func_name, double card_est)`

Appends a timestamped cardinality estimate to `costsize.log`:
```
2024/03/26 17:30:45: pid[12345] in [calc_joinrel_size_estimate]: 0.450000000
```

---

## ML Cardinality Injection (`costsize.c`)

### `read_from_fspn_estimate(const char *filename)`

Reads pre-computed base-relation cardinality estimates from a plain text file (one float per line) into `card_ests[]`. Called once per session on the first invocation of `set_baserel_size_estimates()` when `ml_cardest_enabled = on`.

The file is generated offline by an FSPN (Factorized Sum-Product Network) model trained on the exported query features.

### `read_from_fspn_join_estimate(const char *filename)`

Same as above, for join estimates. Populates `join_card_ests[]`. Called once per session on the first join estimation.

### `set_baserel_size_estimates(PlannerInfo *root, RelOptInfo *rel)`

**Standard PostgreSQL function**, modified to inject ML estimates.

Original behavior: estimates row count using column statistics (histograms, MCVs) and applies WHERE clause selectivity.

With `ml_cardest_enabled = on`:
1. Loads estimates from `ml_cardest_fname` on the first call
2. Replaces the statistics-based row estimate with `card_ests[query_no]`
3. Increments `query_no` so the next call reads the next estimate in the array
4. If `debug_card_est = on`, writes `old:new` pair to `old_new_single_est.txt` for evaluation

The estimate order must match the order in which `set_baserel_size_estimates()` is called during planning — which is the same order captured by `print_single_rel()` during the training data export phase.

### `calc_joinrel_size_estimate(PlannerInfo *root, RelOptInfo *joinrel, ...)`

**Standard PostgreSQL function**, modified to inject ML join estimates.

Original behavior: multiplies outer × inner row counts by join clause selectivity.

With `ml_joinest_enabled = on`:
1. Loads estimates from `ml_joinest_fname` on the first call
2. Replaces the computed join estimate with `join_card_ests[join_est_no]`
3. Increments `join_est_no`
4. If `debug_card_est = on`, appends `old:new` pair to `old_new_join_est.txt`

Returns the injected estimate clamped to a valid row count via `clamp_row_est()`.

---

## ML Cost Inference via HTTP (`allpaths.c`)

### `inference_ml_cost(char *node_type, Path *path, ...)`

Gated by `use_cost_ml_server = on`. Called inside each `cost_*` function (for all scan and join node types) after the standard cost model runs.

**What it does:**

1. Reads 100+ internal fields from the `Path` struct and any node-type-specific structs (e.g., `IndexPath`, `HashPath`, `MergePath`)
2. Serializes them into a JSON object using cJSON
3. POSTs the JSON to the ML inference server at `addr_cost_ml_server` via libcurl
4. Parses the response JSON for `startup_cost`, `total_cost`, `startup_std_cost`, `total_std_cost`
5. Overwrites the path's cost fields with the predicted values

**Features exported to the server** include:

| Category | Fields |
|----------|--------|
| Row counts | `nn_outer_path_rows`, `nn_inner_path_rows`, `nn_path_rows` |
| Selectivity | `nn_selectivity`, `nn_indexselectivity` |
| Page I/O | `nn_pages`, `nn_pages_fetched`, `nn_spc_seq_page_cost`, `nn_spc_random_page_cost` |
| Qual costs | `nn_scan_qual_cost_startup/per_tuple`, `nn_qp_qual_cost_startup/per_tuple` |
| Join internals | `nn_mergejointuples`, `nn_rescannedtuples`, `nn_numbuckets`, `nn_numbatches`, `nn_hashjointuples` |
| Index metadata | `nn_index_pages`, `nn_index_tuples`, `nn_index_tree_height`, `nn_index_unique` |
| Aggregation | `nn_ngroup`, `nn_group_run_cost`, `nn_agg_transcost_per_tuple`, `nn_agg_finalcost_per_tuple` |
| Uncertainty | `startup_std_cost`, `total_std_cost` (returned from server) |
| Derived features | `nn_eq_1` through `nn_eq_14` (pre-computed mathematical relationships) |

With `ml_lambda_worst_case_minimization = alpha > 0`, costs are adjusted to `mu + alpha * sigma` for worst-case-aware plan selection.

---

## Data Flow

```
TRAINING PHASE
    Query execution
        ↓
    set_baserel_size_estimates()  ←  print_single_rel() → single_tbl_est_record.txt
        ↓
    calc_joinrel_size_estimate()  ←  print_join_rel()   → join_est_record_job.txt
        ↓
    cost_*() functions            ←  inference_ml_cost() logs features
        ↓
    Offline: train FSPN / NN on exported features

INFERENCE PHASE
    Query planning
        ↓
    set_baserel_size_estimates()
        └─ ml_cardest_enabled → read card_ests[] → override row estimate
        ↓
    calc_joinrel_size_estimate()
        └─ ml_joinest_enabled → read join_card_ests[] → override join estimate
        ↓
    cost_*() functions
        └─ use_cost_ml_server → POST features → override startup/total cost
        ↓
    Plan selection (best path by ML-adjusted cost)
```

---

## Output Files

| File | Written when | Content |
|------|-------------|---------|
| `single_tbl_est_record.txt` | `print_single_tbl_queries = on` | Per-table query features (training data) |
| `join_est_record_job.txt` | `print_sub_queries = on` | Per-join query features (training data) |
| `costsize.log` | Always (when relevant functions called) | Timestamped query counters and estimate values |
| `old_new_single_est.txt` | `debug_card_est = on` | `original_estimate:ml_estimate` pairs for base relations |
| `old_new_join_est.txt` | `debug_card_est = on` | `original_estimate:ml_estimate` pairs for joins |

---

## Build

```bash
cd postgresql-14.5-selectivity-injection-main
./configure
make
make install
```

For pg_hint_plan (requires PostgreSQL already installed):

```bash
cd pg_hint_plan-REL14_1_4_3
make
make install
```

---

## Usage Example

```sql
-- Export training data
SET print_single_tbl_queries = on;
SET print_sub_queries = on;

-- Run your workload here to populate the training files

-- Inject FSPN cardinality estimates
SET ml_cardest_enabled = on;
SET ml_cardest_fname = '/path/to/base_estimates.txt';
SET ml_joinest_enabled = on;
SET ml_joinest_fname = '/path/to/join_estimates.txt';

-- Optionally, use ML server for full cost prediction
SET use_cost_ml_server = on;
SET addr_cost_ml_server = 'http://localhost:8000/test';

-- Run optimized query
EXPLAIN ANALYZE SELECT ...;
```
