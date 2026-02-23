Executive Summary

StreamPulse currently processes 80 million daily plays using a single-machine architecture (64 GB RAM, 16 cores). Two pipelines are at immediate scalability risk due to memory and compute bottlenecks. One pipeline is already exceeding the batch window and requires urgent horizontal scaling. Two pipelines are operating safely within limits and should remain on Pandas. Immediate action is required to prevent production instability within the next 6–12 months as data grows 30% annually.

Pipeline A: Daily Play Counts
Overview

Input: 12 GB CSV (~80M rows)
Runtime: 2.5 hours
Memory: 58 GB peak (91% utilization)
Growth rate: 30% YoY

Bottleneck: Memory

Pandas typically requires 2–3× the raw file size in memory due to object overhead and intermediate aggregations.
12 GB file → ~58 GB memory usage
58 GB / 64 GB = 91% utilization

At 30% growth:
Year 1 → ~75 GB required
Year 2 → ~98 GB required
This will exceed available RAM and cause MemoryError.

Risk Level: CRITICAL
Already near capacity
Will fail within ~6–8 months
Core business function (marketing + trending charts)

Diagnosis:
Primary bottleneck is memory pressure during groupby + sorting.
Runtime (2.5h) is acceptable today but will increase to ~4.2h in 2 years.

Recommendation
Immediate Optimization (Quick Win)
Only 3 of 15 columns are required.

df = pd.read_csv(file, usecols=['song_id', 'artist_id', 'played_at'])

Explanation:
usecols tells Pandas to read only required columns from disk.
This reduces memory footprint significantly (potentially cutting usage by ~70–80%).

Short-Term (This Sprint)
Drop unused columns immediately
Ensure correct dtypes (int32, category)
Pre-aggregate in chunks

Medium-Term:
Migrate to Spark local mode
Long-Term:
Move to Spark cluster when daily data exceeds ~20 GB

Pipeline B: User Recommendation Model
Overview

Input: 5M users × 200 features (1B values)
Runtime: 9 hours (EXCEEDS 6-hour batch window)
Memory: 22 GB peak
Bottleneck: Compute (CPU-bound)
Memory usage is safe (22 GB of 64 GB).

The issue:
Users are processed sequentially
5 million loop iterations
No parallelization
Risk Level: CRITICAL (Already Broken)
Runtime = 9h
Batch window = 6h
Pipeline already failing SLA

Why Vertical Scaling Won’t Fix It
Adding more RAM will not help because:
The issue is CPU-bound sequential processing
Python loop cannot utilize all 16 cores effectively
Even upgrading to 32 cores would not fully solve scaling beyond next year.

Recommendation:
Immediate
Replace sequential loop with vectorized NumPy operations
Use multiprocessing on 16 cores

Short-Term (High Priority):
Migrate to Spark cluster
Distribute scoring across partitions (users split across executors)
Strategic Direction:
Move recommendation scoring to distributed compute permanently.

Pipeline C: Song Metadata Sync
Overview
Input: 50 MB (2,000–5,000 songs)
Runtime: 25 minutes
Memory: 200 MB
Bottleneck: I/O (API latency)
Sequential REST calls.

Risk Level: LOW
Tiny dataset
Well within batch window
Memory negligible
Spark overhead would exceed benefit

Recommendation
Stay on single machine.

Optional improvement:
Use async requests for faster API calls
No distributed migration needed.

Pipeline D: Revenue Attribution
Overview
Input: 15 GB across 3 tables
Runtime: 4.5 hours
Memory: 52 GB peak
Joins spike memory usage

Bottleneck: Memory (Join amplification)
Large joins create temporary DataFrames:
80M plays × 10M subscriptions
Intermediate objects spike RAM

At 30% growth:
Data → ~19.5 GB
Memory spike → likely >67 GB
Runtime → ~5.8 hours (danger zone)
Risk Level: HIGH (Becomes Critica in 12 months)
Currently fits but:
52 GB / 64 GB = 81% utilization
Only 1.5h buffer left in batch window

Recommendation:
Immediate Optimizations
Push joins to PostgreSQL (SQL pushdown)
Filter early
Use indexed joins
Ensure correct join keys

Short-Term:
Test Spark local mode for join-heavy workload

Medium-Term:
Migrate to Spark cluster when data >20 GB

Pipeline E: A/B Test Analytics
Overview
Input: 1.5 GB total (3 experiments)
Runtime: 8 minutes per experiment
Memory: 3 GB
Bottleneck: None (Healthy)
Risk Level: LOW
Fast
Small data
Minimal memory
No SLA issues

Recommendation
Keep on Pandas + SciPy.
Distributed computing would add unnecessary complexity.

Scalability Audit Summary:


| Pipeline                | Bottleneck     | Data Size | Runtime | Memory | Risk     | Recommendation                         |
| ----------------------- | -------------- | --------- | ------- | ------ | -------- | -------------------------------------- |
| A: Daily Play Counts    | Memory         | 12 GB     | 2.5h    | 58 GB  | CRITICAL | Optimize immediately, migrate to Spark |
| B: User Recommendations | Compute        | 1B values | 9h      | 22 GB  | CRITICAL | Distribute scoring via Spark cluster   |
| C: Song Metadata Sync   | I/O            | 50 MB     | 25m     | 200 MB | LOW      | Stay on Pandas                         |
| D: Revenue Attribution  | Memory (joins) | 15 GB     | 4.5h    | 52 GB  | HIGH     | Optimize SQL, migrate next             |
| E: A/B Test Analytics   | None           | 1.5 GB    | 8m      | 3 GB   | LOW      | No migration needed                    |


Migration Priority Roadmap
Priority 1 — Immediate (This Sprint)

Pipeline B: User Recommendations
Reason: Already exceeding batch window
Action: Parallelize immediately → Begin Spark cluster migration

Priority 2 — Short-Term (Next 2 Sprints)

Pipeline A: Daily Play Counts
Reason: Memory at 91%, will fail within months
Action: Optimize columns → Move to Spark local

Priority 3 — Medium-Term (Next Quarter)

Pipeline D: Revenue Attribution
Reason: Join-heavy, will exceed memory in 12 months
Action: SQL pushdown → Evaluate Spark cluster

No Migration Needed
Pipelines C and E
Reason:
Small data
Stable runtime
No memory pressure
No SLA risk

Final Recommendation to CTO

Two pipelines (Recommendations and Daily Play Counts) require immediate distributed computing adoption. Revenue Attribution should be optimized now and prepared for migration within 12 months. The remaining pipelines should stay on Pandas to avoid premature distribution. A phased Spark adoption strategy minimizes risk while avoiding unnecessary complexity.