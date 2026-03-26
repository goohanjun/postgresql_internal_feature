# PostgreSQL Internal Feature Extraction & ML Cost Injection

A PostgreSQL 14.5 fork that extends the query optimizer to:
1. Export internal planner state as training data for ML models
2. Inject ML-predicted cardinality and cost estimates back into the planner

Based on [lplm (dbis-ukon)](https://github.com/dbis-ukon/lplm).

---

## Components

| Directory | Role |
|-----------|------|
| `postgresql-14.5-selectivity-injection-main/` | Main fork — all optimizer changes live here |
| `pg_hint_plan-REL14_1_4_3/` | Extension for manual plan hints via SQL comments |

### Key modified files

| File | Changes |
|------|---------|
| `src/backend/optimizer/path/costsize.c` | Training data export, ML cardinality injection |
| `src/backend/optimizer/path/allpaths.c` | ML cost inference via HTTP |
| `src/backend/commands/explain.c` | Extended `EXPLAIN` output with internal features and ML predictions |
| `src/include/nodes/pathnodes.h` | `Path` struct extended with `nn_*` feature fields and `ml_*` cost fields |
| `src/backend/utils/misc/guc.c` | New GUC parameters |

---

## GUC Parameters

### Cardinality injection

| Parameter | Type | Purpose |
|-----------|------|---------|
| `ml_cardest_enabled` | bool | Override base-relation row estimates with ML predictions |
| `ml_joinest_enabled` | bool | Override join row estimates with ML predictions |
| `ml_cardest_fname` | string | File of pre-computed base-relation estimates (one float per line) |
| `ml_joinest_fname` | string | File of pre-computed join estimates |
| `debug_card_est` | bool | Write `old:new` estimate pairs to debug files |

### Cost inference via ML server

| Parameter | Type | Purpose |
|-----------|------|---------|
| `use_cost_ml_server` | bool | POST node features to an ML server and use predicted costs |
| `use_join_ml_only` | bool | Only call ML server for join nodes |
| `addr_cost_ml_server` | string | ML inference server URL (default: `http://172.17.0.1:8000/test`) |
| `ml_lambda_worst_case_minimization` | double | Alpha for worst-case cost: `mu + alpha * sigma` |

### Feature export and EXPLAIN extensions

| Parameter | Type | Purpose |
|-----------|------|---------|
| `print_single_tbl_queries` | bool | Export single-table query features to `single_tbl_est_record.txt` |
| `print_sub_queries` | bool | Export join query features to `join_est_record_job.txt` |
| `print_ml_cost_features` | bool | Show ML-predicted costs in `EXPLAIN` output per plan node |
| `print_nn_cost_features` | bool | Show all raw `nn_*` internal features in `EXPLAIN` output per plan node |

---

## EXPLAIN Output Extensions (`explain.c`)

These two flags extend `EXPLAIN` (and `EXPLAIN ANALYZE`) to surface optimizer internals for every node in the query plan tree.

### `print_ml_cost_features`

Adds ML prediction fields to each plan node:

- `ml_node_startup_cost` / `ml_node_total_cost` — ML-predicted cost for this node
- `ml_node_startup_std_cost` / `ml_node_total_std_cost` — prediction uncertainty
- `ml_agg_startup_cost` / `ml_agg_total_cost` — cumulative ML cost up the plan tree
- `ml_agg_startup_std_cost` / `ml_agg_total_std_cost` — cumulative uncertainty
- `ml_inner_run_cost`, `ml_inner_rescan_run_cost`, `ml_inner_rescan_start_cost` — join inner-side costs
- `ml_sort_left_cost` / `ml_sort_right_cost` + std devs — MergeJoin sort costs

Useful for comparing PostgreSQL's built-in estimates against ML predictions node by node.

### `print_nn_cost_features`

Adds all raw `nn_*` fields to each plan node — the exact internal values computed in `costsize.c` during cost estimation. These are the same features that `inference_ml_cost()` serializes and sends to the ML server, covering row counts, selectivity, I/O, qual costs, join internals, index metadata, aggregation parameters, and 14 pre-computed equation features (`nn_eq_1` … `nn_eq_14`).

Useful for inspecting what feature values the model received for a specific query plan.

### Feature flow

```
costsize.c  →  compute nn_* fields, store on Path struct
                └─ [use_cost_ml_server] POST to ML server → get ml_* predictions back

createplan.c  →  copy Path → Plan struct (nn_* and ml_* fields)

explain.c  →  [print_ml_cost_features] emit ml_* per node
              [print_nn_cost_features] emit nn_* per node
```

---

## Cardinality Injection (`costsize.c`)

### `set_baserel_size_estimates()`

Standard PostgreSQL function for estimating single-table row counts, modified to substitute ML predictions when `ml_cardest_enabled = on`.

On the first call it reads all estimates from `ml_cardest_fname` into `card_ests[]`, then uses `card_ests[query_no]` for each subsequent base relation, incrementing `query_no` each time. The estimate order matches the order captured by `print_single_tbl_queries`.

### `calc_joinrel_size_estimate()`

Standard PostgreSQL function for estimating join output rows, modified to substitute ML predictions when `ml_joinest_enabled = on`. Same lazy-load pattern using `join_card_ests[]` and `join_est_no`.

---

## Training Data Export (`costsize.c`)

When `print_single_tbl_queries` or `print_sub_queries` is on, the planner writes query features to flat files during normal query execution.

**`single_tbl_est_record.txt`** — one entry per base-relation sizing event:
```
query: 42
RELOPTINFO (orders): rows=15000 width=64
	baserestrictinfo: o_totalprice > 1000.0
```

**`join_est_record_job.txt`** — one entry per join considered:
```
query: 7
==================inner_rel==================:
RELOPTINFO (lineitem): rows=6001215 width=72
	baserestrictinfo: l_shipdate < '1998-09-01'
==================outer_rel==================:
RELOPTINFO (orders): rows=1500000 width=48
```

Predicate expressions are serialized by `fprint_expr()`, which recursively converts the internal expression tree to SQL-like text (`table.column op value`).

---

## ML Cost Inference (`allpaths.c`)

### `inference_ml_cost()`

Called inside every `cost_*()` function when `use_cost_ml_server = on`. After the standard cost model runs, it serializes 100+ `nn_*` fields from the `Path` struct into JSON, POSTs to the ML server, and overwrites the path's startup/total costs with the response.

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
-- run your workload here

-- Inject ML cardinality estimates
SET ml_cardest_enabled = on;
SET ml_cardest_fname = '/path/to/base_estimates.txt';
SET ml_joinest_enabled = on;
SET ml_joinest_fname = '/path/to/join_estimates.txt';

-- Inspect internal features in EXPLAIN output
SET print_ml_cost_features = on;
SET print_nn_cost_features = on;
EXPLAIN (FORMAT JSON) SELECT ...;
```
