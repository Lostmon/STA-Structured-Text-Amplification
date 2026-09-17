# Android STA (Structured Text Amplification) Vulnerability Index

Official index and tracking hub for the security research conducted by **Manuel García Peña (Lostmon)**.

This research documents vulnerability vectors based on the **STA (Structured Text Amplification)** pattern — an algorithmic and persistence-focused denial-of-service (DoS) class affecting critical Android components (`libminikin.so`, `SystemUI`, `Binder` IPC) and impacting Chromium/Gecko browsers, messaging apps, and LLMs on Android 16.

---

## 🔍 The STA Architectural Pattern

**Structured input ➡️ serialization ➡️ Binder/IPC transaction ➡️ uncaught exception ➡️ ANR or process termination.**

Android lacks early length controls before processing complex layouts or dispatching oversized Intents. The same pattern recurs across multiple layers:

- **libminikin** — line-breaking, glyph shaping, text measurement
- **Binder** — hard 1 MB limit with no safe fallback
- **SavedState / TaskInfo** — serialization without truncation
- **SystemUI** — no graceful degradation on corrupt state

---

## 🗂️ Vector Index

### 🛑 Class A — Binder / SavedState / IPC

Vectors where a structured payload exceeds the 1 MB Binder limit, breaking IPC channels with `TransactionTooLargeException`.

| ID | Vector | Impact | Link |
|----|--------|--------|------|
| **STA-003** | Share Intent Browser Crash | Crash (Chrome 152 / Edge 2026) | [Read analysis](https://lostmon.blogspot.com/2026/08/sta-003-when-sharing-oversized-link.html) |
| **STA-005** | WhatsApp SavedState amplification (×20.6) | Persistent crash loop | [View vector](https://lostmon.blogspot.com/2026/08/whatsapp-when-large-draft-becomes.html) |
| **STA-006/007** | Google Translate → invisible PDF | ANR + TLE | [Read analysis](https://lostmon.blogspot.com/2026/08/sta-006-007-translate-when-translating.html) |
| **STA-009/010/018** | Print Preview Amplification Chain | SystemUI crash loop, TaskPersister corruption | [View full chain](https://lostmon.blogspot.com/2026/09/sta-print-preview-vector.html) |
| **STA-012** | Threads deep link → persistent crash loop | Permanent crash loop | [Read analysis](https://lostmon.blogspot.com/2026/08/sta-012-threads-when-deep-link-becomes.html) |
| **STA-001** | Oversized link → context menu → TLE | Contextual crash | [View vector](https://lostmon.blogspot.com/2026/08/sta-001-when-oversized-link-reaches.html) |

### 📲 Class B — libminikin / Rendering Engine

Vectors demonstrating that application-level mitigations are insufficient: the algorithmic instability resides in AOSP native components.

| ID | Vector | Impact | Link |
|----|--------|--------|------|
| **STA-017** | Long-press link → `breakLineOptimal` O(n²) | ANR (Chrome + Firefox Tier A) | [View stack trace](https://lostmon.blogspot.com/2026/09/upstream-convergence-sta-patterns-in.html) |
| **STA-020** | Omnibox address-bar focus ANR | ANR + Force finish (Chrome + Edge) | [Read analysis (section 5)](https://lostmon.blogspot.com/2026/09/three-vectors-one-root-cause.html) |
| **STA-031** | Paint.measureText() → HarfBuzz shaping | Sixth entry point — Google Docs | [Read analysis](https://lostmon.blogspot.com/2026/09/libminikin-sixth-entry-point.html) |

**Methodology note:** Each Tier A vector includes a complete stack trace, `libminikin.so` BuildId (`4fabe53671b5ead88314c00a1fd6d67d`), and mitigation context.

---

## 📊 Ecosystem Status & Upstream Convergence (2026)

Throughout 2026, technical convergence has been observed across **Chromium, AndroidX, and AOSP**. Although Android security bulletins **have not patched `libminikin`**, parallel mitigations have been identified that validate the STA diagnosis:

| Component | Commit | Mechanism | Date |
|-----------|--------|-----------|------|
| **AOSP InputMethod** | `a438ce17` | SafeList → byte[]/writeBlob | Jan 2026 |
| **AndroidX Credential Manager** | `393e20ae` | LargePayloadSupport (FD) | Apr 2026 |
| **Chromium PDF Selection** | `84b615a0` | SelectionUtils / 100 KB limit | May 2026 |
| **AndroidX NotificationCompat** | `90ffa6a7` | Blocks oversized compat extras | May 2026 |
| **AndroidX PdfView** | `8882927e` | Anchors (~44 B) + async restoration | Jul 2026 |
| **Chromium Oversized Clipboard** | `4751a769` | ContentProvider URI | Aug 2026 |
| **Chromium Native Messaging** | `f5c51669` | `Union(byte[], SharedMemory)` | Aug 2026 |
| **Chromium TLE Telemetry** | `931ee1ab` | `SentMessageSize` + TLE | Sep 2026 |

**Observable convergence:** constrain → redirect → replace → observe.

**Persistent asymmetry:** `libminikin` still lacks a global length gate in `LineBreakOptimizer::computeBreaks()`.

📖 **[Read the full upstream convergence analysis](https://lostmon.blogspot.com/2026/09/upstream-convergence-sta-patterns-in.html)**

---

## 🧪 Research Tooling

- **STA Lab — Android Weight Analyzer:** per-font cost modeling (Roboto, Noto, HarfBuzz shaping), Parcel amplification factor, O(n²) risk detection
- **STA Pattern Generator:** calibrated payloads for research (no weaponization)
- **Pattern Detector:** grapheme, codepoint, and bidi run analysis

---

## 📢 Disclosure & Contact

This research is distributed publicly for technical audit and mobile software resilience purposes. Its visibility actively supports mental-health awareness efforts promoted by the **BojosXtu** initiative.

- **Research log:** [lostmon.blogspot.com](https://lostmon.blogspot.com)
- **Social initiative:** [BojosXtu on Instagram](https://instagram.com)
- **Contact:** `bojosxtu@gmail.com` · `lostmon@gmail.com`
- **Twitter:** `@lostmon`

---

## 📚 Complete Publication Index

### Individual vectors
- [STA-001 — Oversized link → context menu](https://lostmon.blogspot.com/2026/08/sta-001-when-oversized-link-reaches.html)
- [STA-003 — Share Intent crash](https://lostmon.blogspot.com/2026/08/sta-003-when-sharing-oversized-link.html)
- [STA-006/007 — Translate frozen UI](https://lostmon.blogspot.com/2026/08/sta-006-007-translate-when-translating.html)
- [STA-009/010/018 — Print Preview Vector](https://lostmon.blogspot.com/2026/09/sta-print-preview-vector.html)
- [STA-012 — Threads persistent crash loop](https://lostmon.blogspot.com/2026/08/sta-012-threads-when-deep-link-becomes.html)
- [STA-031 — The Sixth Entry Point](https://lostmon.blogspot.com/2026/09/libminikin-sixth-entry-point.html)
- [Three Vectors, One Root Cause — STA-003/017/020](https://lostmon.blogspot.com/2026/09/three-vectors-one-root-cause.html)

### Analysis & correlations
- [Full whitepaper — Resilience Gaps in Android IPC, SavedState and Text Layout](https://lostmon.blogspot.com/2026/07/resilience-gaps-in-android-ipc.html)
- [Upstream Convergence — STA Patterns in Android & Chromium (2026)](https://lostmon.blogspot.com/2026/09/upstream-convergence-sta-patterns-in.html)
- [Upstream Android and Chromium changes in 2026](https://lostmon.blogspot.com/2026/08/upstream-android-and-chromium-changes.html)
- [CDN Tsunami and STA — Same amplification pattern](https://lostmon.blogspot.com/2026/08/cdn-tsunami-and-sta-same-amplification.html)
- [STA — UTF-16 Serialization Density Experiment](https://lostmon.blogspot.com/2026/08/sta-utf-16-serialization-density_01007502893.html)
- [libminikin: 10 years of vulnerability at Android's core](https://lostmon.blogspot.com/2026/07/libminikin-10-anos-de-vulnerabilidad-en_0671440346.html)
- [Algorithmic DoS in libminikin.so](https://lostmon.blogspot.com/2026/06/algorithmic-dos-en-libminikinso.html)
- [Structured Text Amplification (STA) — Conceptual framework](https://lostmon.blogspot.com/2026/06/structured-text-amplification-sta.html)
- [STA: 48 Hours Later — Chrome correlation](https://lostmon.blogspot.com/2026/08/sta-48-hours-later-chrome-correlation.html)

### Reflections & personal context
- [The dilemma of coordinated disclosure](https://lostmon.blogspot.com/2026/08/el-dilema-de-la-divulgacion-coordinada.html)
- [Challenging Android from a couch and a smartphone](https://lostmon.blogspot.com/2026/07/desafiar-android-desde-el-sofa-y-un.html)
- [STA research timeline](https://lostmon.blogspot.com/2026/07/timeline-investigacion-sta-estilos.html)
- [We are not silence](https://lostmon.blogspot.com/2026/09/no-somos-silencio.html)
- [My couch, my lab and refuge](https://lostmon.blogspot.com/2026/08/mi-sofa-mi-laboratorio-y-refugio.html)

---

*Copyright © 2026 Manuel García Peña (Lostmon). All rights reserved.*
