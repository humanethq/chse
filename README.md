![Humanet](https://github.com/humanethq/chse/blob/main/CHSE.png)

<div align="center">

<br/>

**Lock-free, SIMD-accelerated deduplication and anomaly detection engine built on Hyperdimensional Computing for high-throughput data pipelines.**

<br/>

[![Status](https://img.shields.io/badge/Status-Production%20Ready-brightgreen?style=flat-square)](.)
[![Language](https://img.shields.io/badge/Language-Rust-orange?style=flat-square)](.)
[![Architecture](https://img.shields.io/badge/Architecture-HDC%20%2F%20VSA-informational?style=flat-square)](.)
[![QPS](https://img.shields.io/badge/Throughput-526k%20QPS-blueviolet?style=flat-square)](.)
[![LLM Cost](https://img.shields.io/badge/LLM%20Cost%20Reduction-91.6%25-success?style=flat-square)](.)
[![IRS](https://img.shields.io/badge/Info%20Retention-92.7%25-blue?style=flat-square)](.)
[![License](https://img.shields.io/badge/License-Proprietary-red?style=flat-square)](.)

</div>

---

## What is CHSE-X?

CHSE-X is a production-grade, high-performance data engine built on **Hyperdimensional Computing (HDC)** and **Vector Symbolic Architectures (VSA)**. It performs exact deduplication, near-duplicate detection, LLM prompt compression, and anomaly pre-filtering — without locks, indexes, ML models, or traditional database overhead.

**The core insight:** instead of storing records in B-Trees or hash tables, CHSE-X represents every record as a random vector in a 10,240-dimensional mathematical space. Writes are vector additions. Reads are dot products. Write conflicts are structurally impossible — not prevented by locks, but impossible by the mathematics of commutativity.

**The unexpected product:** when we ran LLM prompts through the deduplication engine before sending them to the API, we cut input tokens by **91.6%** and total API costs by **85.3%** — in under 1.5ms, with zero ML inference, zero GPU, zero external dependencies. On one benchmark scenario, raw queries failed completely due to rate limits while compressed queries succeeded 100% of the time.

---

## Quick Start

```toml
# Cargo.toml
[dependencies]
chse = "0.1"
```

```rust
use chse::{ChseEngineX, HybridDedup, ContextCompressor};

// ── Exact deduplication — 526k QPS ──
let mut engine = ChseEngineX::new_with_persistence(250, Some("chse.wal"));

if engine.is_duplicate(&record) {
    // duplicate — skip
} else {
    engine.insert(&record);
    // process new record
}

// ── Hybrid pipeline — exact + fuzzy (F1: 78.20) ──
let mut dedup = HybridDedup::new();
for record in stream {
    if !dedup.process(&record) {
        // unique record
    }
}

// ── LLM Context Compression ──
let compressor = ContextCompressor::new(0.72);  // recommended threshold
let compressed = compressor.compress(&raw_log_text);
// Result: 21,016 tokens → 1,764 tokens in <1.5ms
```

> **Note:** API is stabilizing. See [`examples/`](examples/) for complete usage.

---

## Benchmark Results

All benchmarks: Apple Silicon ARM64, `RUSTFLAGS="-C target-cpu=native -C target-feature=+dotprod"`.

### Full Optimization Journey (12,000 → 526,316 QPS)

| Phase | F1 | QPS | Key Change |
|:---|---:|---:|:---|
| Baseline hybrid pipeline | 75.60 | 12,000 | — |
| + i8 Quantization | 75.45 | 25,075 | StateVector 20KB→10KB, fits L2 cache |
| + Rayon parallel batch | 78.18 | 138,889 | 10-core parallel CHSE scan |
| + SIMD Jaccard + Zero-Alloc LSH | 78.02 | 248,756 | FxHashSet→Vec scratchpad, branchless hash |
| + 3-Level Holographic Hierarchy | **78.20** | **526,316** | Mega-chunk pruning, SoA layout, dotprod |

**43x throughput improvement. F1 score improved simultaneously.**

### Deduplication Pipeline (MusicBrainz 50k)

| Method | Precision | Recall | F1 | QPS |
|:---|---:|---:|---:|---:|
| MinHash LSH alone | 45.18% | 95.84% | 61.41 | 82,645 |
| CHSE alone | 23.16% | 8.24% | 12.16 | 12,000 |
| **CHSE-X + MinHash Hybrid** | **65.50%** | **97.00%** | **78.20** | **526,316** |

### Core Engine Throughput

| Operation | Latency | Throughput |
|:---|---:|---:|
| State Superposition (Write) | 194 ns | **5.15M ops/sec** |
| Query Projection (Read) | 341 ns | **2.93M queries/sec** |
| Exact Dedup (F1 99.38%) | — | **15,000 inserts/sec** |
| Hybrid Pipeline (WAL active) | — | **446,429 QPS** |

### LLM Context Compression — Real Groq API Test (5 Scenarios)

Tested across 5 production scenarios using `llama-3.3-70b-versatile`. Total: 21,016 raw tokens → 1,764 compressed tokens.

| Metric | Raw | CHSE-X Compressed | Improvement |
|:---|---:|---:|---:|
| Input tokens | 21,016 | 1,764 | **−91.6%** |
| API cost | $0.01330 | $0.00196 | **−85.3%** |
| LLM latency | 2,122ms | 1,674ms | **−21.1%** |
| Rate limit failures | ❌ Scenario 1 failed (TPM exceeded) | ✅ 100% success | **Rate-limit rescue** |
| Compression time | — | **avg 1.50ms** | negligible |
| Info retention (IRS) | baseline | **92.70%** | — |
| Exact fact retention | baseline | **100%** | zero critical data loss |

**At scale:** 100,000 agent actions/day → from $399/day to $59/day. **$123,000/year saved per production system.**

### IRS Threshold Curve

| Threshold | Compression | IRS | Recommendation |
|:---|---:|---:|:---|
| 0.40 | 79.64% | 88.50% | Too aggressive |
| 0.55 | 79.43% | 96.00% | Good |
| **0.72** | **79.43%** | **96.00%** | **Recommended** |
| 0.95 | 79.43% | 96.00% | Conservative |

IRS plateaus at 96% from threshold 0.55 onward. Threshold 0.72 is the recommended production value.

---

## Architecture

### Full System Pipeline

```
┌─────────────────────────────────────────────────────────────────┐
│                    CHSE-X Full Pipeline                         │
│                                                                 │
│  Raw Input (log / event / record)                               │
│        │                                                        │
│        ▼                                                        │
│  [ Outlier Protection Layer ]  ←── critical keywords bypass     │
│        │ non-critical                                           │
│        ▼                                                        │
│  [ Timestamp & UUID Masking ]  ←── Rust SIMD, <0.1ms           │
│        │                                                        │
│        ▼                                                        │
│  [ CHSE-X Exact Layer ]        ←── HDC dot-product, O(1)        │
│        │ no match                                               │
│        ▼                                                        │
│  [ MinHash LSH Layer ]         ←── Jaccard similarity           │
│        │ no match                                               │
│        ▼                                                        │
│  [ Insert into both stores ]                                    │
│        │                                                        │
│        ▼                                                        │
│  [ WAL Append ]                ←── Sequential SSD write         │
└─────────────────────────────────────────────────────────────────┘
```

### 3-Level Holographic Hierarchy (526k QPS Core)

```
Query Vector
    │
    ▼
┌─────────────────────────────────────────────┐
│  Level 1: Mega-Chunks  (N=2000 records)     │
│  dot_product ≥ -5600 ? → descend            │
│  else → skip entire subtree (early exit)    │
└──────────────┬──────────────────────────────┘
               │ pass
               ▼
┌─────────────────────────────────────────────┐
│  Level 2: Super-Chunks (N=500 records)      │
│  4× parallel SIMD scan (dot_product_sdot)   │
└──────────────┬──────────────────────────────┘
               │ pass
               ▼
┌─────────────────────────────────────────────┐
│  Level 3: Chunks       (N=250 records)      │
│  2× SIMD scan → threshold decision          │
└─────────────────────────────────────────────┘
```

### LLM Context Compression Pipeline

```
Raw Prompt (21,016 tokens avg)
    │
    ▼
[ Outlier Protection Layer ]     ← "panic", "corruption", "fatal"
    │ critical → bypass direct     keywords bypass dedup entirely
    ▼
[ Timestamp / UUID Masking ]     ← strip dynamic values
    │                               that break similarity
    ▼
[ Holographic Frequency          ← identical lines merged into
  Superposition ]                  "[error] × 84 | 20:00→20:45"
    │
    ▼
[ Jaccard Deduplication ]        ← remove lines with sim ≥ 0.72
    │
    ▼
Compressed Prompt (1,764 tokens) ← avg 1.50ms, zero ML needed
    │
    ▼
[ LLM API ]                      ← 21% faster, 85% cheaper
```

### Hierarchical Agent Memory (L1/L2/L3)

```
GET /api/chse/agent/memory/context?mode=hierarchical

┌─────────────────────────────────────────┐
│  ## [L1] Last 15 minutes                │
│  Raw, uncompressed — instant events     │
├─────────────────────────────────────────┤
│  ## [L2] Last 24 hours                  │
│  CHSE superpose compressed              │
│  "DB timeout × 847 | 09:00→18:00"       │
├─────────────────────────────────────────┤
│  ## [L3] Last 30 days                   │
│  Nightly prune summaries                │
│  "Redis OOM recurring, limit too low"   │
└─────────────────────────────────────────┘
```

### Persistence Architecture

```
              [New Record / Event]
                      │
        ┌─────────────┴─────────────┐
        ▼                           ▼
 [RAM: CHSE-X Core]          [Disk: Sequential WAL]
 • Active StateVector        • Append-only log
 • LSH FlatChainedTable      • BufWriter 128KB buffer
        │                           │
 (Chunk Full: N=250)        (Chunk Full: N=250)
        │                           │
        ▼                           ▼
 [Closed Chunk Store]  ──►  [Flush to SSD]
        │
        ▼ (periodic)
 [Snapshot: zero-copy byte cast]
 [WAL truncated after snapshot]

 Crash Recovery:
 load snapshot → replay WAL → 100% bit-perfect
```

### REST API Layer

```
POST /api/chse/stream/filter
    Input:  { "source": "nginx.log", "event_text": "..." }
    Output: { "is_duplicate": true, "similarity": 0.98, "action": "DROP" }

POST /api/chse/agent/memory/query
    Input:  { "query": "VM replication timeout", "max_context_tokens": 1500,
              "superpose": true }
    Output: { "matched_episodes": [...], "compressed_prompt_context": "..." }

POST /api/chse/dedup
    Input:  { "records": [...], "return_removed": true }
    Output: { "unique": [...], "removed": [...] }

GET  /api/chse/agent/memory/context?mode=hierarchical&l1=15&l2=24&l3=30
    Output: { "l1": "...", "l2": "...", "l3": "..." }

GET  /api/chse/health
    Output: { "status": "ok", "engine": "rust_native", "qps_capacity": 526316 }
```

---

## Key Properties

**Lock-free writes by construction**
Vector superposition is commutative and associative. `S_new = S_old ⊕ V_record`. Two concurrent writes are two simultaneous additions. Write conflicts are mathematically impossible — not engineered away, but structurally absent.

**O(1) amortized reads**
Queries are single SIMD dot-product passes. No index traversal. No page lookups. No lock contention.

**Database-grade durability**
WAL + Snapshot with bit-perfect crash recovery. 446k QPS with persistence active. Classical ACID databases at comparable throughput: ~100x slower.

**Outlier Protection Layer (OPL)**
Critical events (`panic`, `corruption`, `fatal`, `oom`, `segfault`, `auth fail`, `data loss`) bypass deduplication entirely — they are always preserved verbatim in output. Deterministic, not statistical.

**Holographic Frequency Superposition**
Instead of dropping duplicate log lines, CHSE-X merges them with frequency and timestamp metadata: `[Connection Refused | 84× | 20:00→20:45]`. The LLM sees the same information in one line instead of 84.

**Hierarchical Agent Memory**
L1 (last 15 min, raw) → L2 (last 24h, CHSE-compressed) → L3 (last 30 days, nightly summaries). Agent context grows logarithmically, not linearly.

**Zero-dependency LLM compression**
No ML model. No GPU. No external service. Pure Rust SIMD. IRS 92.70%, exact fact retention 100%, avg compression time 1.50ms.

**Honest about limits**
Per-user anomaly detection collapses under cold-start conditions (Kaggle PaySim: F1 = 0.17). We documented the failure in full.

---

## Use Cases

**LLM Cost Reduction**
Run agent logs, support tickets, RAG context, and error traces through CHSE-X before any LLM API call. 91.6% token reduction, 85.3% cost reduction. Also rescues queries that would otherwise fail rate limits.

**Autonomous Agent Memory**
Sleep Mode runs nightly at 03:00 UTC via Celery: reads yesterday's logs → CHSE superpose compress → Groq LLM summarize → save to persistent memory → prune raw logs. Agent memory stays bounded regardless of system uptime.

**Event & Log Deduplication**
Log pipelines, clickstream processing, payment event streams. 526k+ events/sec with WAL durability and OPL protection for critical events.

**Record Linkage**
Entity resolution across dirty datasets. Single-pass exact + fuzzy matching with configurable Jaccard threshold.

**Anomaly Pre-filter**
Sits upstream of expensive ML inference. Filter with CHSE-X first, run your model only on what matters.

---

## Known Limitations

| Limitation | Details |
|:---|:---|
| Cold-start anomaly detection | Per-user modeling requires transaction history. Real-world fraud detection: F1 = 0.17 on Kaggle PaySim. Full analysis in verification report. |
| Near-duplicate recall (CHSE alone) | Single-field changes produce orthogonal vectors by design. Pure CHSE recall: ~8%. Hybrid with MinHash: ~97%. |
| LLM compression on natural language | Optimized for structured/repetitive content (logs, traces, events). General prose compression is less effective. |
| CXL / RDMA | Distributed memory fabric is an architectural target, not yet implemented. |
| ARM64 dotprod optimization | Peak 526k QPS requires AArch64 dotprod. x86 AVX-512 path delivers comparable throughput on modern Intel/AMD. |

---

## Critical Bugs We Found (and Fixed)

Any HDC implementation will hit these. We documented them so you don't have to rediscover them:

**SplitMix64 Seed De-correlation**
Sequential seeds → LCG → correlated hypervectors. Average similarity: +514 (expected: 0). After SplitMix64 hashing: average similarity -0.03. Perfect pseudo-orthogonality.

**SIMD Boundary Alignment**
D=10,000 doesn't divide by LANE_SIZE=32. Last SIMD iteration reads beyond array boundary — undefined behavior. Fixed: D=10,240.

**i16 Silent Overflow**
`reduce_sum()` on i16 wraps silently at 32,767 in dense superposition. Cast to i32 before accumulation.

---

## Verification Report

Full documentation from synthetic benchmarks to real-world testing:

- SplitMix64 seed de-correlation (critical correctness fix)
- SIMD boundary and i16 overflow corrections
- Synthetic vs. real-world gap (PaySim F1=0.17 collapse analysis)
- MinHash baseline comparison
- L3 cache bottleneck and i8 quantization solution
- VP-Tree failure analysis (flat Vec beats trees at N=200)
- Phase-by-phase optimization with trade-off analysis
- Real Groq API test: 5 scenarios, full cost breakdown
- IRS multi-threshold curve analysis

→ [`docs/verification_report.md`](docs/verification_report.md)

---

## Roadmap

```
VSA Core (SIMD, SplitMix64, i8 quantization)     ████████████████████  COMPLETE
Exact Deduplication Engine                       ████████████████████  COMPLETE
Hybrid Pipeline (CHSE-X + MinHash LSH)           ████████████████████  COMPLETE
Parallel Batch Processing (Rayon)                ████████████████████  COMPLETE
3-Level Holographic Hierarchy                    ████████████████████  COMPLETE
WAL + Snapshot Persistence                       ████████████████████  COMPLETE
C-FFI + Python Bridge                            ████████████████████  COMPLETE
REST API (stream/filter, memory/query, dedup)    ████████████████████  COMPLETE
LLM Context Compressor (Groq validated)          ████████████████████  COMPLETE
Timestamp/UUID Masking                           ████████████████████  COMPLETE
Holographic Frequency Superposition              ████████████████████  COMPLETE
Outlier Protection Layer (OPL)                   ████████████████████  COMPLETE
L1/L2/L3 Hierarchical Agent Memory               ████████████████████  COMPLETE
Streaming Compression (StreamingCompressor)      ████████████████████  COMPLETE
Sleep Mode / Neural Pruning (Celery 03:00 UTC)   ████████████████████  COMPLETE
Shamir's Secret Sharing (Prism Service)          ████████████████████  COMPLETE
Decentralized DHT Block Storage (P3 Service)     ████████████████████  COMPLETE
Smart Structure-Aware Chunking                   ████░░░░░░░░░░░░░░░░  IN PROGRESS
Adaptive Threshold Controller                    ░░░░░░░░░░░░░░░░░░░░  PLANNED
Wasm Single Address-Space Runtime                ░░░░░░░░░░░░░░░░░░░░  FUTURE
CXL Memory Fabric Integration                    ░░░░░░░░░░░░░░░░░░░░  FUTURE
```

---

## Research Foundation

Built on published research in hyperdimensional computing, vector symbolic architectures, and lock-free systems.

Kanerva (1988, 2009) · Plate (1995) · Gayler (2003) · Herlihy & Shavit — *The Art of Multiprocessor Programming*

---

## Contact

Research inquiries, collaboration, and enterprise licensing:

**[contact@humanet.uk](mailto:contact@humanet.uk)**

---

<div align="center">

<br/>

*"The bottleneck is never the hardware. It is always the abstraction we placed between ourselves and it."*

<br/>

**Humanet Research · CHSE-X v1.0 · 2026**

</div>
