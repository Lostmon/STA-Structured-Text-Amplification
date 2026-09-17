```markdown
# 🛡️ Marco Teórico — Structured Text Amplification (STA)

> **Investigación técnica independiente desarrollada por Manuel García Peña ([@Lostmon](https://github.com/Lostmon))**
> *Un modelo conceptual sobre denegación de servicio algorítmica, fallos de resiliencia en canales IPC Binder, persistencia en SavedState e inestabilidad en el renderizado nativo de Android.*

---

## 📢 Filosofía del Proyecto y Compromiso Social

Esta investigación documenta fallos arquitectónicos mientras demuestra resiliencia técnica y personal. Todo el trabajo se realiza desde un entorno doméstico, con un teléfono de gama media, sin laboratorio ni acceso interno a código fuente. Su visibilidad respalda los esfuerzos de concienciación en salud mental promovidos por **BojosXtu**.

- **Bitácora oficial:** [lostmon.blogspot.com](https://lostmon.blogspot.com)
- **Iniciativa social:** [BojosXtu en Instagram](https://instagram.com/bojosxtu)
- **Contacto:** bojosxtu@gmail.com · lostmon@gmail.com

---

## 🔬 El Patrón Estructural STA: Definición Formal

> *Structured Text Amplification (STA) es un patrón de amplificación de recursos arquitectónico en el que una entrada textual estructurada atraviesa múltiples capas software, causando un incremento progresivo del coste computacional, del consumo de memoria o de la propagación de estado, hasta superar los límites de estabilidad de uno o más componentes downstream.*

**Pipeline general:**

```

Entrada estructurada (URL / HTML / Intent / Clipboard / Deep Link)
↓
Parser / Validación (ausente o insuficiente)
↓
Framework APIs
↓
Binder IPC (límite duro 1 MB)
↓
Rendering Engine (TextView → StaticLayout → LineBreaker → libminikin) ← Clase B
↓
Layout / Compose
↓
State Persistence (TaskPersister / SavedStateRegistry) ← Clase A
↓
SystemUI / recuperación OEM
↓
Fallo observado (ANR / Crash Loop / Hard Reboot)

```

### 1.1 Propiedades recurrentes

| Propiedad | Descripción |
|-----------|-------------|
| **Entrada estructurada** | Información textual jerárquica o codificada (URLs, HTML, JSON, Intent extras, Markdown), no binaria arbitraria. |
| **Procesamiento multi-etapa** | El mismo contenido lógico atraviesa múltiples capas independientes del framework. |
| **Amplificación progresiva** | El coste de procesamiento crece en cada etapa sucesiva. |
| **Infraestructura compartida** | Apps sin relación reutilizan los mismos componentes de Android (libminikin, FragmentManager, TaskPersister, SystemUI). |
| **Comportamiento no lineal** | El consumo de recursos es desproporcionado respecto al tamaño de la entrada. |
| **Impacto cross-componente** | Los fallos aparecen lejos del punto de entrada original (ej. click en browser → crash de SystemUI). |

---

## 🧩 Clasificación: Clase A y Clase B

### Clase A — IPC / SavedState Amplification

**Mecanismo:** Serialización de estado → Bundle → Parcel → Binder (>1 MB).

**Recurso agotado:** Buffer de transacción Binder (1.048.576 bytes).

**Factor de amplificación:** Depende de la profundidad de anidamiento de Fragment y de la metadata asociada. Observado entre ×2.8 y ×20.6 en mediciones de campo.

**Resultado:** `TransactionTooLargeException` no capturada → crash loop persistente.

**Vectores representativos:** STA-001, STA-003, STA-005, STA-006/007, STA-009/010/018, STA-012.

### Clase B — Interaction Surface Amplification

**Mecanismo:** Layout de texto síncrono en el hilo UI (libminikin line breaking + glyph shaping).

**Recurso agotado:** Tiempo de CPU del hilo principal (umbral ANR ≈5 s).

**Factor de amplificación:** Multiplicador 10–20× antes de entrar en O(n²) o bloqueo greedy. La densidad de caracteres especiales (# / % €, U+2800) dispara el número de candidatos de line-break.

**Resultado:** Bloqueo del hilo principal → ANR (5–16 s).

**Vectores representativos:** STA-017, STA-020, STA-031.

**Nota crítica:** Ambas clases comparten la misma categoría de causa raíz (ausencia de validación de longitud antes de operaciones costosas), pero **no son reducibles a un único modelo matemático**. El fix de una no aborda la otra.

---

## 🗂️ Índice de Vectores Documentados

La investigación cataloga **32 vectores reproducibles** en dos clases:

- **Clase A (Binder / SavedState / IPC):** Transacciones que superan el límite de 1 MB provocando `TransactionTooLargeException`.
- **Clase B (libminikin / Renderizado):** Complejidad algorítmica en componentes nativos que genera ANRs críticos.

El catálogo completo, con CVSS 3.1, nivel de evidencia (Tier A/B) y persistencia, está en el [README principal](./README.md).

---

## 🛠️ Anatomía del Bloqueo y Evidencia Forense

Los volcados de producción confirman bloqueos en el **hilo UI** superando el umbral de 5 segundos del watchdog de Android, con el hilo principal ejecutando código nativo en `libminikin.so`:

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

**BuildId de `libminikin.so`:** `4fabe53671b5ead88314c00a1fd6d67d`

El mismo BuildId aparece en múltiples apps (Chrome, Edge, Firefox, Google Docs), lo que confirma que el problema reside en el componente de sistema, no en la implementación de cada aplicación.

Los stack traces completos, Binder diagnostics y capturas forenses están documentados en la bitácora del autor.

---

## 📊 Convergencia Upstream (2026)

Durante 2026 se ha observado una **convergencia técnica** en AOSP, AndroidX y Chromium: múltiples proyectos han introducido mitigaciones paralelas orientadas a limitar payloads pesados, redirigir transacciones y observar fallos de frontera en Binder.

Patrón consistente: **constrain → redirect → replace → observe**.

Sin embargo, **`libminikin` permanece como la única superficie investigada sin barrera global de longitud** en `LineBreakOptimizer::computeBreaks()`.

Análisis completo: [Upstream Convergence — STA Patterns in Android & Chromium (2026)](https://lostmon.blogspot.com/2026/09/upstream-convergence-sta-patterns-in.html)

---

## 📚 Documentación Complementaria

- [README principal](./README.md) — Índice de vectores y publicación completa
- [Whitepaper v6 — Resilience Gaps in Android IPC, SavedState and Text Layout](https://lostmon.blogspot.com/2026/07/resilience-gaps-in-android-ipc.html)
- [Upstream Convergence Analysis](https://lostmon.blogspot.com/2026/09/upstream-convergence-sta-patterns-in.html)
- [Three Vectors, One Root Cause](https://lostmon.blogspot.com/2026/09/three-vectors-one-root-cause.html)
- [The Sixth Entry Point — HarfBuzz Shaping](https://lostmon.blogspot.com/2026/09/libminikin-sixth-entry-point.html)

---

## 📢 Divulgación y Uso Responsable

Este repositorio **no contiene payloads weaponizados**. La documentación se distribuye con fines de auditoría técnica, resiliencia de software y análisis de framework.

Reproduction steps están incluidos para que vendors y equipos de seguridad puedan validar y mitigar los patrones documentados.

Si trabajas en seguridad móvil, análisis de framework o resiliencia de software, este marco está pensado para que puedas contribuir a cerrar la brecha arquitectónica.

---

*Copyright © 2026 Manuel García Peña (Lostmon). Todos los derechos reservados.*
```
