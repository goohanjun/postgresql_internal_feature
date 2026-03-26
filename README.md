# PostgreSQL Internal Feature Extraction & ML Cost Injection

A PostgreSQL 14.5 fork that exports internal planner state for ML training and injects ML-predicted estimates back into the planner.

Based on [lplm (dbis-ukon)](https://github.com/dbis-ukon/lplm).

---

## What this fork can print

### Training data for cardinality models

Set `print_sub_queries = on` to dump the inner and outer relation info for every join the planner considers, written to `join_est_record_job.txt`:

```
query: 7
==================inner_rel==================:
RELOPTINFO (lineitem): rows=6001215 width=72
	baserestrictinfo: l_shipdate < '1998-09-01'
==================outer_rel==================:
RELOPTINFO (orders): rows=1500000 width=48
```

Each entry includes table names, estimated row count, and all WHERE predicates on that relation serialized as SQL-like text.

### ML-predicted costs in EXPLAIN output

Set `print_ml_cost_features = on` to add ML cost predictions to every node in `EXPLAIN` output:

- `ml_node_startup_cost` / `ml_node_total_cost` — ML-predicted cost for this node
- `ml_node_startup_std_cost` / `ml_node_total_std_cost` — prediction uncertainty
- `ml_agg_startup_cost` / `ml_agg_total_cost` — cumulative ML cost up the plan tree
- `ml_agg_startup_std_cost` / `ml_agg_total_std_cost` — cumulative uncertainty
- `ml_inner_run_cost`, `ml_inner_rescan_run_cost`, `ml_inner_rescan_start_cost` — join inner-side costs
- `ml_sort_left_cost` / `ml_sort_right_cost` + std devs — MergeJoin sort costs

### Raw internal cost features in EXPLAIN output

Set `print_nn_cost_features = on` to add all raw `nn_*` internal planner features to every node in `EXPLAIN` output. These cover row counts, selectivity, I/O costs, qual evaluation costs, join internals (hash buckets, merge selectivity ranges), index metadata, aggregation parameters, and 14 pre-computed equation features (`nn_eq_1` … `nn_eq_14`).

These are the exact values sent to the ML server by `inference_ml_cost()`, so you can verify what the model received for any specific query plan.

---

## GUC Parameters

| Parameter | Purpose |
|-----------|---------|
| `print_sub_queries` | Export join query features to `join_est_record_job.txt` |
| `print_ml_cost_features` | Show ML-predicted costs in `EXPLAIN` per node |
| `print_nn_cost_features` | Show all raw `nn_*` internal features in `EXPLAIN` per node |
| `ml_cardest_enabled` | Override base-relation row estimates with ML predictions |
| `ml_joinest_enabled` | Override join row estimates with ML predictions |
| `ml_cardest_fname` | File of pre-computed base-relation estimates (one float per line) |
| `ml_joinest_fname` | File of pre-computed join estimates |
| `use_cost_ml_server` | POST node features to an ML server and use predicted costs |
| `addr_cost_ml_server` | ML inference server URL (default: `http://172.17.0.1:8000/test`) |
| `ml_lambda_worst_case_minimization` | Alpha for worst-case cost formula: `mu + alpha * sigma` |
| `debug_card_est` | Write `old:new` estimate pairs to debug files |

---

## Build

```bash
cd postgresql-14.5-selectivity-injection-main
./configure
make
make install
```

---

## Usage Example

### Inspecting internal features with `print_nn_cost_features`

Enable the flag and run `EXPLAIN` to see raw planner internals on every node:

```sql
SET print_nn_cost_features = on;
EXPLAIN (FORMAT JSON) SELECT * FROM orders o JOIN lineitem l ON o.orderkey = l.orderkey
WHERE o.totalprice > 1000;
```

Each plan node in the output will include fields like:

```json
{
  "Node Type": "Hash Join",
  "Startup Cost": 0.00,
  "Total Cost": 98345.20,
  "nn_outer_path_rows": 1500000.0,
  "nn_inner_path_rows": 6001215.0,
  "nn_path_rows": 5765432.0,
  "nn_selectivity": 0.95,
  "nn_numbuckets": 131072,
  "nn_numbatches": 1,
  "nn_hashjointuples": 5765432.0,
  "nn_innerpages": 102456,
  "nn_outerpages": 25412,
  "nn_hash_qual_cost_per_tuple": 0.01,
  "nn_eq_1": 154123.5,
  ...
}
```

These values are exactly what gets sent to the ML server when `use_cost_ml_server = on`.

### Using `inference_ml_cost()` to replace costs with ML predictions

Start an ML inference server that accepts POST requests with the `nn_*` feature JSON and returns predicted costs:

```json
// expected response format
{
  "startup_cost": 0.0,
  "total_cost": 91200.5,
  "startup_std_cost": 0.0,
  "total_std_cost": 3100.2
}
```

Then enable the server and run a query:

```sql
SET use_cost_ml_server = on;
SET addr_cost_ml_server = 'http://localhost:8000/test';

-- Optionally use worst-case cost: cost = mean + alpha * std
SET ml_lambda_worst_case_minimization = 1.5;

EXPLAIN SELECT * FROM orders o JOIN lineitem l ON o.orderkey = l.orderkey;
```

The planner will call the ML server for each candidate path during planning, replace the built-in startup/total costs with the predicted values, and select the plan with the lowest ML-adjusted cost. Enable `print_ml_cost_features = on` alongside to see the predictions in `EXPLAIN` output.

```sql
-- Export join features for training
SET print_sub_queries = on;

-- Inject ML cardinality estimates
SET ml_cardest_enabled = on;
SET ml_cardest_fname = '/path/to/base_estimates.txt';
SET ml_joinest_enabled = on;
SET ml_joinest_fname = '/path/to/join_estimates.txt';
```
