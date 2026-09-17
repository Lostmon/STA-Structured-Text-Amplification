# Mitigaciones para Desarrolladores — Defensa en Profundidad contra el Patrón STA

Las mitigaciones aplicadas en capas superiores, como una aplicación, AndroidX o Chromium, pueden reducir la exposición de una aplicación concreta frente a entradas estructuradas de gran tamaño o elevada complejidad.

Sin embargo, estas medidas deben considerarse **defensa en profundidad**. No constituyen por sí mismas una corrección de una condición subyacente en AOSP, en una biblioteca nativa o en otro componente de plataforma.

El objetivo de este documento es proporcionar medidas defensivas para desarrolladores de Android que procesen texto, `Intent`, `Bundle`, `SavedState`, deep links o contenido potencialmente controlado por terceros.

> **Importante:** los límites indicados a continuación son valores defensivos de ejemplo, no límites de seguridad universales. Deben ajustarse al modelo de amenaza y a los requisitos funcionales de cada aplicación.

---

## Resumen de vectores y mitigaciones

| Vector | Clase | Superficie | Defensa recomendada |
|---|---|---|---|
| STA-001 | A | `ACTION_VIEW` / entrada externa | Validación de URI y límites de entrada |
| STA-003 | A | `ACTION_SEND` / `EXTRA_TEXT` | Validación del payload |
| STA-005 | A | `SavedState` / `Bundle` | Persistir identificadores, no datos voluminosos |
| STA-006/007 | A | IPC / transferencia de datos | Reducir payload y aplicar chunking |
| STA-009/010/018 | A | Printing / datos asociados | Validación previa y procesamiento incremental |
| STA-012 | A | Deep links | Validación de URI y componentes |
| STA-017 | B | Text layout / `libminikin` | Limitar entrada, composición y tamaño de bloques |
| STA-020 | B | `TextView` / UI | Limitar contenido y renderizar por bloques |
| STA-031 | B | Text shaping / renderizado | Chunking + procesamiento fuera del hilo UI |

---

# 1. Defensa contra la Clase A

La Clase A comprende escenarios en los que datos potencialmente voluminosos atraviesan mecanismos de IPC, persistencia de estado u otras interfaces que pueden imponer límites de tamaño o generar costes de serialización.

Una aplicación no debería depender exclusivamente de que Android rechace, trunque o gestione automáticamente un payload excesivo.

La estrategia recomendada es:

1. Validar la entrada lo antes posible.
2. Establecer límites de tamaño apropiados para el caso de uso.
3. Evitar transportar datos grandes mediante `Intent` o `Bundle`.
4. Utilizar almacenamiento persistente para datos voluminosos.
5. Dividir datos grandes en unidades procesables cuando sea necesario.

---

## 1.1 Sanitización de Intents entrantes

Las aplicaciones que reciben enlaces, texto o contenido mediante `Intent` deberían validar explícitamente los datos antes de introducirlos en componentes que puedan realizar parsing, layout, serialización o IPC adicional.

Por ejemplo:

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)

    intent?.let { incomingIntent ->

        when (incomingIntent.action) {

            Intent.ACTION_SEND -> {
                if (incomingIntent.type == "text/plain") {

                    val rawText =
                        incomingIntent.getStringExtra(Intent.EXTRA_TEXT)

                    if (rawText != null && rawText.length > 50_000) {
                        Log.w(
                            "STA_DEFENSE",
                            "Share payload exceeds application limit"
                        )

                        finish()
                        return
                    }
                }
            }

            Intent.ACTION_VIEW -> {

                val uri = incomingIntent.data

                if (uri == null || uri.toString().length > 50_000) {
                    Log.w(
                        "STA_DEFENSE",
                        "URI exceeds application limit"
                    )

                    finish()
                    return
                }
            }
        }
    }
}
```

El valor de `50_000` es solamente un **ejemplo de política defensiva**. No debe interpretarse como un límite universal de Android, Binder, Chromium o de cualquier otro componente.

El límite apropiado depende del caso de uso. Una aplicación que únicamente acepta identificadores cortos puede establecer un límite mucho menor.

### No confundir longitud con seguridad

El tamaño de una cadena no es el único factor relevante.

El coste posterior puede depender también de:

- estructura del contenido;
- composición Unicode;
- caracteres de control;
- operaciones de parsing;
- shaping tipográfico;
- número de objetos generados;
- serialización;
- número de componentes atravesados.

Por ello, una política de longitud debe combinarse, cuando sea apropiado, con validación semántica y de composición.

---

## 1.2 Evitar la persistencia de estados masivos

`onSaveInstanceState()` debe utilizarse para guardar el **estado necesario para reconstruir la interfaz**, no como mecanismo de almacenamiento general de documentos o respuestas completas.

Evita almacenar grandes cadenas o estructuras complejas directamente dentro del `Bundle`.

### ❌ Evitar

```kotlin
override fun onSaveInstanceState(outState: Bundle) {
    super.onSaveInstanceState(outState)

    outState.putString(
        "draft",
        longUserText
    )
}
```

### ✅ Preferible

```kotlin
override fun onSaveInstanceState(outState: Bundle) {
    super.onSaveInstanceState(outState)

    outState.putLong(
        "draft_id",
        currentDraftId
    )
}
```

Los datos voluminosos pueden mantenerse en almacenamiento persistente y el estado de la interfaz puede conservar únicamente un identificador, clave o pequeño conjunto de metadatos necesarios para recuperar dichos datos.

Dependiendo de la arquitectura de la aplicación pueden utilizarse, por ejemplo:

- Room;
- un repositorio persistente;
- almacenamiento de archivos;
- una base de datos local;
- otros mecanismos adecuados al ciclo de vida de la aplicación.

La recuperación posterior debe diseñarse para evitar volver a introducir un payload masivo directamente en el hilo de UI.

---

## 1.3 Validación antes de operaciones de impresión

Las aplicaciones que generan documentos para impresión deberían controlar el tamaño y complejidad del contenido antes de iniciar operaciones que impliquen serialización, IPC o procesamiento adicional.

Por ejemplo:

```kotlin
val document = buildDocument()

if (document.estimatedSize > APPLICATION_PRINT_LIMIT) {
    Toast.makeText(
        this,
        "Documento demasiado grande para imprimir",
        Toast.LENGTH_SHORT
    ).show()

    return
}

val printManager =
    getSystemService(Context.PRINT_SERVICE) as PrintManager

val printAdapter = MyPrintAdapter(document)

printManager.print(
    "job",
    printAdapter,
    PrintAttributes.Builder().build()
)
```

`APPLICATION_PRINT_LIMIT` y `estimatedSize` representan **políticas y lógica de aplicación**, no APIs ni límites proporcionados directamente por Android.

La estimación debe implementarse de acuerdo con el formato real utilizado por la aplicación.

Cuando sea posible, es preferible procesar documentos grandes de forma incremental en lugar de construir un único objeto monolítico.

---

# 2. Defensa contra la Clase B

La Clase B se refiere a situaciones en las que entradas de texto pueden provocar costes elevados durante operaciones de parsing, medición, shaping, line breaking o renderizado.

Una característica importante de estos escenarios es que **la longitud por sí sola no siempre describe adecuadamente el coste computacional**.

Una entrada relativamente corta puede presentar una composición significativamente más compleja que otra de igual longitud.

Por ello, la defensa debe considerar conjuntamente:

- longitud;
- composición;
- procedencia del contenido;
- frecuencia de procesamiento;
- tamaño de los bloques;
- trabajo realizado en el hilo UI.

---

## 2.1 Validación de composición Unicode

Cuando el caso de uso no requiere determinados caracteres de control o mecanismos Unicode complejos, puede ser apropiado filtrarlos.

Por ejemplo:

```kotlin
private fun sanitizeStructuredText(input: String): String {
    return input
        // Controles bidi explícitos
        .replace(
            Regex("[\\u202A-\\u202E\\u2066-\\u2069]"),
            ""
        )

        // Zero Width Joiner, si la aplicación no lo necesita
        .replace("\u200D", "")

        // Limitar secuencias excesivas de combining marks
        .replace(
            Regex("[\\u0300-\\u036F]{3,}"),
            ""
        )

        // Normalizar Braille Pattern Blank cuando no sea necesario
        .replace("\u2800", " ")
}
```

### Precaución

Esta estrategia **no debe aplicarse indiscriminadamente**.

Los caracteres bidi, ZWJ y combining marks tienen usos legítimos en muchos idiomas y sistemas de escritura. Eliminarlos puede alterar el significado o la representación visual del contenido.

Cuando estos caracteres sean necesarios, es preferible aplicar límites de tamaño, chunking y renderizado incremental en lugar de eliminarlos.

---

## 2.2 Restricción de entrada en componentes visuales

Los componentes que reciben texto externo no deberían aceptar cantidades arbitrarias de contenido cuando no existe una necesidad funcional para ello.

Por ejemplo:

```xml
<EditText
    android:id="@+id/my_input"
    android:maxLength="10000"
    android:inputType="textMultiLine" />
```

También puede establecerse un límite mediante `InputFilter`:

```kotlin
val editText = findViewById<EditText>(R.id.my_input)

val staFilter = InputFilter { source, start, end, dest, dstart, dend ->

    val insertedLength = end - start

    val newLength =
        dest.length -
        (dend - dstart) +
        insertedLength

    if (newLength <= 10_000) {
        null
    } else {
        ""
    }
}

editText.filters = arrayOf(staFilter)
```

### Limitación importante

`InputFilter` protege principalmente la ruta de entrada a través de ese `Editable`.

**No constituye una defensa global del proceso.**

Por ejemplo, no protege automáticamente frente a texto introducido mediante:

- `setText()`;
- restauración de estado;
- `Intent`;
- base de datos;
- archivos;
- WebView;
- contenido recibido de red;
- otros componentes de la aplicación.

Por ello, la validación debe realizarse también en los puntos donde el contenido entra en el sistema de procesamiento de la aplicación.

---

## 2.3 Chunking y renderizado incremental

Cuando una aplicación necesita trabajar con documentos extensos, una alternativa más robusta consiste en evitar un único `TextView` que contenga todo el documento.

Una arquitectura defensiva puede utilizar:

1. **Chunking**  
   Dividir el contenido en bloques de tamaño controlado.

2. **Procesamiento fuera del hilo UI**  
   Realizar operaciones costosas en `Dispatchers.Default` u otro ejecutor apropiado.

3. **Viewport rendering**  
   Renderizar únicamente las partes necesarias para la ventana visible.

Por ejemplo:

```kotlin
lifecycleScope.launch(Dispatchers.Default) {

    val params =
        TextViewCompat.getTextMetricsParams(myTextView)

    val precomputedText =
        PrecomputedTextCompat.create(
            chunk,
            params
        )

    withContext(Dispatchers.Main) {
        myTextView.setPrecomputedText(
            precomputedText
        )
    }
}
```

### Papel de `PrecomputedTextCompat`

`PrecomputedTextCompat` puede ayudar a evitar que determinadas operaciones de cálculo de métricas se realicen directamente durante el trabajo del hilo de UI.

Sin embargo:

> **No debe considerarse una corrección de un algoritmo costoso en la plataforma.**

Si una determinada operación nativa presenta un coste elevado para una composición concreta, mover o precalcular parte del trabajo no implica necesariamente que desaparezca ese coste.

Por ello, cuando el contenido puede ser adversarial, **chunking y límites de entrada siguen siendo medidas importantes incluso cuando se utiliza `PrecomputedTextCompat`.**

---

# 3. Principio de defensa en profundidad

Las aplicaciones deberían evitar depender de una única barrera.

Una arquitectura defensiva puede seguir aproximadamente este flujo:

```text
                 ENTRADA EXTERNA
                       │
                       ▼
              ┌──────────────────┐
              │ Validación básica │
              │ tamaño / formato │
              └────────┬─────────┘
                       │
                       ▼
              ┌──────────────────┐
              │ Validación de    │
              │ composición      │
              └────────┬─────────┘
                       │
                       ▼
              ┌──────────────────┐
              │ Chunking /       │
              │ procesamiento    │
              │ incremental      │
              └────────┬─────────┘
                       │
                       ▼
              ┌──────────────────┐
              │ Worker /         │
              │ procesamiento    │
              │ fuera de UI      │
              └────────┬─────────┘
                       │
                       ▼
              ┌──────────────────┐
              │ Renderizado      │
              │ limitado al      │
              │ viewport         │
              └──────────────────┘
```

La idea fundamental es evitar que una entrada externa pase directamente desde una interfaz de entrada hasta una operación potencialmente costosa o un mecanismo IPC sin ningún control intermedio.

---

# 4. Cómo validar las mitigaciones

Las mitigaciones deben probarse en las mismas rutas funcionales en las que la aplicación procesa datos externos.

Un procedimiento básico puede ser:

1. Generar una entrada de prueba representativa del patrón STA.
2. Introducirla mediante la interfaz correspondiente.
3. Verificar que la aplicación aplica el límite previsto.
4. Comprobar que el contenido no alcanza componentes que no deberían recibirlo.
5. Repetir la prueba con diferentes tamaños y composiciones.
6. Capturar información de diagnóstico cuando exista una anomalía.

Por ejemplo:

```bash
adb bugreport
```

Entre las señales que pueden resultar relevantes durante una investigación se encuentran:

| Señal | Interpretación posible |
|---|---|
| `TransactionTooLargeException` | Payload excesivo en una ruta IPC concreta |
| `LineBreakOptimizer::computeBreaks` | Actividad relevante de la ruta de line breaking |
| `InputDispatcher` timeout | El procesamiento del hilo UI excedió el tiempo disponible |
| `BaseBundleMonitorImpl` / Large Bundle | Presencia de un `Bundle` de tamaño elevado |
| Bloqueos prolongados en `onMeasure()` | Coste elevado durante medición/layout |

Estas señales **no deben interpretarse aisladamente como prueba de una vulnerabilidad STA**. Deben correlacionarse con el escenario reproducido, la pila de llamadas, el payload y el comportamiento observado.

---

# 5. Límites de aplicación frente a correcciones de plataforma

Una aplicación puede reducir considerablemente su superficie de exposición mediante límites y procesamiento incremental.

Sin embargo, estas medidas no necesariamente resuelven una condición vulnerable existente en:

- AOSP;
- Android Framework;
- Binder;
- bibliotecas nativas;
- `libminikin`;
- componentes de terceros;
- Chromium;
- otras capas de la plataforma.

Por ejemplo, limitar la entrada de una aplicación concreta mediante `InputFilter` no implica que otros consumidores de Android que procesen el mismo tipo de contenido estén protegidos.

Del mismo modo, una mitigación implementada en Chromium no implica necesariamente que una condición equivalente existente en una biblioteca de plataforma haya sido corregida.

Por esta razón, deben distinguirse tres niveles:

### Nivel 1 — Mitigación de aplicación

Reduce la exposición de una aplicación concreta.

Ejemplos:

- límites de entrada;
- validación de `Intent`;
- chunking;
- viewport rendering;
- evitar `Bundle` voluminosos.

### Nivel 2 — Mitigación de componente

Reduce la exposición de una biblioteca o framework concreto.

Ejemplos:

- límites internos;
- cambios de procesamiento;
- rechazo de entradas;
- algoritmos alternativos.

### Nivel 3 — Corrección de plataforma

Elimina o modifica la condición subyacente en el componente responsable.

Una mitigación de Nivel 1 o Nivel 2 **no debe presentarse como sustituto automático de una corrección de Nivel 3**.

---

# 6. Recomendaciones prácticas

Para aplicaciones que procesan contenido externo:

- Establecer límites explícitos de tamaño.
- Validar `Intent` antes de procesar sus extras.
- Validar deep links antes de pasarlos a componentes de parsing o UI.
- Evitar almacenar documentos completos en `Bundle`/`SavedState`.
- Utilizar identificadores para reconstruir estados grandes.
- Evitar `TextView` monolíticos para documentos extensos.
- Dividir documentos en bloques.
- Utilizar procesamiento fuera del hilo UI cuando corresponda.
- Considerar la composición Unicode además de la longitud.
- No eliminar caracteres Unicode legítimos salvo que el caso de uso lo permita.
- Probar entradas adversariales y no solamente entradas grandes.
- Correlacionar ANR, excepciones y stacks con la ruta exacta de procesamiento.
- Mantener una defensa de aplicación aunque exista una mitigación conocida en una capa inferior.

---

# 7. Referencias

- **Marco teórico STA**
- **Whitepaper v6 — Resilience Gaps in Android IPC, SavedState and Text Layout**
- **Upstream Convergence — STA Patterns in Android & Chromium (2026)**
- Documentación oficial de Android sobre `Intent`, `Bundle`, `SavedState`, `TextView`, `InputFilter`, `PrecomputedText` y arquitectura de aplicaciones.

---

# 8. Disclaimer

Estas mitigaciones son recomendaciones independientes del investigador y constituyen medidas de **defensa en profundidad**.

No representan necesariamente la postura oficial de Google, Android, Chromium, Microsoft ni de ningún otro vendor.

Los límites numéricos utilizados en los ejemplos son valores de política de aplicación y **no deben interpretarse como límites de seguridad universales de Android**.

Los desarrolladores deben adaptar las medidas a la funcionalidad de su aplicación, sus requisitos de compatibilidad y su modelo de amenaza.

La aplicación de estas medidas tampoco constituye, por sí sola, evidencia de que una condición STA haya sido eliminada de la plataforma subyacente.

---

Copyright © 2026 Manuel García Peña (Lostmon). Todos los derechos reservados.
