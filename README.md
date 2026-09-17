# Android STA (Structured Text Amplification) Vulnerability Index

Repositorio oficial y centro de seguimiento de la investigación de seguridad desarrollada por **Manuel García Peña (Lostmon)**.

La investigación documenta los vectores basados en el patrón **STA (Structured Text Amplification)** — denegación de servicio algorítmica y de persistencia que afecta a componentes críticos de Android (`libminikin.so`, `SystemUI`, IPC `Binder`) e impacta a navegadores Chromium/Gecko, apps de mensajería y modelos LLM en Android 16.

---

## 🔍 El Patrón Estructural STA

**Entrada estructurada ➡️ serialización ➡️ transacción Binder/IPC ➡️ excepción no capturada ➡️ ANR o terminación de proceso.**

Android carece de controles de longitud tempranos antes de procesar layouts complejos o despachar Intents de gran tamaño. El mismo patrón recurre en múltiples capas:

- **libminikin** — line-breaking, glyph shaping, medición de texto
- **Binder** — límite duro de 1 MB sin safe fallback
- **SavedState / TaskInfo** — serialización sin truncation
- **SystemUI** — sin degradación graceful ante estado corrupto

---

## 🗂️ Índice de Vectores

### 🛑 Clase A — Binder / SavedState / IPC

Vectores donde un payload estructurado desborda el límite de 1 MB de Binder, rompiendo canales IPC con `TransactionTooLargeException`.

| ID | Vector | Impacto | Enlace |
|----|--------|---------|--------|
| **STA-003** | Share Intent Browser Crash | Crash (Chrome 152 / Edge 2026) | [Leer análisis](https://lostmon.blogspot.com/2026/08/sta-003-when-sharing-oversized-link.html) |
| **STA-005** | WhatsApp SavedState amplification (×20.6) | Crash loop persistente | [Ver vector](https://lostmon.blogspot.com/2026/08/whatsapp-when-large-draft-becomes.html) |
| **STA-006/007** | Google Translate → PDF invisible | ANR + TLE | [Leer análisis](https://lostmon.blogspot.com/2026/08/sta-006-007-translate-when-translating.html) |
| **STA-009/010/018** | Print Preview Amplification Chain | SystemUI crash loop, TaskPersister corrupto | [Ver cadena completa](https://lostmon.blogspot.com/2026/09/sta-print-preview-vector.html) |
| **STA-012** | Threads deep link → crash loop persistente | Crash loop permanente | [Leer análisis](https://lostmon.blogspot.com/2026/08/sta-012-threads-when-deep-link-becomes.html) |
| **STA-001** | Oversized link → context menu → TLE | Crash contextual | [Ver vector](https://lostmon.blogspot.com/2026/08/sta-001-when-oversized-link-reaches.html) |

### 📲 Clase B — libminikin / Rendering Engine

Vectores que demuestran que las mitigaciones a nivel de aplicación son insuficientes: la inestabilidad algorítmica reside en los componentes nativos de AOSP.

| ID | Vector | Impacto | Enlace |
|----|--------|---------|--------|
| **STA-017** | Long-press link → `breakLineOptimal` O(n²) | ANR (Chrome + Firefox Tier A) | [Ver stack trace](https://lostmon.blogspot.com/2026/09/upstream-convergence-sta-patterns-in.html) |
| **STA-020** | Omnibox address-bar focus ANR | ANR + Force finish (Chrome + Edge) | [Leer análisis (sección 5)](https://lostmon.blogspot.com/2026/09/three-vectors-one-root-cause.html) |
| **STA-031** | Paint.measureText() → HarfBuzz shaping | Sexto entry point — Google Docs | [Leer análisis](https://lostmon.blogspot.com/2026/09/libminikin-sixth-entry-point.html) |

**Nota metodológica:** Cada vector Tier A incluye stack trace completo, BuildId de `libminikin.so` (`4fabe53671b5ead88314c00a1fd6d67d`) y contexto de mitigación.

---

## 📊 Estado del Ecosistema y Convergencia Upstream (2026)

A lo largo de 2026 se ha observado convergencia técnica en **Chromium, AndroidX y AOSP**. Aunque los boletines de seguridad de Android **no han parcheado `libminikin`**, se han identificado mitigaciones paralelas que validan el diagnóstico STA:

| Componente | Commit | Mecanismo | Fecha |
|------------|--------|-----------|-------|
| **AOSP InputMethod** | `a438ce17` | SafeList → byte[]/writeBlob | Ene 2026 |
| **AndroidX Credential Manager** | `393e20ae` | LargePayloadSupport (FD) | Abr 2026 |
| **Chromium PDF Selection** | `84b615a0` | SelectionUtils / 100 KB limit | May 2026 |
| **AndroidX NotificationCompat** | `90ffa6a7` | Bloquea compat extras oversized | May 2026 |
| **AndroidX PdfView** | `8882927e` | Anchors (~44 B) + async restoration | Jul 2026 |
| **Chromium Oversized Clipboard** | `4751a769` | ContentProvider URI | Ago 2026 |
| **Chromium Native Messaging** | `f5c51669` | `Union(byte[], SharedMemory)` | Ago 2026 |
| **Chromium TLE Telemetry** | `931ee1ab` | `SentMessageSize` + TLE | Sep 2026 |

**Convergencia observable:** constrain → redirect → replace → observe.

**Asimetría persistente:** `libminikin` sigue sin barrera global de longitud en `LineBreakOptimizer::computeBreaks()`.

📖 **[Leer el análisis completo de convergencia upstream](https://lostmon.blogspot.com/2026/09/upstream-convergence-sta-patterns-in.html)**

---

## 🧪 Herramientas de Investigación

- **STA Lab — Android Weight Analyzer:** modelo de coste por fuente (Roboto, Noto, HarfBuzz shaping), factor de amplificación Parcel, detección de O(n²)
- **Generador de patrones STA:** payloads calibrados para investigación (sin weaponización)
- **Detector de patrones:** análisis de graphemes, codepoints y bidi runs

---

## 📢 Divulgación y Contacto

La investigación se distribuye públicamente con fines de auditoría técnica y resiliencia del software móvil. Su visibilidad respalda activamente los esfuerzos de concienciación sobre salud mental promovidos por **BojosXtu**.

- **Bitácora de investigación:** [lostmon.blogspot.com](https://lostmon.blogspot.com)
- **Iniciativa social:** [BojosXtu en Instagram](https://instagram.com)
- **Contacto:** `bojosxtu@gmail.com` · `lostmon@gmail.com`
- **Twitter:** `@lostmon`

---

## 📚 Índice Completo de Publicaciones

### Vectores individuales
- [STA-001 — Oversized link → context menu](https://lostmon.blogspot.com/2026/08/sta-001-when-oversized-link-reaches.html)
- [STA-003 — Share Intent crash](https://lostmon.blogspot.com/2026/08/sta-003-when-sharing-oversized-link.html)
- [STA-006/007 — Translate frozen UI](https://lostmon.blogspot.com/2026/08/sta-006-007-translate-when-translating.html)
- [STA-009/010/018 — Print Preview Vector](https://lostmon.blogspot.com/2026/09/sta-print-preview-vector.html)
- [STA-012 — Threads persistent crash loop](https://lostmon.blogspot.com/2026/08/sta-012-threads-when-deep-link-becomes.html)
- [STA-031 — The Sixth Entry Point](https://lostmon.blogspot.com/2026/09/libminikin-sixth-entry-point.html)
- [Three Vectors, One Root Cause — STA-003/017/020](https://lostmon.blogspot.com/2026/09/three-vectors-one-root-cause.html)

### Análisis y correlaciones
- [Whitepaper completo — Resilience Gaps in Android IPC, SavedState and Text Layout](https://lostmon.blogspot.com/2026/07/resilience-gaps-in-android-ipc.html)
- [Upstream Convergence — STA Patterns in Android & Chromium (2026)](https://lostmon.blogspot.com/2026/09/upstream-convergence-sta-patterns-in.html)
- [Upstream Android and Chromium changes in 2026](https://lostmon.blogspot.com/2026/08/upstream-android-and-chromium-changes.html)
- [CDN Tsunami and STA — Same amplification pattern](https://lostmon.blogspot.com/2026/08/cdn-tsunami-and-sta-same-amplification.html)
- [STA — UTF-16 Serialization Density Experiment](https://lostmon.blogspot.com/2026/08/sta-utf-16-serialization-density_01007502893.html)
- [libminikin: 10 años de vulnerabilidad en el núcleo de Android](https://lostmon.blogspot.com/2026/07/libminikin-10-anos-de-vulnerabilidad-en_0671440346.html)
- [Algorithmic DoS en libminikin.so](https://lostmon.blogspot.com/2026/06/algorithmic-dos-en-libminikinso.html)
- [Structured Text Amplification (STA) — Marco conceptual](https://lostmon.blogspot.com/2026/06/structured-text-amplification-sta.html)
- [STA: 48 Hours Later — Chrome correlation](https://lostmon.blogspot.com/2026/08/sta-48-hours-later-chrome-correlation.html)

### Reflexiones y contexto personal
- [El dilema de la divulgación coordinada](https://lostmon.blogspot.com/2026/08/el-dilema-de-la-divulgacion-coordinada.html)
- [Desafiar Android desde el sofá y un smartphone](https://lostmon.blogspot.com/2026/07/desafiar-android-desde-el-sofa-y-un.html)
- [Timeline de la investigación STA](https://lostmon.blogspot.com/2026/07/timeline-investigacion-sta-estilos.html)
- [No somos silencio](https://lostmon.blogspot.com/2026/09/no-somos-silencio.html)
- [Mi sofá, mi laboratorio y refugio](https://lostmon.blogspot.com/2026/08/mi-sofa-mi-laboratorio-y-refugio.html)

---

*Copyright © 2026 Manuel García Peña (Lostmon). Todos los derechos reservados.*
