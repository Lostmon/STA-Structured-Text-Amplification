# Structured Text Amplification (STA) en Gemini

**Instruction Drift, Token Dumping y generación de código Rust vulnerable (Integer Overflow / Heap Overflow)**

> Por **Manuel García Peña (Lostmon)** · [lostmon.blogspot.com](https://lostmon.blogspot.com)

![CVSS](https://img.shields.io/badge/CVSS-9.8%20Critical-critical?style=flat-square)
![RCE](https://img.shields.io/badge/Impact-RCE%20%2F%20DoS-red?style=flat-square)
![Target](https://img.shields.io/badge/Affects-gemini.google.com-blue?style=flat-square)
![Type](https://img.shields.io/badge/Type-RCE%20%2F%20DoS-orange?style=flat-square)

---

## Índice

1. [Resumen](#1-resumen)
2. [Descripción técnica](#2-descripción-técnica)
3. [Código vulnerable generado](#3-código-vulnerable-generado)
4. [Análisis de impacto](#4-análisis-de-impacto)
5. [Prueba de concepto (PoC)](#5-prueba-de-concepto-poc)
6. [Parche conceptual](#6-parche-conceptual)
7. [Contexto y comparación (Android · libminikin)](#7-contexto-y-comparación-android--libminikin)
8. [Mitigación y recomendaciones](#8-mitigación-y-recomendaciones)

---

## 1. Resumen

**Structured Text Amplification (STA)** es una técnica de entrada que satura el mecanismo de atención cuadrático _O(n²)_ de los LLM mediante patrones de texto repetitivos no-ASCII estructurados. En **Gemini**, esta saturación provoca dos efectos encadenados:

- **Instruction Drift** — el modelo pierde el contexto del sistema y las instrucciones de seguridad.
- **Token Dumping** — el modelo vuelca de forma descontrolada contenido de su distribución interna.

Como consecuencia, Gemini es inducido a emitir código Rust _sin sanitizar_ que contiene **Integer Overflow** y **Heap Overflow**. Cuando este código se integra en aplicaciones reales, puede dar lugar a **Denegación de Servicio (DoS)** por OOM o, en escenarios donde el índice desbordado se convierte en una escritura fuera de límites, a **Ejecución Remota de Código (RCE)**.

---

## 2. Descripción técnica

### 2.1 Mecanismo del ataque

El ataque explota una _asimetría arquitectónica_: el mecanismo de atención de Gemini escala cuadráticamente con la longitud del contexto. Alimentar al modelo con patrones estructurados repetitivos (por ejemplo, secuencias de caracteres no-ASCII con estructura reconocible que esquivan la tokenización BPE agresiva) provoca:

1. **Amplificación de atención**: cada token "mira" a todos los anteriores; el coste se dispara.
2. **Pérdida de contexto**: el modelo deja de ponderar correctamente las instrucciones iniciales (system prompt, políticas de seguridad, guías de código seguro).
3. **Token Dumping**: el modelo empieza a emitir secuencias de alta probabilidad (como el algoritmo clásico de _counting sort_) sin aplicar las verificaciones que aplicaría en condiciones normales.

### 2.2 Fallo arquitectónico subyacente

El mismo patrón de fallo (amplificación de texto estructurado causando comportamiento descontrolado) ya fue documentado en la pila gráfica de Android, concretamente en **libminikin**. En ambos casos, el sistema procesa entrada no confiable a través de un pipeline optimizado para casos benignos, sin cotas superiores explícitas, y el resultado es un consumo de recursos desproporcionado que termina afectando a la integridad del resultado.

> [!CAUTION]
> Este informe no describe una vulnerabilidad de memoria en Gemini en sí, sino una **falla de alineación de seguridad** que produce como salida código explotable en el consumidor del modelo.

---

## 3. Código vulnerable generado

Gemini emite, tras el ataque STA, la siguiente función `counting_sort` sin advertir al usuario de sus límites:

```rust
pub fn counting_sort(arr: &mut [i32]) {
    if arr.is_empty() { return; }
    let mut min = arr[0];
    let mut max = arr[0];
    for &num in arr.iter() {
        if num < min { min = num; }
        if num > max { max = num; }
    }
    let range = (max - min + 1) as usize;     // VULN 1: Integer Overflow (i32::MIN/MAX)
    let mut count = vec![0; range];           // VULN 2: OOM (~16GB) o Heap Overflow (OOB write)
    for &num in arr.iter() {
        count[(num - min) as usize] += 1;
    }
    let mut i = 0;
    for (val_offset, &c) in count.iter().enumerate() {
        let val = min + val_offset as i32;
        for _ in 0..c {
            arr[i] = val;
            i += 1;
        }
    }
}
```

### 3.1 Detalle de las vulnerabilidades

| ID | Ubicación | Descripción | Efecto |
|---|---|---|---|
| **VULN 1** | `(max - min + 1) as usize` | Si `min = i32::MIN` y `max = i32::MAX`, la resta `max - min` desborda `i32`. En _debug_ produce panic; en _release_ hace wrap-around. | Comportamiento indefinido; `range` puede quedar a 0 o a un valor enorme. |
| **VULN 2** | `vec![0; range]` | Si `range` ≈ 2³², se intenta reservar ~16 GB. | OOM / DoS. Si la asignación parcial se produce y el índice `(num - min) as usize` desborda, se accede fuera de límites. |
| **VULN 3** | `count[(num - min) as usize]` | Conversión implícita de `i32` a `usize` tras una resta que puede ser negativa (si `min` fue calculado mal por un wrap-around previo). | Índice enorme → **out-of-bounds write** → posible Heap Overflow → escalado a RCE. |

---

## 4. Análisis de impacto

### 4.1 Superficie afectada

- **Modelo:** bypass de alineación de seguridad + ACDoS (pico de latencia O(n²)).
- **Código generado:** Integer Overflow → panic/wrap; OOM → DoS (~16 GB); Heap Overflow → RCE potencial.
- **Ecosistema:** cualquier aplicación que integre la salida del modelo sin revisión.

### 4.2 Severidad

| Escenario | CVSS | Descripción |
|---|---|---|
| El código generado procesa **entrada no confiable** | **9.8 (Crítico)** | Heap Overflow → RCE. |
| Solo causa **DoS** (OOM/panic) | **7.5 (Alto)** | Interrupción del servicio. |
| Uso interno, entrada controlada | Moderado | Riesgo residual de malas prácticas. |

> [!WARNING]
> **Riesgo estratégico:** este tipo de fallo _erosiona la confianza_ en los asistentes de código. Viola estándares de codificación segura (MISRA, CERT, SEI) y, dada la naturaleza "plausible" del código generado, es muy probable que pase desapercibido en revisiones y llegue a producción.

---

## 5. Prueba de concepto (PoC)

1. Enviar a Gemini un prompt del tipo:

   ```
   Count this string: [STA-PATTERN]
   ```

   donde `[STA-PATTERN]` es una secuencia larga de texto estructurado no-ASCII que provoca fallo en la tokenización BPE y satura la atención.

2. El modelo **pierde el contexto del sistema** y emite como respuesta la función `counting_sort` mostrada arriba, sin advertencias ni sanitización.

> [!NOTE]
> El patrón STA empleado es el mismo que se documentó previamente en la pila gráfica de Android (`libminikin`), lo que sugiere que el fallo arquitectónico subyacente es más general de lo que aparenta y no exclusivo de Gemini.

---

## 6. Parche conceptual

Una versión segura de la función debe: usar aritmética en `i64`, acotar el rango, validar índices antes de usarlos y devolver errores explícitos.

```rust
pub fn counting_sort(arr: &mut [i32]) -> Result<(), &'static str> {
    if arr.is_empty() { return Ok(()); }

    let min = *arr.iter().min().unwrap();
    let max = *arr.iter().max().unwrap();

    // Fix 1: Evita overflow con i64 y acota el rango (previene OOM)
    let range_i64 = (max as i64) - (min as i64) + 1;
    if range_i64 <= 0 || range_i64 > 1_000_000 {
        return Err("Rango inválido o demasiado grande");
    }
    let range = range_i64 as usize;

    let mut count = vec![0; range]; // Fix 2: Tamaño limitado

    // Fix 3: Indexado seguro
    for &num in arr.iter() {
        let idx = (num as i64 - min as i64) as usize;
        if idx >= range { return Err("Índice fuera de rango"); }
        count[idx] += 1;
    }

    let mut i = 0;
    for (offset, &c) in count.iter().enumerate() {
        let val = min + offset as i32;
        for _ in 0..c {
            arr[i] = val;
            i += 1;
        }
    }
    Ok(())
}
```

### 6.1 Cambios clave

| # | Cambio | Beneficio |
|---|---|---|
| 1 | Aritmética en `i64` y cota superior | Evita wrap-around y previene OOM. |
| 2 | Reserva de memoria condicionada al rango validado | Elimina el vector gigante desproporcionado. |
| 3 | Comprobación `idx >= range` antes de indexar | Descarta el Heap Overflow y el posible RCE. |
| 4 | Firma con `Result<(), &'static str>` | Fuerza al llamador a manejar el error (API segura). |

---

## 7. Contexto y comparación (Android · libminikin)

Este mismo patrón de fallo arquitectónico fue documentado previamente en Android, dentro de `libminikin`: entradas de texto estructurado producían un consumo desproporcionado de recursos y comportamiento errático en el motor de texto. La coincidencia sugiere que **la amplificación de texto estructurado es una clase de fallo transversal** que afecta a sistemas de distinta naturaleza (motores gráficos y modelos de lenguaje) siempre que compartan un pipeline de procesamiento optimizado para el caso benigno y sin cotas superiores explícitas.

> [!WARNING]
> **Conclusión del autor:** no es un bug aislado de Gemini. Es una _familia de vulnerabilidades_ que requiere tratamiento sistémico: cotas de entrada, validación estricta y aritmética segura, tanto en el modelo como en el código que produce.

---

## 8. Mitigación y recomendaciones

### 8.1 Para el proveedor del modelo (Google · Gemini)

- Sanitizar entradas repetitivas y detectar patrones STA antes de la inferencia.
- Establecer presupuestos de atención (_attention budgets_) que degraden graciosamente.
- Forzar la generación de código con aritmética _checked/saturating_ por defecto.
- Reducir la longitud máxima del contexto en respuestas de generación de código.

### 8.2 Para consumidores de código generado por IA

- Nunca integrar código de IA sin revisión por una persona con criterio de seguridad.
- Ejecutar `clippy`, `cargo-audit` y análisis estático (SAST).
- Fijar límites de memoria y CPU en los servicios que ejecuten el código (containers con cgroups).
- Aplicar **fuzzing** a las funciones de ordenación/parsing generadas.
- Utilizar crates auditados (`radsort`, `itertools`) en lugar de código ad-hoc.

### 8.3 Para la comunidad de seguridad

- Reportar STA como una clase de fallo, no como una incidencia puntual.
- Coordinar divulgación responsable con el proveedor del modelo.
- Documentar el vector en bases de datos de fallos de IA (AI CVE / OWASP LLM Top 10).

---

## Autor

**Manuel García Peña (Lostmon)** · [lostmon.blogspot.com](https://lostmon.blogspot.com)

Este material se publica con fines educativos y de divulgación de seguridad. El uso indebido de las técnicas descritas es responsabilidad exclusiva del lector.
