# 🛡️ Android STA (Structured Text Amplification) Vulnerability Index

> **Investigación técnica independiente desarrollada por Manuel García Peña ([@Lostmon](https://github.com))**  
> *Un marco conceptual sobre la denegación de servicio algorítmica, fallos de resiliencia en canales IPC Binder, persistencia en SavedState e inestabilidad en el renderizado nativo de Android (libminikin).*

---

## 📢 Filosofía del Proyecto y Compromiso Social
Esta investigación documenta fallos de código y demuestra resiliencia técnica y personal, respaldando los esfuerzos de salud mental de **BojosXtu**.
*   **Bitácora oficial:** [lostmon.blogspot.com](https://lostmon.blogspot.com)
*   **Iniciativa Social:** [BojosXtu en Instagram](https://www.instagram.com/p/DaxP36ntx58/)
*   **Contacto:** bojosxtu@gmail.com · lostmon@gmail.com

---

## 🔬 El Patrón Estructural STA: Marco Conceptual
El patrón **STA (Structured Text Amplification)** define un modelo de explotación basado en la **asimetría de costes computacionales**, revelando una debilidad arquitectónica de más de una década en Android por la falta de validaciones tempranas de longitud y densidad antes de cruzar fronteras críticas del sistema.

---

## 🗂️ Índice de Vectores de Vulnerabilidad
La investigación categoriza **31 vectores activos** en dos clases:

*   **Clase A (Binder / SavedState / IPC):** Transacciones que superan el límite de 1 MB provocando `TransactionTooLargeException` (ej. STA-003, STA-005, STA-006/007, STA-009/010/018, STA-012, STA-001).
*   **Clase B (libminikin / Renderizado):** Complejidad \(O(n^2)\) en componentes nativos de bajo nivel que generan ANRs críticos (ej. STA-017, STA-020, STA-031).

---

## 🛠️ Anatomía del Bloqueo y Convergencia Upstream
Los volcados de memoria confirman bloqueos en el *UI Thread* superando los 5 segundos en funciones nativas como `libminikin.so`. Se observa una convergencia en AOSP, AndroidX y Chromium con parches orientados a limitar payloads pesados y transacciones mediante alternativas seguras. El detalle completo del whitepaper, stack traces y publicaciones clave se encuentra disponible en la bitácora del autor.

---
*Copyright © 2026 Manuel García Peña (Lostmon). Todos los derechos reservados.*
