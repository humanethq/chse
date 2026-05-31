![Humanet](https://github.com/humanethq/chse/blob/main/CHSE.png) 
<div align="center">

<br/>


**Lock-free, SIMD-accelerated deduplication and anomaly detection engine built on Hyperdimensional Computing for high-throughput data pipelines.**

<br/>

[![Status](https://img.shields.io/badge/Status-Production%20Prototype-brightgreen?style=flat-square)](.)
[![Language](https://img.shields.io/badge/Language-Rust-orange?style=flat-square)](.)
[![Architecture](https://img.shields.io/badge/Architecture-HDC%20%2F%20VSA-informational?style=flat-square)](.)
[![License](https://img.shields.io/badge/License-Proprietary-red?style=flat-square)](.)

</div>

---

## What is CHSE?

CHSE is a high-performance data processing engine that uses **Hyperdimensional Computing (HDC)** and **Vector Symbolic Architectures (VSA)** to perform exact deduplication, near-duplicate detection, and anomaly pre-filtering at scale — without locks, indexes, or traditional database overhead.

The core insight: instead of storing records in B-Trees or hash tables, CHSE represents data as high-dimensional vectors and operates on them using SIMD-accelerated linear algebra. The result is a system that is **fundamentally lock-free by construction**, not by engineering effort.

---

## Quick Start

```toml
# Cargo.toml
[dependencies]
chse = "0.1"
```

```rust
use chse::{ChseEngine, HybridDedup};

// Exact deduplication — 145k+ records/sec
let mut engine = ChseEngine::new();

if engine.is_duplicate(&record) {
    // duplicate — skip
} else {
    engine.insert(&record);
    // process new record
}

// Hybrid pipeline — exact + fuzzy (F1: 78.02, 145k QPS)
let mut dedup = HybridDedup::new();

for record in stream {
    if !dedup.process(&record) {
        // unique record
    }
}
```

> **Note:** API is stabilizing. See [`examples/`](examples/) for complete usage.

---

## Benchmark Results

All benchmarks: Apple Silicon ARM64, `RUSTFLAGS="-C target-cpu=native"`, MusicBrainz 50k dataset.

### Hybrid Deduplication Pipeline

| Method | Precision | Recall | F1 | QPS |
|:---|---:|---:|---:|---:|
| MinHash LSH alone | 45.18% | 95.84% | 61.41 | 82,645 |
| **CHSE + MinHash Hybrid** | **66.62%** | **91.73%** | **78.18** | **138,889** |

+27% better precision, +68% higher throughput vs. MinHash alone.

### Core Engine

| Operation | Latency | Throughput |
|:---|---:|---:|
| State Superposition (Write) | 194 ns | **5.15M ops/sec** |
| Query Projection (Read) | 341 ns | **2.93M queries/sec** |
| Exact Dedup (F1 99.38%) | — | **15,000 inserts/sec** |

### Optimization Journey (from baseline to final)

| Phase | F1 | QPS | Δ |
|:---|---:|---:|:---|
| Baseline | 75.60 | 12,000 | — |
| + i8 Quantization | 75.45 | 25,075 | 2.1x |
| + Rayon parallel batch | 78.18 | 138,889 | 11.6x |
| + FxHashSet dedup | 78.02 | ~145,000 | **final** |

---

## Architecture

```
┌──────────────────────────────────────────────────┐
│              CHSE Hybrid Pipeline                │
│                                                  │
│  Record ──→ [ CHSE Exact Layer ]                 │
│                    │ no match                    │
│                    ▼                             │
│             [ MinHash LSH Layer ]                │
│                    │ no match                    │
│                    ▼                             │
│          Insert into both stores                 │
└──────────────────────────────────────────────────┘
```

**CHSE Layer** encodes records as hypervectors and queries chunked superposition state. Catches exact and near-exact duplicates at O(1) amortized cost with SIMD acceleration.

**MinHash LSH Layer** handles records CHSE classifies as new. Jaccard similarity over character shingles catches fuzzy duplicates — spelling variation, missing fields, minor edits.

**Chunked State Management** partitions superposition state to maintain signal-to-noise ratio as record count scales.

---

## Key Properties

**Lock-free writes**
Vector superposition is commutative and associative. Two concurrent writes are two simultaneous additions into high-dimensional space. Write conflicts are structurally impossible — not prevented by mutexes, but impossible by construction.

**O(1) amortized reads**
Queries are dot-product projections. No index traversal. No page lookups. A single SIMD pass extracts the signal.

**Exact + fuzzy deduplication**
CHSE handles exact matches with near-perfect precision. MinHash LSH handles fuzzy matches. Together they outperform either system alone on both F1 and throughput.

**Honest about limits**
Per-user anomaly detection collapses under cold-start conditions (Kaggle PaySim real-world: F1 = 0.17). CHSE is designed for global-profile filtering, high-frequency account monitoring, and deduplication — not sparse per-user behavioral modeling.

---

## Use Cases

**Event & log deduplication**
Log pipelines, clickstream processing, payment event streams. Exact duplicate suppression at 145k+ events/sec.

**Record linkage**
Entity resolution across dirty datasets with spelling variation, missing fields, or inconsistent formatting. Single-pass exact + fuzzy matching.

**Anomaly pre-filter**
Sits upstream of expensive ML inference. Global behavioral signals at nanosecond latency. Filter with CHSE first, run your model only on what matters.

---

## Known Limitations

Understanding where CHSE fails is as important as knowing where it works.

| Limitation | Details |
|:---|:---|
| Cold-start anomaly detection | Per-user modeling requires history. Accounts with ≤1 transaction cannot be profiled. Real-world fraud detection F1 = 0.17 on Kaggle PaySim. |
| Near-duplicate recall (pure CHSE) | Single-field changes produce orthogonal vectors. Pure CHSE near-dup recall: ~8%. Hybrid with MinHash: ~97%. |
| VSA accuracy at scale | SNR degrades as superposed record count grows. Chunk partitioning mitigates this. |
| CXL / RDMA | Distributed memory fabric and zero-copy network I/O are architectural targets, not yet implemented. |

---

## Verification Report

Full documentation of the journey from synthetic benchmarks to real-world testing — including every failure and why:

- SplitMix64 seed de-correlation (critical correctness fix)
- SIMD boundary and i16 overflow corrections
- Synthetic vs. real-world gap (PaySim collapse root cause analysis)
- MinHash baseline comparison
- L3 cache bottleneck diagnosis and quantization solution
- Phase-by-phase optimization with honest trade-off analysis

→ [`docs/verification_report.md`](docs/verification_report.md)

---

## Roadmap

```
VSA Core (SIMD, SplitMix64, i8 quantization)   ████████████████████  COMPLETE
Exact Deduplication Engine                      ████████████████████  COMPLETE
Hybrid Pipeline (CHSE + MinHash LSH)            ████████████████████  COMPLETE
Parallel Batch Processing (Rayon)               ████████████████████  COMPLETE
Global Anomaly Pre-filter                       ████░░░░░░░░░░░░░░░░  IN PROGRESS
Wasm Single Address-Space Runtime               ░░░░░░░░░░░░░░░░░░░░  PLANNED
CXL Memory Fabric Integration                   ░░░░░░░░░░░░░░░░░░░░  FUTURE
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

**Humanet Research · CHSE v1.0 · 2026**

</div>
