# Mitigaciones para Desarrolladores — Defensa en Profundidad contra el Patrón STA

Las mitigaciones aplicadas en capas superiores, como una aplicación, AndroidX o Chromium, pueden reducir la exposición de una aplicación concreta frente a entradas estructuradas de gran tamaño o elevada complejidad.

Sin embargo, estas medidas deben considerarse **defensa en profundidad**. No constituyen por sí mismas una corrección de una condición subyacente en AOSP, en una biblioteca nativa o en otro componente de plataforma.

El objetivo de este documento es proporcionar medidas defensivas para desarrolladores de Android que procesen texto, `Intent`, `Bundle`, `SavedState`, deep links o contenido potencialmente controlado por terceros.

> **Importante:** los límites indicados a continuación son valores defensivos de ejemplo, no límites de seguridad universales. Deben ajustarse al modelo de amenaza y a los requisitos funcionales de cada aplicación.

> **Nota de versión (v2):** esta versión añade las secciones 9–13, derivadas de una cadena de reproducción específica (STA Class A sobre `Task`/`TaskInfo`/ATMS, documentada por separado) que involucró navegadores, un gestor de deep links de mensajería, y el propio subsistema `RescueParty` de la plataforma. Las secciones 1–8 permanecen sin cambios respecto a la versión anterior.

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
| *(nuevo)* | A | `TaskInfo` / ATMS / `TaskOrganizer` | Cota de longitud en `fillTaskInfo()`/`writeTaskToParcel()` — ver §13 |
| *(nuevo)* | A | Deep link target / `ClientLifecycleManager` | Degradación resiliente del ciclo de vida — ver §9 |
| *(nuevo)* | A | `ContentProvider` compartido | Higiene de permisos de URI — ver §10 |
| *(nuevo)* | A | Buffer de binder compartido | Patrón circuit-breaker, no solo try/catch puntual — ver §11 |

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
| `DeadObjectException` / `DeadSystemException` | Canal binder hacia el destinatario inutilizable (no implica necesariamente que el proceso remoto haya terminado — ver §11) |
| `BadParcelableException: ... only received N of M` | Deserialización incompleta de una lista paginada (`ParceledListSlice`), compatible con agotamiento del buffer, no solo con un payload individual sobredimensionado |
| `error -28 (No space left on device)` en logs de binder | Buffer de transacciones binder saturado — puede afectar también a llamadas pequeñas posteriores en el mismo canal |

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

# 9. Defensa para aplicaciones que actúan como destino de Deep Link

Una reproducción específica de la cadena STA Class A (`Task`/`TaskInfo`/ATMS, documentada por separado) mostró que una aplicación de mensajería, al ser el **destino** de un deep link (`esquema://acción/...`, un enlace universal `https://.../...`, o un `content://` servido por otra app), puede quedar expuesta a un fallo en la entrega de su propia transacción de ciclo de vida:

```
ClientLifecycleManager: android.os.TransactionTooLargeException: data parcel size ~558000 bytes
    at android.app.IApplicationThread$Stub$Proxy.scheduleTransaction(...)
Caused by: android.os.DeadObjectException
```

El resultado observado fue la terminación del proceso de la aplicación objetivo y, tras un reintento fallido, el abandono del lanzamiento por parte de Android (`"Second failure launching <app>, giving up"`) — es decir, **la aplicación queda inalcanzable vía ese deep link hasta que algo en el estado subyacente cambie**, sin ningún mensaje de error útil para el usuario.

Esto no es una excepción que la aplicación objetivo pueda capturar con un `try/catch` convencional — ocurre en la entrega de la transacción por parte del framework, antes de que el código de la aplicación tenga oportunidad de ejecutarse. Sin embargo, hay medidas defensivas razonables:

## 9.1 No asumir que `onResume()`/`onStop()` son operaciones libres de fallo

En la misma cadena de reproducción, un navegador crasheó en su propio `onResume()`/`onStop()` al consultar el estado de su tarea:

```kotlin
// Patrón observado como causa de crash — NO hacer esto sin protección:
override fun onResume() {
    super.onResume()
    val info = activityManager.appTasks[0].taskInfo  // puede lanzar DeadSystemException
}
```

### ✅ Defensa recomendada

```kotlin
override fun onResume() {
    super.onResume()

    val info = try {
        activityManager.appTasks.firstOrNull()?.taskInfo
    } catch (e: DeadObjectException) {
        Log.w("STA_DEFENSE", "ATMS binder channel unavailable, using cached state", e)
        null
    } catch (e: RuntimeException) {
        // DeadSystemException es una subclase no siempre捕 accesible
        // directamente según la versión de API; capturar RuntimeException
        // de forma defensiva en este punto concreto es razonable.
        Log.w("STA_DEFENSE", "Unexpected ATMS failure, degrading gracefully", e)
        null
    }

    if (info == null) {
        // Degradar: usar el último estado conocido en memoria,
        // o simplemente omitir la actualización de UI dependiente
        // de este dato, en vez de propagar la excepción.
        applyLastKnownTaskState()
    } else {
        applyTaskState(info)
    }
}
```

Cualquier llamada a `ActivityManager`/`ActivityTaskManager` desde un método de ciclo de vida que el sistema invoca automáticamente (`onResume`, `onStop`, `onPause`, callbacks de `TaskStackListener`, etc.) debería tratarse como una operación que puede fallar por causas ajenas a la propia aplicación, y degradar en vez de propagar.

## 9.2 No adjuntar payloads voluminosos a los Intents de deep link propios

Si la aplicación construye sus propios Intents de deep link (por ejemplo, al generar un enlace para compartir, o al reenviar un `ACTION_VIEW` internamente tras resolver una ruta), aplican las mismas reglas de §1.1: validar longitud y composición antes de adjuntar datos al `Intent`, y evitar extras voluminosos en el `Bundle` del propio Intent — ese estado puede terminar persistido en el `Task` del sistema y re-serializado repetidamente por componentes que la aplicación no controla (SystemUI, `TaskOrganizer`, `RecentTasks`), no solo leído una vez por el receptor directo.

## 9.3 Diseñar el manejador de deep link para fallar de forma visible al usuario

Si el framework abandona el lanzamiento tras un fallo de entrega, el usuario no ve ningún mensaje — el enlace simplemente "no hace nada". Cuando sea posible, una aplicación puede registrar telemetría propia (fuera de la ruta de IPC fallida, p. ej. en el siguiente arranque exitoso) para detectar patrones de lanzamientos abandonados y, si corresponde, informar al usuario o reintentar con una ruta alternativa más ligera (por ejemplo, abrir la pantalla principal en vez de la pantalla profunda específica).

---

# 10. Higiene de permisos en `ContentProvider` compartidos

En la misma investigación se observó que un `ContentProvider` de una aplicación de mensajería, usado para compartir un documento HTML con un navegador, concedía acceso de lectura en un flujo (compartir directo) y lo denegaba con `SecurityException: Permission Denial ... not exported` en otro flujo aparentemente equivalente (a través de un chooser del sistema).

Independientemente de la causa exacta en ese caso concreto, esto ilustra un patrón de higiene de permisos que conviene revisar en cualquier `ContentProvider` que exponga contenido a otras aplicaciones:

- Preferir concesiones de permiso **de URI específica y de duración acotada** (`Context.grantUriPermission()` con un URI concreto, o `FLAG_GRANT_READ_URI_PERMISSION` en el propio `Intent` de compartición) frente a exportar el provider de forma amplia.
- Revocar el permiso explícitamente cuando ya no sea necesario (`Context.revokeUriPermission()`), en vez de depender únicamente de que el proceso consumidor termine.
- Si el mismo contenido puede alcanzar al consumidor por **más de una ruta** (compartición directa, chooser del sistema, deep link reenviado), verificar que el comportamiento de concesión de permisos sea el mismo en todas ellas — una inconsistencia entre rutas puede ser tanto un problema de seguridad (acceso no previsto) como de disponibilidad (acceso legítimo denegado de forma intermitente).
- Registrar (en logs propios de diagnóstico, no necesariamente visibles al usuario) los casos en los que una app externa intenta acceder sin permiso, para poder distinguir un intento malicioso de una inconsistencia propia en la ruta de concesión.

---

# 11. Resiliencia frente a saturación del buffer de binder compartido

Una reproducción adicional de la misma cadena mostró evidencia directa de que el fallo no siempre se limita a una transacción individual demasiado grande: tras una transacción de aproximadamente 1,1 MB, una llamada completamente normal de 124 bytes, en el mismo canal de binder, también falló con el mismo error de kernel (`-28`, sin espacio en el buffer de transacciones).

Esto tiene una implicación práctica para el diseño de aplicaciones que dependen de llamadas IPC frecuentes hacia componentes del sistema (ATMS, `TaskOrganizer`, gestores de ciclo de vida, etc.):

> **Un `try/catch` alrededor de una única llamada no es suficiente si el canal subyacente puede quedar saturado por actividad previa ajena a esa llamada concreta.**

### 11.1 Patrón recomendado: circuit breaker simple

En vez de reintentar indefinidamente cada llamada fallida de forma aislada, es preferible un patrón que detecte una ráfaga de fallos en el mismo canal y deje de intentar temporalmente, degradando con gracia:

```kotlin
class AtmsCallCircuitBreaker(
    private val failureThreshold: Int = 3,
    private val cooldownMs: Long = 10_000
) {
    private var consecutiveFailures = 0
    private var openUntil: Long = 0

    fun <T> callOrNull(block: () -> T): T? {
        val now = SystemClock.elapsedRealtime()

        if (now < openUntil) {
            // Circuito abierto: no reintentar todavía, degradar directamente
            return null
        }

        return try {
            val result = block()
            consecutiveFailures = 0
            result
        } catch (e: RuntimeException) {
            consecutiveFailures++
            if (consecutiveFailures >= failureThreshold) {
                openUntil = now + cooldownMs
                Log.w(
                    "STA_DEFENSE",
                    "Circuit opened after $consecutiveFailures consecutive IPC failures"
                )
            }
            null
        }
    }
}
```

Este patrón no corrige la causa raíz (que es responsabilidad de la plataforma — ver §13), pero evita que una aplicación entre en un bucle de reintentos que añada más presión a un canal ya saturado, lo cual es exactamente el mecanismo que, en la cadena de reproducción observada, prolongó y agravó el fallo en vez de permitir su recuperación.

---

# 12. No depender de la auto-recuperación de la plataforma

La misma cadena de reproducción mostró que el mecanismo de auto-recuperación de la plataforma (`RescueParty` en AOSP, extendido por algunos fabricantes) puede escalar, cuando sus mitigaciones de menor coste no resuelven la condición, hasta un **reinicio completo involuntario del dispositivo** (`PackageWatchdogImpl.executeWarmReboot()` → `PowerManager.reboot()`), sin que medie ninguna acción del usuario.

Para el desarrollador de aplicaciones, esto tiene una implicación directa: **una aplicación no debería asumir que, si su componente de UI entra en un bucle de fallos, la plataforma lo resolverá de forma transparente**. Las mitigaciones de plataforma existen precisamente porque el caso no se resolvió por sí solo a tiempo, y su efecto visible para el usuario (modo seguro, reinicio) es disruptivo.

Recomendaciones:

- Si la aplicación mantiene procesos o componentes de larga duración (servicios en primer plano, componentes persistentes), implementar sus propios límites de reintento y *fallback* a un estado mínimo funcional, en vez de confiar en que el sistema los reinicie de forma indefinida sin consecuencias visibles.
- Evitar diseñar flujos que dependan de que un componente del sistema (SystemUI, un gestor de tareas recientes, etc.) esté siempre disponible de forma síncrona; tratar su ausencia temporal como un caso esperado, no excepcional.
- Si la aplicación detecta patrones de reinicio inusualmente frecuentes del propio proceso (por ejemplo, comparando el *uptime* del proceso con el histórico), registrar esa señal para diagnóstico propio — puede ser la única forma de que el desarrollador se entere de que su aplicación está contribuyendo a una condición de este tipo en el dispositivo de un usuario.

---

# 13. Referencia de nivel de plataforma: cota de longitud en `Task.fillTaskInfo()`/`TaskInfo.writeTaskToParcel()`

Esta sección resume, a título de referencia técnica, una propuesta de parche conceptual a nivel de plataforma (AOSP) que aborda directamente el campo de datos no acotado identificado en la investigación (`Intent.data` del `baseIntent` tras `cloneFilter()`, y el campo `TaskInfo.capturedLink`). No es una mitigación que un desarrollador de aplicaciones de terceros pueda aplicar — se incluye aquí porque **complementa** las mitigaciones de aplicación anteriores: estas últimas reducen la exposición individual, pero la condición subyacente solo se cierra con un cambio como el descrito a continuación.

## 13.1 Qué corrige

- Acota la longitud serializada del data URI del `baseIntent` (que `cloneFilter()` preserva íntegro) y del campo `capturedLink` (que no pasa por el recorte aplicado a callers no privilegiados).
- Aplica el límite tanto en el punto de producción (`Task.fillTaskInfo()`) como, como defensa adicional, en el punto de serialización (`TaskInfo.writeTaskToParcel()`).
- Preserva `scheme` y `authority` del URI truncado en vez de anularlo por completo, para no perder utilidad de diagnóstico, y añade parámros de consulta informativos (`__truncated__`, `__original_length__`) para telemetría.
- No muta el objeto `Task` en memoria — solo la copia serializada hacia el cliente.

## 13.2 Correcciones aplicadas a la propuesta original antes de incluirla aquí

Una revisión técnica de la propuesta original identificó tres problemas que se corrigen en esta versión de referencia:

1. **Pérdida de `flags` del Intent.** La propuesta original sustituía el bloque de `fillTaskInfo()` por una llamada a `cloneFilter()` sin restablecer `flags` después. `cloneFilter()` **no copia `flags`** (documentado así en AOSP — clona únicamente `action`, `data`, `type`, `component` y `categories`). El código original SÍ restablecía `flags` manualmente tras clonar; la cota de longitud debe preservar esa misma llamada, no eliminarla.
2. **Rama `intent == null` no cubierta.** El código original siempre asignaba `info.baseIntent` (a un `Intent` vacío si no había uno). La propuesta solo asignaba dentro de un condicional `if (intent != null)`, dejando `info.baseIntent` sin tocar en el caso contrario.
3. **Error aritmético en la justificación del umbral.** La propuesta original justificaba 64 KB como "un cuarto del buffer de transacción de ~1 MB" — 1 MB / 4 son 256 KB, no 64 KB. El umbral en sí puede seguir siendo 64 KB si así se decide, pero la justificación debe corregirse o el umbral debe ajustarse para que sean consistentes.

## 13.3 Boceto de referencia corregido

```java
// Task.java — dentro de fillTaskInfo(TaskInfo info, boolean stripExtras, ...)

private static final int MAX_SERIALIZED_URI_LENGTH = 64 * 1024; // documentar la base real del límite elegido

private Uri sanitizeUriForIpc(@Nullable Uri uri, int taskId) {
    if (uri == null) return null;
    final String uriString = uri.toString();
    if (uriString.length() <= MAX_SERIALIZED_URI_LENGTH) {
        return uri;
    }
    // Conserva scheme+authority; descarta el resto; añade metadatos de diagnóstico.
    return new Uri.Builder()
            .scheme(uri.getScheme())
            .authority(uri.getAuthority())
            .appendQueryParameter("__truncated__", "true")
            .appendQueryParameter("__original_length__", String.valueOf(uriString.length()))
            .build();
}

// Reemplazo del bloque original, preservando el comportamiento para intent == null
// y restableciendo flags explícitamente tras cloneFilter():
final Intent baseIntent = getBaseIntent();
final int baseIntentFlags = baseIntent == null ? 0 : baseIntent.getFlags();
info.baseIntent = baseIntent == null
        ? new Intent()
        : stripExtras ? baseIntent.cloneFilter() : new Intent(baseIntent);
info.baseIntent.setFlags(baseIntentFlags);
if (info.baseIntent.getData() != null) {
    info.baseIntent.setData(sanitizeUriForIpc(info.baseIntent.getData(), mTaskId));
}
```

```java
// TaskInfo.java — dentro de writeTaskToParcel(), como segunda capa de defensa

dest.writeTypedObject(baseIntent, 0);
dest.writeTypedObject(
    capturedLink != null && capturedLink.toString().length() > MAX_SERIALIZED_URI_LENGTH
        ? null
        : capturedLink,
    flags
);
```

## 13.4 Nota de eficiencia (no funcional, pero relevante en la ruta observada)

`sanitizeUriForIpc()` llama a `uri.toString()` para medir la longitud en cada invocación de `fillTaskInfo()`. En la cadena de reproducción documentada, este método se invoca con mucha frecuencia durante un bucle de fallos (decenas de veces por segundo). Materializar la cadena completa del URI —potencialmente de cientos de KB— solo para medir su longitud, en cada llamada, es evitable: el resultado puede cachearse en el propio objeto `Task` junto al `Uri` original, invalidándolo solo cuando el `Uri` cambia, en vez de recalcularse en cada serialización.

## 13.5 Qué NO corrige esta propuesta

Para que quede explícito y no se presente como solución completa:

- No corrige la variante observada en `ShellTaskOrganizer.registerOrganizer()`/`registerTaskOrganizer()` (fallo por agotamiento acumulativo del buffer, no por el tamaño de esa llamada concreta — ver §11).
- No implementa degradación en el lado cliente (SystemUI u otros consumidores) ante `DeadObjectException`/`DeadSystemException` — eso requiere el cambio descrito en la recomendación de la sección 9.1 de este documento, aplicado en el propio código de la plataforma.
- No aborda si el estado de tareas recientes persistido en disco puede rehidratar un URI ya truncado o uno aún sin truncar de una sesión anterior al fix.

---

Copyright © 2026 Manuel García Peña (Lostmon). Todos los derechos reservados.
