# 🛡️ Theoretical Framework — Structured Text Amplification (STA)

> **Independent technical research developed by Manuel García Peña ([@Lostmon](https://github.com/Lostmon))**
> *A conceptual model of algorithmic denial of service, resilience failures in Binder IPC channels, SavedState persistence, and instability in Android's native rendering.*

---

## 📢 Project Philosophy and Social Commitment

This research documents architectural failures while demonstrating technical and personal resilience. All work is carried out from a home environment, with a mid-range phone, with no lab and no internal access to source code. Its visibility supports the mental-health awareness efforts promoted by **BojosXtu**.

- **Official blog:** [lostmon.blogspot.com](https://lostmon.blogspot.com)
- **Social initiative:** [BojosXtu on Instagram](https://instagram.com/bojosxtu)
- **Contact:** bojosxtu@gmail.com · lostmon@gmail.com

---

## 🔬 The STA Structural Pattern: Formal Definition

> *Structured Text Amplification (STA) is an architectural resource-amplification pattern in which structured textual input traverses multiple software layers, causing a progressive increase in computational cost, memory consumption, or state propagation, until it exceeds the stability limits of one or more downstream components.*

**General pipeline:**

```
Structured input (URL / HTML / Intent / Clipboard / Deep Link)
↓
Parser / Validation (absent or insufficient)
↓
Framework APIs
↓
Binder IPC (hard limit 1 MB)
↓
Rendering Engine (TextView → StaticLayout → LineBreaker → libminikin) ← Class B
↓
Layout / Compose
↓
State Persistence (TaskPersister / SavedStateRegistry) ← Class A
↓
SystemUI / OEM recovery
↓
Observed failure (ANR / Crash Loop / Hard Reboot)
```

### Recurring properties

| Property | Description |
|-----------|-------------|
| **Structured input** | Hierarchical or encoded textual information (URLs, HTML, JSON, Intent extras, Markdown), not arbitrary binary data. |
| **Multi-stage processing** | The same logical content traverses multiple independent framework layers. |
| **Progressive amplification** | Processing cost grows at each successive stage. |
| **Shared infrastructure** | Unrelated apps reuse the same Android components (libminikin, FragmentManager, TaskPersister, SystemUI). |
| **Non-linear behavior** | Resource consumption is disproportionate to input size. |
| **Cross-component impact** | Failures appear far from the original entry point (e.g. a click in the browser → a SystemUI crash). |

---

## 🧩 Classification: Class A and Class B

### Class A — IPC / SavedState Amplification

**Mechanism:** State serialization → Bundle → Parcel → Binder (>1 MB).

**Exhausted resource:** Binder transaction buffer (1,048,576 bytes).

**Amplification factor:** Depends on Fragment nesting depth and associated metadata. Observed between ×2.8 and ×20.6 in field measurements.

**Result:** Uncaught `TransactionTooLargeException` → persistent crash loop.

**Representative vectors:** STA-001, STA-003, STA-005, STA-006/007, STA-009/010/018, STA-012.

### Class B — Interaction Surface Amplification

**Mechanism:** Synchronous text layout on the UI thread (libminikin line breaking + glyph shaping).

**Exhausted resource:** Main-thread CPU time (ANR threshold ≈5 s).

**Amplification factor:** 10–20× multiplier before entering the O(n²)/greedy-blocking path. The density of special characters (# / % €, U+2800) drives up the number of line-break candidates.

**Result:** Main-thread blocking → ANR (5–16 s).

**Representative vectors:** STA-017, STA-020, STA-031.

**Critical note:** Both classes share the same root-cause category (absence of length validation before expensive operations), but **are not reducible to a single mathematical model**. Fixing one does not address the other.

---

## 🗂️ Index of Documented Vectors

The research catalogs **32 reproducible vectors** across two classes:

- **Class A (Binder / SavedState / IPC):** Transactions exceeding the 1 MB limit, triggering `TransactionTooLargeException`.
- **Class B (libminikin / Rendering):** Algorithmic complexity in native components causing critical ANRs.

The full catalog, with CVSS 3.1, evidence tier (Tier A/B), and persistence, is in the [main README](./README.md).

---

## 🛠️ Anatomy of the Block and Forensic Evidence

Production dumps confirm blocks on the **UI thread** exceeding Android's 5-second watchdog threshold, with the main thread executing native code in `libminikin.so`:

```
"main" prio=5 tid=1 Native   ← UI THREAD BLOCKED
| state=R

native: minikin::getPrevWordBreakForCache     libminikin.so
native: minikin::StyleRun::getLineMetrics     libminikin.so
native: minikin::MeasuredText::getLineMetrics libminikin.so
native: minikin::LineBreakOptimizer::computeBreaks  ← O(n²) path
native: minikin::breakLineOptimal             libminikin.so
native: android::nComputeLineBreaks           libhwui.so
```

**`libminikin.so` BuildId:** `4fabe53671b5ead88314c00a1fd6d67d`

The same BuildId appears across multiple apps (Chrome, Edge, Firefox, Google Docs), confirming the problem lies in the system component, not in each application's own implementation.

Full stack traces, Binder diagnostics, and forensic captures are documented on the author's blog.

---

## 📊 Upstream Convergence (2026)

During 2026, a **technical convergence** has been observed across AOSP, AndroidX, and Chromium: multiple projects have introduced parallel mitigations aimed at limiting heavy payloads, redirecting transactions, and observing boundary failures in Binder.

Consistent pattern: **constrain → redirect → replace → observe**.

However, **`libminikin` remains the only investigated surface without a global length guard** in `LineBreakOptimizer::computeBreaks()`.

Full analysis: [Upstream Convergence — STA Patterns in Android & Chromium (2026)](https://lostmon.blogspot.com/2026/09/upstream-convergence-sta-patterns-in.html)

---

## 📚 Supplementary Documentation

- [Main README](./README.md) — Vector index and full publication
- [Whitepaper v6 — Resilience Gaps in Android IPC, SavedState and Text Layout](https://lostmon.blogspot.com/2026/07/resilience-gaps-in-android-ipc.html)
- [Upstream Convergence Analysis](https://lostmon.blogspot.com/2026/09/upstream-convergence-sta-patterns-in.html)
- [Three Vectors, One Root Cause](https://lostmon.blogspot.com/2026/09/three-vectors-one-root-cause.html)
- [The Sixth Entry Point — HarfBuzz Shaping](https://lostmon.blogspot.com/2026/09/libminikin-sixth-entry-point.html)

---

## 📢 Disclosure and Responsible Use

This repository **does not contain weaponized payloads**. The documentation is distributed for technical auditing, software resilience, and framework analysis purposes.

Reproduction steps are included so vendors and security teams can validate and mitigate the documented patterns.

If you work in mobile security, framework analysis, or software resilience, this framework is designed so you can contribute to closing the architectural gap.

---

*Copyright © 2026 Manuel García Peña (Lostmon). All rights reserved.*
