# Content-Aware-Caching-Algorithm



# Content-Aware Caching in xv6 (RISC-V)

A metadata-driven caching layer for xv6 that outperforms traditional LRU by leveraging 
file **type**, **size**, and **historical access frequency** to guide caching, promotion, and eviction.

---

## ✨ Key Ideas

- **Content-aware scoring**: Each cache entry carries file metadata and an **importance** counter that increments on hits.
- **Promotion on threshold**: Hot entries crossing a threshold are moved to the list head for faster future access.
- **Evict by least-importance**: On space pressure, remove the entries with the lowest importance first.
- **Simple + fast**: Singly linked list structure; minimal overhead suitable for xv6 constraints.
- **Extensible**: Tracks last-used timestamps so you can add time decay or hybrid LFU/LRU later.

---

## 🧱 Architecture


+---------------------------+
\| User Applications         |
+---------------------------+
\| System Call Layer         |
+---------------------------+
\| Content-Aware Cache Layer |
+---------------------------+
\| Disk Storage              |
+---------------------------+



**Cache entry fields (conceptual):**
- `filename`, `size`, `data*`
- `importance` (hit counter)
- `last_used` (logical clock)
- `next` (singly linked list)

---

## 🧪 Benchmarks (xv6 simulation)

**Workload**: 100 files, 20,000 accesses (standard) + 10,000 (important-files burst), cache ≈ **6 MB**.

### Baseline vs Proposed (LRU → Content-Aware)

| Metric | LRU | Content-Aware | Absolute Δ | Relative change vs LRU |
|---|---:|---:|---:|---:|
| **Hit rate** | 47.02% | 75.83% | **+28.81 pp** | **+61.27%** |
| **Disk reads** | 10,597 | 4,835 | **−5,762** | **−54.37%** |
| **Execution time** | 935 ms | 809 ms | **−126 ms** | **−13.48%** |
| **Cache entries stored** | 26 | 73 | **+47** | **+180.77%** |
| **Cache utilization** | 74.6% | 94.1% | **+19.50 pp** | **+26.14%** |

> **Notes**
> - “pp” = **percentage points** (difference of percentages).
> - “Relative change” = (Proposed − LRU) / LRU.

**Takeaways**
- Metadata signals predict future accesses better than recency alone.
- Big cut in disk I/O translates to lower latency even with minor compute overhead.

---

## ⚙️ Build & Run (xv6-riscv)

> Requires: Linux/macOS, RISC-V toolchain, `qemu`

# Build & run xv6 in QEMU
make qemu


Inside xv6, run your file-access tests or workload generator to observe cache stats (hit/miss, evictions). If you’ve added a debug flag, rebuild with:


make clean && make CFLAGS+='-DCACHE_DEBUG'

---

## 🔍 How It Works (Algorithm sketch)

**Lookup**

1. Scan list for `filename`.
2. On **hit**: increment `importance`, update `last_used`; if `importance > THRESHOLD` and not already head → promote to head.
3. On **miss**: read from disk; evict least-importance entries until there’s space; insert new entry at head with `importance=1`.

**Eviction**

* Repeatedly remove the **lowest-importance** entries (ties may use oldest `last_used`) until `current_size + new_size ≤ max_size`.

This yields a simple, fast path with predictable behavior under xv6 constraints.

---

## 📁 Repo Hints

Typical xv6 touchpoints for a file-system cache:

* `fs/` and `kernel/` areas for cache hooks around read paths
* A small `cache/` or `kernel/cache*` set of sources for the data structures & policy
* Optional `user/` test programs to stress the cache (sequential, hot-set, mixed-size, bursty)

> If your repo layout differs, add a short section listing the exact files you modified (e.g., `fs.c`, `bio.c`, `buf.h`, `cache.c`).

---

## 🧩 Configuration

* `CACHE_MAX_SIZE` — total cache bytes (default ≈ 6 MB in experiments)
* `THRESHOLD` — promotion threshold for “hot” entries
* `DECAY` (optional) — if you enable time decay
* `TIE_BREAK` — policy on equal-importance: oldest first or largest first

Expose these via `#define`s or a small config header so experiments are reproducible.

---

## 🔬 Reproducing Results

1. **Set parameters** used in the paper:

   * Cache size ≈ **6 MB**
   * Workloads: **20,000** mixed accesses + **10,000** important-files burst
2. **Build**: `make clean && make`
3. **Run** workload tool inside xv6; capture logs (hits/misses, evictions).
4. **Compute metrics** from logs and compare to the table above.

> Tip: Emit CSV-friendly lines from the kernel (guarded by `#ifdef`) and post-process with a tiny Python script.

---

## 🛣️ Roadmap

* Adaptive thresholds (learned from access stats)
* Hybrid LFU/LRU with time decay
* Concurrency tuning for heavy parallelism
* Multi-level cache experiments (RAM ↔ SSD ↔ HDD)
* Port to a Unix-like OS layer for real-world benchmarking

---

## 📚 Background Reading

* Context-Aware Proactive Caching — Zheng et al., arXiv:1606.04236
* Semantics-Aware Replacement (SACS) — Negrão et al., JISA (2015)
* Age-of-Information Caching — Ahani & Yuan, arXiv:2111.11608

---

## 👤 Author

**Vatsal Saxena**
