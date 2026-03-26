# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Summary

This is a **query optimization research platform** that investigates whether machine learning models can provide better cardinality estimates than PostgreSQL's built-in heuristics.

### The Problem

PostgreSQL's cost-based optimizer selects execution plans by estimating how many rows each query stage will produce (**cardinality**) and what fraction of rows match filter conditions (**selectivity**). When these estimates are wrong, the planner picks suboptimal plans — wrong join orders, wrong scan methods — causing slow queries. The built-in estimates rely on simple column statistics (histograms, MCVs) that break down for multi-column correlations and complex predicates.

### The Solution: Selectivity Injection

The PostgreSQL fork adds the ability to **inject pre-computed ML estimates** into the planning pipeline, bypassing the default statistics-based estimation. The ML model used is **FSPN (Factorized Sum-Product Networks)**, trained offline on past query workloads.

Key GUC parameters added to the fork:

| Parameter | Purpose |
|-----------|---------|
| `ml_cardest_enabled` | Toggle ML-based base-relation cardinality estimation on/off |
| `ml_joinest_enabled` | Toggle ML-based join cardinality estimation on/off |
| `ml_cardest_fname` | Path to file containing pre-computed base-relation estimates |
| `ml_joinest_fname` | Path to file containing pre-computed join estimates |
| `debug_card_est` | Write old vs. new estimate comparisons for evaluation |

At planning time, estimates are loaded from files into `card_ests[]` / `join_card_ests[]` arrays via `read_from_fspn_estimate()` / `read_from_fspn_join_estimate()`, then substituted into `set_baserel_size_estimates()` and `calc_joinrel_size_estimate()` in `costsize.c`.

### The Two Components

| Component | Role |
|-----------|------|
| **`postgresql-14.5-selectivity-injection-main/`** | PostgreSQL 14.5 fork with ML cardinality estimation built into the planner |
| **`pg_hint_plan-REL14_1_4_3/`** | Extension for manual plan hints via SQL comments — complementary fallback when ML estimates are still insufficient |
| **`postgresql/`** | Legacy/alternate installation |

**pg_hint_plan** hints look like:
```sql
/*+ HashJoin(a b) SeqScan(a) */ SELECT ...
```

The two tools are complementary: ML injection handles the common case automatically; pg_hint_plan handles edge cases manually.

## Build Commands

### PostgreSQL (main fork)

```bash
cd postgresql-14.5-selectivity-injection-main

# Configure (first time or after configure.ac changes)
./configure

# Build
make

# Install
make install
```

### pg_hint_plan extension

```bash
cd pg_hint_plan-REL14_1_4_3

# Build and install (requires PostgreSQL already installed and pg_config in PATH)
make
make install
```

## Running Tests

### PostgreSQL regression tests

```bash
cd postgresql-14.5-selectivity-injection-main/src/test/regress

# Run all parallel tests against a temporary server instance
make check

# Run against an already-running installed server
make installcheck

# Run a single specific test
make check TESTS="test_name"
```

Test SQL inputs live in `src/test/regress/sql/`, expected outputs in `src/test/regress/expected/`. The `pg_regress` binary drives test execution.

### pg_hint_plan tests

```bash
cd pg_hint_plan-REL14_1_4_3
make check
```

Test modules: `init`, `base_plan`, `pg_hint_plan`, `ut-A`, `ut-S`, `ut-J`, `ut-L`, `ut-G`, `ut-R`, `ut-fdw`, `ut-W`, `ut-T`.

## Architecture

### Selectivity injection (the core research feature)

The optimizer pipeline flows: **parser → rewriter → planner (path generation → cost estimation → plan selection) → executor**.

The fork's ML estimation hooks into the cost estimation phase. Key files:

| File | Role |
|------|------|
| `src/backend/optimizer/path/costsize.c` | Main injection site: `set_baserel_size_estimates()` and `calc_joinrel_size_estimate()` replaced with ML predictions |
| `src/backend/optimizer/path/clausesel.c` | Per-clause selectivity + query feature extraction for ML input |
| `src/backend/optimizer/path/allpaths.c` | Path generation; reads ML cost inference results |
| `src/backend/utils/adt/selfuncs.c` | Stock selectivity estimation functions (used as fallback) |
| `src/include/utils/selfuncs.h` | Selectivity API declarations |

### pg_hint_plan extension

Hooks into PostgreSQL's planner via `planner_hook` and `join_search_hook` function pointers. Main files:

| File | Role |
|------|------|
| `pg_hint_plan.c` | Hook registration, hint parsing from SQL comments |
| `core.c` | Core plan modification logic (copied/adapted from PG internals) |
| `make_join_rel.c` | Join order and path overrides |

Hints are specified in SQL comments: `/*+ SeqScan(t1) HashJoin(t1 t2) */`.

### Build system

Uses GNU Autotools (`configure` + `make`). Requires GNU make (not BSD make). The `src/Makefile.global.in` template drives per-module compilation. Contrib modules and extensions build against an installed PostgreSQL via `pg_config`.
