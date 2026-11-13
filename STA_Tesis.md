# STRUCTURED TEXT AMPLIFICATION (STA)

## Propagación no acotada de estado estructurado y amplificación de recursos a través de capas de procesamiento

**Título en inglés:** Structured Text Amplification (STA): Unbounded
Structured-State Propagation and Resource Amplification Across
Processing Layers

**Autor:** Manuel García Peña --- Lostmon\
**Investigación:** STA Research\
**Perfil:** Independent Security Researcher\
**Repositorio oficial:**
https://github.com/Lostmon/STA-Structured-Text-Amplification\
**Blog de investigación:** https://lostmon.blogspot.com\
**Propuesta de clasificación:** Unbounded Structured-State Propagation
Across Processing Layers\
**Ámbitos de investigación:** Android AOSP, Binder IPC, SavedState,
libminikin, Chromium, Gecko, arquitecturas de procesamiento de texto e
inteligencia artificial.

**Versión:** 2.0 --- Consolidated Research Thesis\
**Fecha:** Noviembre de 2026\
**Estado:** Investigación técnica independiente y propuesta taxonómica
en evaluación.

------------------------------------------------------------------------

# RESUMEN (ABSTRACT)

La evolución de las arquitecturas de software modernas ha incrementado
la dependencia de estructuras textuales como mecanismo universal de
comunicación, representación, persistencia y transferencia de
información.

URLs, URIs, Intents, documentos HTML, metadatos, estados serializados,
contenido enriquecido y entradas destinadas a modelos de lenguaje
atraviesan múltiples componentes y fronteras de procesamiento. Sin
embargo, las garantías de seguridad aplicadas durante la aceptación
inicial de estos datos no siempre se conservan durante las posteriores
operaciones de transformación, materialización, persistencia y
serialización.

Esta investigación presenta **Structured Text Amplification (STA)**, un
patrón arquitectónico de amplificación de recursos en el que una entrada
estructurada, o el estado derivado de ella, adquiere un coste
computacional, de memoria o de transporte desproporcionado al propagarse
entre diferentes capas de software sin presupuestos de recursos
suficientemente restrictivos.

La investigación identifica dos manifestaciones principales en el
ecosistema Android:

-   **Clase A --- State and IPC Amplification:** propagación y
    rematerialización de estados estructurados a través de mecanismos de
    persistencia, serialización y comunicación entre procesos.
-   **Clase B --- Computational and Rendering Amplification:**
    incremento desproporcionado del coste de procesamiento durante
    operaciones de parsing, medición, conformación de glifos y
    composición tipográfica.

El corpus de investigación reúne 32 vectores documentados, con
evidencias técnicas de diferentes rutas de entrada y procesamiento,
incluyendo trazas de ejecución, identificadores de compilación,
mediciones de recursos, análisis de código y reproducciones
experimentales.

La investigación incorpora además resultados de distintas versiones de
Android y fabricantes, así como el análisis de navegadores basados en
Chromium y Gecko.

Una línea complementaria examina comportamientos relacionados con la
amplificación de entradas estructuradas en sistemas de inteligencia
artificial, incluyendo interfaces de ingestión, tokenización,
procesamiento de contexto y generación. Estos resultados se estudian
como una extensión experimental del modelo, sin asumir que todos
comparten una causa raíz o que constituyen automáticamente la misma
debilidad CWE.

La tesis propone un modelo de contención basado en presupuestos
computacionales, validación independiente en cada frontera de
procesamiento, control de la rematerialización del estado y mecanismos
de degradación segura.

Finalmente, plantea una taxonomía específica para describir la
propagación no acotada de estado estructurado, cuya finalidad es
complementar las clasificaciones existentes de consumo incontrolado de
recursos.

**Contribución principal:** demostrar que la seguridad de una entrada
estructurada no puede evaluarse únicamente en el punto donde se recibe.
También debe garantizarse durante todo su ciclo de vida y en cada
componente que vuelve a materializarla o procesarla.

------------------------------------------------------------------------

# 1. INTRODUCCIÓN

## 1.1. Contexto

Las arquitecturas modernas se construyen mediante la composición de
múltiples subsistemas independientes.

En Android, una URL puede comenzar como contenido recibido por un
navegador, transformarse en un Intent, incorporarse al estado de una
tarea, persistirse, serializarse mediante Parcel y transmitirse a otros
procesos a través de Binder.

En un navegador, el mismo contenido puede atravesar el DOM, mecanismos
de selección, generación de vistas previas, interfaces de compartición y
componentes nativos de renderizado.

En sistemas de inteligencia artificial, una entrada puede transformarse
en documentos internos, secuencias de tokens, estructuras de contexto y
representaciones utilizadas por diferentes etapas de inferencia.

En todos estos escenarios aparece una característica común: el dato
original deja de ser una representación pasiva y pasa a participar en
operaciones cuyo coste depende de su estructura, composición y
tratamiento posterior.

La investigación STA parte de una pregunta:

**¿Qué sucede cuando una entrada que parece pequeña o manejable en su
punto de origen se convierte en una carga computacional o de estado
desproporcionada después de atravesar diferentes capas del sistema?**

La hipótesis de trabajo es que determinadas arquitecturas carecen de
controles acumulativos capaces de limitar el coste real de los datos
estructurados durante todo su ciclo de vida.

## 1.2. Problema de investigación

Las estrategias tradicionales de validación suelen concentrarse en:

-   Longitud de la entrada.
-   Tamaño del objeto recibido.
-   Validez sintáctica.
-   Límites del parser.
-   Límites de memoria de un componente.
-   Restricciones del mecanismo de transporte.

Sin embargo, estas comprobaciones pueden resultar insuficientes cuando
un dato validado inicialmente se transforma en diferentes
representaciones y es procesado repetidamente por consumidores
independientes.

Una entrada puede respetar el límite de un componente y, posteriormente,
superar los recursos disponibles en otro.

El problema no reside necesariamente en una operación individual, sino
en la ausencia de una política coherente de recursos a través de toda la
cadena de procesamiento.

## 1.3. Hipótesis principal

La hipótesis central de STA establece que:

> La ausencia de límites efectivos sobre el tamaño, la complejidad y la
> propagación del estado estructurado permite que entradas inicialmente
> aceptables generen un consumo desproporcionado de recursos en
> componentes posteriores, provocando fallos locales o sistémicos.

La hipótesis se considera respaldada cuando pueden establecerse
experimentalmente:

1.  Una entrada o estado de origen identificable.
2.  Una secuencia de transformaciones o propagaciones.
3.  Una o varias fronteras de procesamiento relevantes.
4.  Un incremento de recursos o coste atribuible a esa propagación.
5.  Una condición de fallo reproducible o técnicamente demostrable.

------------------------------------------------------------------------

# 2. DEFINICIÓN FORMAL DE STRUCTURED TEXT AMPLIFICATION

## 2.1. Definición

**Structured Text Amplification (STA)** es un patrón arquitectónico de
amplificación de recursos en el que una entrada textual estructurada, o
el estado derivado de ella, es aceptado, transportado o persistido sin
límites de recursos suficientemente efectivos y posteriormente
propagado, transformado, rematerializado o serializado a través de
múltiples capas de procesamiento.

Como consecuencia, uno o varios componentes posteriores pueden
experimentar un consumo desproporcionado de memoria, CPU, capacidad de
transporte o tiempo de ejecución, hasta superar sus límites operativos o
comprometer la estabilidad del sistema.

STA no requiere:

-   Un formato de serialización específico.
-   Un sistema operativo determinado.
-   Un mecanismo IPC concreto.
-   Una expansión exponencial.
-   Una relación fija entre bytes de entrada y recursos consumidos.
-   Una vulnerabilidad de corrupción de memoria.

Su característica diferencial es la interacción entre estado
estructurado insuficientemente acotado, propagación entre capas y
consumo de recursos en componentes posteriores.

## 2.2. Modelo matemático

Sea una entrada estructurada inicial:

\[ X_0 \]

Cada componente de la arquitectura aplica una transformación:

\[ X\_{i+1}=T_i(X_i) \]

Donde:

-   (X_i) representa el estado en una determinada capa.
-   (T_i) representa una operación de transformación, serialización,
    parsing, persistencia o materialización.
-   (X\_{i+1}) representa la nueva representación del estado.

El coste de procesar cada representación se define como:

\[ C_i=f_i(X_i,S_i) \]

Donde (S_i) representa el estado interno y las condiciones de ejecución
del componente.

El coste total de la cadena puede expresarse como:

\[ C\_{total}=`\sum`{=tex}*{i=0}\^{n}C_i+ `\sum`{=tex}*{j=1}\^{m}R_j \]

Donde (R_j) representa el coste adicional asociado a rematerializaciones
o entregas independientes a consumidores posteriores.

Para una arquitectura real, el coste también puede depender de la
concurrencia, la reutilización de objetos, el almacenamiento compartido
y la frecuencia de procesamiento.

Definimos el factor de amplificación de recursos:

\[ A_R=`\frac{R_{observado}}{R_{referencia}}`{=tex} \]

Donde ambos valores deben medirse bajo condiciones experimentales
comparables.

El factor puede calcularse por separado para:

-   Memoria.
-   CPU.
-   Tiempo de ejecución.
-   Tamaño serializado.
-   Número de operaciones.
-   Coste de inferencia.

No existe un factor universal de amplificación STA. La comparación debe
especificar qué recurso se mide, en qué capa y respecto a qué
referencia.

## 2.3. Concepto de presupuesto computacional

Cada frontera de procesamiento debería mantener un presupuesto
explícito:

\[ B_i=(M_i,T_i,S_i,F_i) \]

Donde:

-   (M_i): memoria máxima permitida.
-   (T_i): tiempo máximo de procesamiento.
-   (S_i): tamaño máximo de estado o representación.
-   (F_i): frecuencia o número de operaciones permitidas.

Una operación es admisible cuando permanece dentro de los límites
definidos para el componente y el contexto de ejecución.

La propiedad fundamental es que la validación de una capa no debe
considerarse una autorización ilimitada para todas las capas
posteriores.

## 2.4. Rematerialización del estado

La rematerialización constituye uno de los mecanismos centrales del
modelo STA.

Se produce cuando un componente reconstruye una representación de un
estado existente para entregarlo a otro consumidor.

Ejemplo:

\[ URI `\rightarrow `{=tex}Intent `\rightarrow `{=tex}TaskInfo
`\rightarrow `{=tex}Parcel `\rightarrow `{=tex}Binder \]

El mismo contenido lógico puede aparecer en representaciones diferentes
y ser serializado de forma independiente para varios consumidores.

La rematerialización no constituye por sí misma una vulnerabilidad. Se
convierte en un factor de riesgo cuando no existen límites efectivos
sobre el estado propagado y su procesamiento termina provocando
agotamiento de recursos o fallos demostrables.

------------------------------------------------------------------------

# 3. PRINCIPIOS FUNDAMENTALES DEL MODELO STA

## 3.1. Validación local frente a seguridad sistémica

Un componente puede cumplir correctamente su contrato local y, aun así,
participar en una cadena que termina comprometiendo la estabilidad del
sistema.

Por tanto:

\[ Validación\_{local}`\neq `{=tex}Seguridad\_{global} \]

La seguridad debe evaluarse sobre el ciclo de vida completo de los
datos.

## 3.2. Asimetría entre representación y coste

El tamaño en bytes de una entrada no determina necesariamente el coste
de procesarla.

Dos cadenas con una longitud semejante pueden generar costes diferentes
dependiendo de:

-   Codificación.
-   Composición Unicode.
-   Metadatos asociados.
-   Cantidad de segmentos.
-   Complejidad de los algoritmos.
-   Número de transformaciones.
-   Estructura de los objetos derivados.

En Android, por ejemplo, una cadena representada en UTF-8 puede ocupar
una cantidad distinta de memoria al convertirse en una representación
UTF-16.

Además, los objetos Spanned pueden incorporar metadatos asociados a
estilos y rangos.

La conversión de codificación, por sí sola, no implica una amplificación
de órdenes de magnitud. El coste adicional debe medirse incluyendo las
representaciones y estructuras realmente construidas.

## 3.3. La frontera de recursos como frontera de confianza

Cada transición entre componentes debe considerarse una nueva
oportunidad de validación.

Un consumidor no debería asumir que el componente anterior ha
garantizado todos los límites que necesita para procesar el dato.

## 3.4. La recuperación también forma parte de la superficie de ataque

Un fallo inicial puede convertirse en un incidente persistente cuando
los mecanismos de recuperación reconstruyen o redistribuyen
repetidamente el mismo estado problemático.

La recuperación debe poder descartar, aislar o degradar el estado que
provoca el fallo.

------------------------------------------------------------------------

# 4. TAXONOMÍA DE MANIFESTACIONES STA

La investigación distingue dos familias principales en Android y otras
líneas de estudio complementarias.

## 4.1. Clase A --- State and IPC Amplification

**Amplificación de estado, persistencia y comunicación entre procesos.**

Mecanismo:

-   Entrada estructurada.
-   Incorporación a objetos internos.
-   Persistencia o almacenamiento temporal.
-   Rematerialización.
-   Serialización.
-   Transporte mediante IPC.
-   Fallo por límites de recursos.

Componentes investigados:

-   Intent.
-   Bundle.
-   SavedState.
-   FragmentManager.
-   TaskInfo.
-   ActivityTaskManagerService.
-   TaskOrganizerController.
-   TransitionController.
-   RecentTasksController.
-   Binder IPC.
-   SystemUI.

Vectores representativos:

STA-001, STA-003, STA-005, STA-006/007, STA-009/010/018 y STA-012.

El efecto observable puede incluir TransactionTooLargeException, fallos
de restauración, terminación de procesos, reinicios repetidos y
denegación de servicio en cascada.

## 4.2. Clase B --- Computational and Rendering Amplification

**Amplificación algorítmica durante el procesamiento de texto.**

Mecanismo:

-   Entrada textual estructurada.
-   Parsing y segmentación.
-   Análisis Unicode y direccionalidad.
-   Resolución de saltos de línea.
-   Conformación de glifos.
-   Medición y composición tipográfica.
-   Bloqueo del hilo de interfaz.

Componentes investigados:

-   TextView.
-   StaticLayout.
-   LineBreaker.
-   Minikin.
-   HarfBuzz.
-   Paint.measureText().
-   Chromium.
-   Gecko.

Vectores representativos:

STA-017, STA-020 y STA-031.

En estos escenarios, el consumo de recursos puede estar relacionado con
la complejidad de la composición y con las rutas algorítmicas
utilizadas.

No debe atribuirse una complejidad O(n²) a todas las operaciones de
Minikin o HarfBuzz. La complejidad debe justificarse mediante análisis
del código, identificación de la ruta concreta y mediciones
reproducibles.

## 4.3. Clase C --- Interface and Ingestion Amplification

**Línea experimental en interfaces de ingestión de inteligencia
artificial.**

Esta categoría estudia las transformaciones que se producen antes de que
el contenido alcance el modelo.

Entre los fenómenos investigados:

-   Conversión de texto en documentos internos.
-   Fragmentación y expansión de contenido.
-   Duplicación de segmentos.
-   Transformaciones de formato.
-   Desajustes entre tamaño de entrada y representación entregada al
    modelo.

El caso documentado de Copilot describe una entrada de 61.560 caracteres
convertida en una representación interna de aproximadamente 1.002.682
caracteres, con una relación observada cercana a 16,28.

La causa identificada en ese experimento se sitúa en la interfaz de
ingestión, no necesariamente en el modelo de lenguaje.

## 4.4. Clase D --- Model and Context Processing Amplification

**Línea experimental sobre procesamiento de contexto e inferencia.**

Se investigan fenómenos como:

-   Incremento desproporcionado del coste de procesamiento.
-   Fragmentación de secuencias.
-   Entradas repetitivas de baja entropía.
-   Prolongación anómala de operaciones.
-   Pérdida de convergencia de tareas.
-   Errores de parsing y generación.

Los experimentos publicados incluyen pruebas con DeepSeek, Grok, Gemini,
Copilot, Leo y Qwen VL, con resultados y síntomas diferentes.

Esta categoría debe entenderse como una extensión experimental del
modelo general, no como una equivalencia automática con los mecanismos
Android de Clase A y B.

La tokenización, el coste de atención y la generación son mecanismos
diferentes. Una relación de amplificación de tokens no demuestra por sí
sola un defecto de propagación de estado.

------------------------------------------------------------------------

# 5. INVESTIGACIÓN FORENSE EN ANDROID

## 5.1. Motor nativo de texto: libminikin.so

El procesamiento tipográfico de Android incorpora distintas operaciones
de análisis, composición y representación.

Entre ellas:

-   Resolución de direccionalidad BiDi.
-   Segmentación de texto.
-   Line breaking.
-   Glyph shaping.
-   Medición de texto.
-   Integración con HarfBuzz.

La investigación documenta escenarios en los que determinadas
composiciones textuales provocan un incremento desproporcionado del
tiempo de procesamiento.

La característica relevante es que entradas aparentemente válidas pueden
alcanzar rutas nativas costosas antes de que el sistema aplique una
limitación suficientemente efectiva.

En determinados casos reproducidos, el procesamiento síncrono bloquea el
hilo principal y provoca ANR.

### 5.1.1. Sexto punto de entrada

Una de las ampliaciones de la investigación identifica la ruta:

Paint.measureText() → procesamiento de texto → HarfBuzz shaping.

El caso documentado en Google Docs amplía la superficie de investigación
más allá de los mecanismos tradicionales de line breaking.

Esto demuestra la importancia de estudiar los diferentes puntos de
entrada a los motores nativos, en lugar de limitar el análisis a una
única API de presentación.

### 5.1.2. Consecuencia arquitectónica

Las mitigaciones implementadas exclusivamente en aplicaciones
individuales pueden reducir la exposición, pero no necesariamente
protegen a otros consumidores que utilizan las mismas bibliotecas
nativas.

Por ello, la investigación considera necesaria una política de límites y
control de complejidad en las capas de plataforma.

## 5.2. Binder IPC y serialización de estado

Binder utiliza un mecanismo de transporte entre procesos con
restricciones de capacidad.

El límite de referencia habitualmente asociado a las transacciones es
aproximadamente 1 MiB, pero no debe interpretarse como un presupuesto
independiente y garantizado para cada objeto o transacción: intervienen
las condiciones del búfer compartido y otros factores del sistema.

La investigación estudia cómo estructuras derivadas de entradas externas
pueden alcanzar rutas de serialización sin una evaluación
suficientemente temprana de su tamaño real.

Entre las operaciones relevantes:

-   writeToParcel().
-   Serialización de Intent.
-   Serialización de Bundle.
-   Restauración de SavedState.
-   Transferencia de TaskInfo.
-   Consultas a servicios de gestión de tareas.

### 5.2.1. Cadena de propagación

Una representación inicial puede atravesar varias etapas:

Entrada externa → Intent → estado interno → TaskInfo → Parcel → Binder →
consumidor.

El problema se agrava cuando el mismo estado se vuelve a materializar
para consumidores independientes.

### 5.2.2. Excepción y recuperación

TransactionTooLargeException puede provocar el fallo de una operación de
transporte o restauración.

Cuando el componente receptor es un servicio compartido o persistente,
el impacto puede extenderse a otros procesos.

La existencia de una excepción no demuestra por sí sola un crash loop
sistémico. Este último requiere documentar la secuencia de fallos, los
reinicios, las dependencias y el comportamiento de recuperación.

------------------------------------------------------------------------

# 6. VECTORES Y EVIDENCIA MULTIPLATAFORMA

## 6.1. Corpus de investigación

La investigación STA reúne 32 vectores documentados en el corpus
consolidado.

El repositorio clasifica los casos principalmente en las familias de
IPC/SavedState y procesamiento nativo de texto.

Entre los vectores con documentación técnica destacada:

  Vector            Superficie                   Manifestación
  ----------------- ---------------------------- ----------------------------------
  STA-001           Context menu / URI           Procesamiento y fallo contextual
  STA-003           Share Intent / Binder        TransactionTooLargeException
  STA-005           WhatsApp / SavedState        Amplificación de estado
  STA-006/007       Transferencia y documentos   ANR y fallos IPC
  STA-009/010/018   Print Preview                Cascada de fallos de sistema
  STA-012           Deep links / SavedState      Fallo persistente
  STA-017           libminikin                   ANR durante procesamiento
  STA-020           Omnibox                      Bloqueo de interfaz
  STA-031           Paint.measureText            Ruta de shaping

Los identificadores representan vectores o rutas de entrada
documentados, no necesariamente debilidades CWE independientes.

## 6.2. Reproducibilidad entre fabricantes

La investigación incluye reproducciones en dispositivos de diferentes
fabricantes y generaciones de Android.

El conjunto de pruebas aportado por el investigador comprende:

-   Xiaomi Redmi Note 14 5G, Android 16, HyperOS 3.0.
-   Samsung Galaxy A13 5G, Android 14, One UI 6.1.
-   OPPO A6 5G, Android 15, ColorOS 15.

La existencia de manifestaciones en distintos fabricantes permite
investigar si el comportamiento procede de componentes compartidos de
Android o de implementaciones particulares de cada fabricante.

No obstante, la reproducción en tres OEM no demuestra automáticamente
una causa raíz común. Para establecerla deben correlacionarse las
trazas, las rutas de ejecución, las versiones de AOSP y los componentes
afectados.

## 6.3. Caso TaskInfo y SystemUI

Una de las líneas de investigación más relevantes examina la propagación
de estado asociado a tareas de Android.

Se estudian especialmente:

-   Task.fillTaskInfo().
-   TaskInfo.
-   TaskInfo.writeToParcel().
-   ActivityTaskManagerService.
-   TaskOrganizerController.
-   TransitionController.
-   RecentTasksController.
-   SystemUI.

El análisis aportado describe transacciones de aproximadamente 365 KB
hasta 1,11 MB en diferentes reproducciones y fallos asociados a varias
interfaces Binder.

En una reproducción documentada se observó una secuencia sostenida de
fallos de SystemUI que llegó a activar mecanismos de recuperación del
dispositivo.

La investigación también documenta una reproducción en la que
RescueParty solicitó un reinicio.

Este caso debe analizarse como una cadena de impacto y no como una
consecuencia universal de cualquier transacción excesiva.

### 6.3.1. Diferencia entre síntoma y causa raíz

El crash de SystemUI constituye el síntoma final.

La investigación de STA debe localizar las operaciones anteriores que
permiten que un estado de origen externo llegue a consumidores críticos
sin controles suficientes.

También debe distinguir qué campos contribuyen efectivamente al tamaño
de las transacciones. No se debe atribuir todo el volumen a Intent.data
o capturedLink sin una medición aislada que lo demuestre.

## 6.4. Convergencia de mitigaciones upstream

Durante 2026 se han identificado cambios independientes en AOSP,
AndroidX y Chromium relacionados con la contención de estados y payloads
voluminosos.

Entre los ejemplos incluidos en el repositorio:

-   AOSP InputMethod: a438ce17.
-   AndroidX Credential Manager: 393e20ae.
-   Chromium PDF Selection: 84b615a0.
-   AndroidX NotificationCompat: 90ffa6a7.
-   AndroidX PdfView: 8882927e.
-   Chromium Oversized Clipboard: 4751a769.
-   Chromium Native Messaging: f5c51669.
-   Chromium TLE Telemetry: 931ee1ab.

Los cambios representan diferentes estrategias:

**Constrain:** imponer límites antes de que el estado crezca.

**Redirect:** trasladar payloads voluminosos a mecanismos alternativos.

**Replace:** sustituir datos completos por identificadores o
referencias.

**Observe:** medir y registrar las operaciones que exceden los límites.

Esta convergencia es compatible con el modelo de STA y demuestra la
relevancia de las estrategias de contención del estado. No debe
presentarse como una confirmación oficial de que los responsables de
esos cambios reconocen una única vulnerabilidad STA.

------------------------------------------------------------------------

# 7. EXTENSIÓN DEL MODELO A INTELIGENCIA ARTIFICIAL

## 7.1. Motivación

Los sistemas de inteligencia artificial incorporan pipelines de
ingestión y transformación de información que presentan similitudes
arquitectónicas con otros sistemas multicapa.

El texto recibido por un asistente puede atravesar:

1.  Interfaz de entrada.
2.  Parser.
3.  Normalización.
4.  Conversión a documentos internos.
5.  Tokenización.
6.  Construcción del contexto.
7.  Inferencia.
8.  Generación.
9.  Renderizado de resultados.

Cada etapa introduce costes y representaciones propias.

La investigación explora si determinadas entradas estructuradas pueden
provocar amplificaciones no previstas durante estas operaciones.

## 7.2. Experimentos de ingestión

Los experimentos publicados documentan diferencias significativas entre
la entrada original y las representaciones construidas por determinadas
interfaces.

El caso de Copilot constituye un ejemplo de amplificación en la capa de
ingestión documental.

Se debe distinguir entre:

-   Tamaño del texto original.
-   Tamaño de la representación interna.
-   Número de tokens.
-   Tiempo de inferencia.
-   Coste de generación.

Estas métricas no son intercambiables.

## 7.3. Comportamiento del modelo

Las pruebas realizadas con diferentes asistentes han observado síntomas
como:

-   Respuestas excesivamente prolongadas.
-   Procesamiento prolongado.
-   Errores de parsing.
-   Fragmentación.
-   Pérdida de convergencia de tareas.
-   Generación de contenido no pertinente.

En particular, los experimentos publicados describen comportamientos
distintos entre DeepSeek, Grok, Gemini, Copilot, Leo y Qwen VL.

No debe afirmarse que todos utilizan la misma arquitectura de atención,
que todos presentan complejidad cuadrática efectiva o que todos
comparten una vulnerabilidad idéntica.

La investigación debe tratar estas diferencias como resultados
experimentales de cada implementación.

## 7.4. Seguridad de contexto e Instruction Drift

La investigación complementaria sobre Gemini estudia cambios en la
fidelidad de las instrucciones y generación de código inseguro bajo
determinadas condiciones de entrada.

Debe establecerse una distinción:

-   La amplificación de recursos pertenece al ámbito de STA.
-   El Instruction Drift pertenece al ámbito de comportamiento y
    seguridad del contexto.
-   La generación de código inseguro constituye un resultado de
    seguridad adicional.
-   La explotación de una vulnerabilidad en el código generado requiere
    validación independiente en el sistema consumidor.

La existencia de un comportamiento no demuestra automáticamente los
demás.

## 7.5. Límites de atribución

La investigación también ha explorado riesgos de contenido oculto,
renderizado de Markdown, solicitudes de imágenes externas y posibles
interacciones con sistemas de memoria.

Estos escenarios deben documentarse como líneas independientes de
investigación de seguridad de interfaces y privacidad.

Una petición HTTP externa, una inyección de instrucciones o una
modificación de memoria no constituyen por sí mismas STA.

Para integrarlas en el modelo sería necesario demostrar una relación
causal con la propagación no acotada y la amplificación de recursos
definida en esta tesis.

------------------------------------------------------------------------

# 8. TAXONOMÍA CWE: PROPUESTA DE NUEVA DEBILIDAD

## 8.1. Limitaciones de las clasificaciones existentes

Las categorías actuales describen aspectos importantes del problema,
pero no siempre expresan el mecanismo arquitectónico completo.

**CWE-400 --- Uncontrolled Resource Consumption**

Describe el agotamiento incontrolado de recursos, pero no caracteriza
específicamente la propagación y rematerialización del estado entre
capas.

**CWE-770 --- Allocation of Resources Without Limits or Throttling**

Describe la ausencia de límites en la asignación de recursos, pero no
necesariamente el ciclo de vida del estado estructurado.

**CWE-405 --- Asymmetric Resource Consumption (Amplification)**

Está relacionada con la amplificación asimétrica del consumo de
recursos.

**CWE-130 --- Improper Handling of Length Parameter Inconsistency**

Describe inconsistencias en parámetros de longitud.

**CWE-502 --- Deserialization of Untrusted Data**

Describe problemas relacionados con la deserialización de datos no
confiables.

Estas categorías pueden describir mecanismos o consecuencias presentes
en determinadas manifestaciones de STA, pero no deben considerarse
equivalentes.

## 8.2. Nombre propuesto

**Unbounded Structured-State Propagation Across Processing Layers**

Nombre corto: STA --- Structured Text Amplification.

## 8.3. Definición propuesta

Una debilidad en la que el software acepta o transporta entradas
textuales estructuradas sin imponer límites de recursos adecuados y
posteriormente conserva, propaga, transforma, rematerializa o serializa
el estado derivado de dichas entradas a través de múltiples capas de
procesamiento.

Los componentes posteriores pueden procesar ese estado sin aplicar
límites que tengan en cuenta su origen o su impacto acumulativo sobre
los recursos.

La repetición de estas operaciones puede provocar transacciones IPC
excesivas, consumo desproporcionado de recursos, fallos de transporte,
reconstrucción incompleta del estado, terminación de procesos o
denegación de servicio en cascada.

## 8.4. Relaciones propuestas

  Relación     CWE       Justificación
  ------------ --------- ----------------------------------------
  ChildOf      CWE-400   Consumo incontrolado de recursos
  ChildOf      CWE-770   Ausencia de límites de asignación
  Related to   CWE-405   Amplificación asimétrica
  PeerOf       CWE-130   Gestión de longitudes
  PeerOf       CWE-502   Deserialización de datos no confiables

Estas relaciones son propuestas para evaluación por CWE, no relaciones
formalmente asignadas.

CWE-833 no se considera una relación inherente: una manifestación STA
puede producir agotamiento de recursos y fallos sin que exista un
deadlock demostrado.

## 8.5. Criterios de identificación

Para identificar una posible debilidad STA deben acreditarse
conjuntamente:

1.  Una entrada estructurada o estado derivado insuficientemente
    acotado.
2.  Su propagación entre múltiples capas.
3.  Una o varias operaciones de transformación, materialización o
    serialización.
4.  La ausencia de límites adecuados en fronteras relevantes.
5.  Un agotamiento de recursos o condición de fallo atribuible a esa
    interacción.

Una entrada grande, una transacción grande o un crash aislado no son
suficientes para clasificar un caso como STA.

------------------------------------------------------------------------

# 9. ARQUITECTURA DE MITIGACIÓN

La mitigación de STA exige un enfoque de defensa en profundidad.

No basta con corregir el punto de entrada donde se manifiesta el fallo
si otros componentes pueden reconstruir y procesar el mismo estado.

## 9.1. Presupuestos de recursos por frontera

Cada transición debe definir límites independientes de:

-   Memoria.
-   Tamaño de representación.
-   Complejidad de procesamiento.
-   Tiempo de ejecución.
-   Frecuencia de operaciones.

Los límites deben adaptarse a las características del componente y al
modelo de amenaza.

## 9.2. Validación antes de la transformación

Los datos deben validarse antes de entrar en operaciones que puedan
multiplicar su coste.

Ejemplos:

-   Antes del parsing complejo.
-   Antes del layout.
-   Antes del shaping.
-   Antes de la serialización.
-   Antes de la persistencia.
-   Antes de una transacción IPC.

La validación debe contemplar tanto el tamaño como la composición cuando
la operación dependa de ambas propiedades.

## 9.3. Serialización segura en Android

Se recomienda:

1.  Estimar el coste de la representación serializada real.
2.  Aplicar límites antes de enviar transacciones.
3.  Evitar almacenar documentos completos en SavedState.
4.  Sustituir payloads voluminosos por identificadores.
5.  Utilizar referencias a almacenamiento persistente.
6.  Emplear transferencia mediante memoria compartida o mecanismos
    alternativos cuando corresponda.
7.  Implementar fallos controlados.
8.  Evitar ciclos de reintento que reproduzcan el mismo estado
    problemático.

### 9.3.1. TaskInfo y Task.fillTaskInfo()

La investigación propone estudiar y limitar los campos de tamaño
variable que se incorporan a TaskInfo, incluyendo Intent.data y
capturedLink.

El punto de intervención debe evaluarse en conjunto con los consumidores
downstream y las rutas de serialización.

La solución no debería limitarse a truncar una cadena arbitrariamente:
debe preservar la coherencia semántica del estado o sustituir el
contenido voluminoso por una referencia segura.

## 9.4. Protección del motor nativo de texto

Se propone:

-   Límites tempranos de entrada.
-   Presupuestos de complejidad.
-   Procesamiento incremental.
-   Límites de tiempo.
-   Evitar operaciones costosas en el hilo UI.
-   Degradación a una representación segura cuando sea viable.
-   Pruebas de regresión con distintas composiciones Unicode.

No se recomienda eliminar indiscriminadamente caracteres Unicode
legítimos, incluidos ZWJ, marcas combinantes o controles
bidireccionales.

La protección debe equilibrar seguridad, accesibilidad y corrección
lingüística.

## 9.5. Aislamiento y protección de interfaces de IA

Para las líneas experimentales de IA se proponen medidas
complementarias:

-   Limitar el tamaño y complejidad de documentos ingeridos.
-   Evitar duplicaciones durante la conversión a documentos internos.
-   Medir la relación entre entrada original y representación efectiva.
-   Aplicar límites de tokens y presupuestos de inferencia.
-   Separar datos externos de instrucciones de control.
-   Tratar el contenido externo como datos no confiables.
-   Exigir autorización para operaciones persistentes sensibles.
-   Controlar las solicitudes de recursos externos generadas durante el
    renderizado.

Estas medidas son controles de seguridad complementarios y no sustituyen
la necesidad de demostrar el mecanismo STA en cada implementación.

------------------------------------------------------------------------

# 10. METODOLOGÍA EXPERIMENTAL Y FORENSE

La validez de la investigación depende de que cada afirmación pueda
relacionarse con evidencias observables.

## 10.1. Identificación del punto de entrada

Documentar:

-   Procedencia del contenido.
-   Formato original.
-   Tamaño inicial.
-   Composición.
-   Operación que introduce el dato.
-   Interacción necesaria para activar el comportamiento.

## 10.2. Trazado de propagación

Seguir el dato a través de las diferentes capas.

Identificar:

-   Componentes participantes.
-   Funciones invocadas.
-   Transformaciones.
-   Representaciones intermedias.
-   Fronteras de serialización.
-   Consumidores independientes.

## 10.3. Instrumentación

Cuando sea posible, registrar:

-   Tamaño de entrada.
-   Tamaño de cada representación.
-   Memoria utilizada.
-   CPU.
-   Tiempo de ejecución.
-   Tamaño de transacciones.
-   Número de materializaciones.
-   Stack traces.
-   Identificadores de compilación.
-   Versiones del sistema.

## 10.4. Reproducibilidad

Cada experimento debe indicar:

-   Modelo de dispositivo.
-   Fabricante.
-   Versión de Android.
-   Versión de la aplicación.
-   Build ID.
-   Condiciones iniciales.
-   Procedimiento de reproducción.
-   Resultado esperado.
-   Resultado observado.
-   Número de repeticiones.
-   Evidencia adjunta.

## 10.5. Separación de evidencia y atribución

Se establecen cuatro niveles:

**Nivel 1 --- Observación:** se registra un comportamiento anómalo.

**Nivel 2 --- Reproducción:** el comportamiento se reproduce bajo
condiciones documentadas.

**Nivel 3 --- Correlación arquitectónica:** se identifica una cadena de
componentes y transformaciones compatible con el fallo.

**Nivel 4 --- Atribución causal:** se aísla el mecanismo responsable
mediante instrumentación, análisis de código o experimentos controlados.

No se debe atribuir una causa raíz únicamente por la coincidencia
temporal entre dos fallos.

## 10.6. Divulgación responsable

La investigación sigue una política de divulgación coordinada.

Los informes dirigidos a fabricantes y proveedores deben incluir
evidencias suficientes para facilitar su análisis, evitando publicar
innecesariamente payloads operativos que puedan facilitar la
explotación.

Los materiales públicos deben diferenciar entre:

-   Vulnerabilidades confirmadas.
-   Reproducciones independientes.
-   Problemas reportados.
-   Mitigaciones parciales.
-   Hipótesis todavía pendientes de validación.

------------------------------------------------------------------------

# 11. LIMITACIONES Y ALCANCE

Esta tesis reconoce varias limitaciones.

Primera: los 32 vectores documentados no representan necesariamente 32
causas raíz independientes. Algunos son diferentes rutas de entrada
hacia mecanismos compartidos.

Segunda: la presencia de un mismo comportamiento en varios dispositivos
no demuestra por sí sola que todos utilicen exactamente la misma
implementación vulnerable.

Tercera: las mediciones de amplificación dependen de las condiciones de
ejecución, versiones, componentes y recursos disponibles.

Cuarta: los comportamientos observados en LLMs no deben extrapolarse
automáticamente entre proveedores ni confundirse con defectos de memoria
o vulnerabilidades de ejecución de código.

Quinta: los cambios upstream identificados deben analizarse
individualmente. Su existencia no acredita por sí misma la corrección o
confirmación de todos los vectores STA.

Sexta: los impactos sobre integridad y confidencialidad requieren
pruebas específicas. La definición central de STA se centra en
propagación y agotamiento de recursos; no presupone acceso a información
privada, modificación de memoria persistente ni ejecución de código.

Estas limitaciones forman parte del rigor metodológico de la
investigación y permiten delimitar con precisión qué se ha demostrado y
qué continúa siendo objeto de estudio.

------------------------------------------------------------------------

# 12. CONTRIBUCIONES DE LA INVESTIGACIÓN

La investigación STA aporta las siguientes contribuciones:

**1. Formalización de un patrón arquitectónico.**

Identifica la propagación no acotada de estado estructurado como una
fuente de consumo desproporcionado de recursos.

**2. Modelo de análisis multicapa.**

Establece la rematerialización, la serialización y las fronteras de
procesamiento como elementos centrales para identificar el problema.

**3. Corpus técnico de vectores.**

Reúne 32 vectores documentados, con evidencias de diferentes superficies
de entrada y procesamiento.

**4. Investigación del framework Android.**

Relaciona rutas de IPC, SavedState, TaskInfo, Binder y procesamiento
nativo de texto.

**5. Investigación multiplataforma.**

Extiende el análisis a navegadores y sistemas de inteligencia artificial
mediante líneas experimentales diferenciadas.

**6. Propuesta de taxonomía.**

Formula una nueva categoría de debilidad para facilitar el
reconocimiento de fallos de propagación de estado entre capas.

**7. Arquitectura de mitigación.**

Propone presupuestos de recursos, validación por frontera,
rematerialización controlada y recuperación segura.

**8. Herramientas de investigación.**

El repositorio incorpora documentación y herramientas de análisis,
incluyendo STA Lab, Android Weight Analyzer, Pattern Generator y Pattern
Detector.

------------------------------------------------------------------------

# 13. CONCLUSIONES

Structured Text Amplification plantea una cuestión fundamental sobre la
seguridad de las arquitecturas modernas: un dato no conserva
necesariamente el mismo coste ni las mismas propiedades cuando atraviesa
diferentes componentes.

La investigación demuestra, mediante diferentes vectores y escenarios
documentados, que las entradas estructuradas pueden adquirir costes de
procesamiento y transporte que no quedan suficientemente controlados por
la validación inicial.

En Android, las manifestaciones investigadas abarcan dos superficies
especialmente relevantes:

-   La propagación y serialización del estado mediante IPC y mecanismos
    de persistencia.
-   El procesamiento algorítmico de texto en componentes nativos
    compartidos.

Ambas comparten una preocupación arquitectónica: la ausencia de límites
adecuados durante determinadas operaciones posteriores al punto de
entrada.

La investigación sobre navegadores e inteligencia artificial amplía el
campo de observación y plantea nuevas preguntas sobre la ingestión, la
transformación y el procesamiento de información estructurada.

Sin embargo, el valor principal de STA no consiste en afirmar que todos
los fallos de texto son iguales, sino en proporcionar un modelo que
permita identificar cuándo la propagación de un estado insuficientemente
acotado se convierte en una condición de agotamiento de recursos.

La seguridad no puede depender exclusivamente de que el primer
componente valide correctamente una entrada. Cada componente que la
recibe, transforma, conserva o vuelve a materializar debe asumir su
propia responsabilidad sobre los recursos que consume.

La resiliencia debe diseñarse para toda la cadena.

**El dato que entra por una puerta no tiene por qué ser el mismo que
llega a la siguiente. Y el coste de procesarlo tampoco.**

Esta es la premisa arquitectónica que da origen a Structured Text
Amplification.

------------------------------------------------------------------------

# 14. REFERENCIAS Y MATERIALES DE INVESTIGACIÓN

## Repositorio principal

Lostmon --- STA Structured Text Amplification:

https://github.com/Lostmon/STA-Structured-Text-Amplification

## Documentación teórica

-   STA Theoretical Framework.
-   Marco teórico de Structured Text Amplification.
-   Whitepaper v6: Resilience Gaps in Android IPC, SavedState and Text
    Layout.
-   README y catálogo de vectores.
-   Mitigations for Developers.

## Propuesta taxonómica

CWE New Weakness Submission --- Structured Text Amplification.

Archivo:

CWE_New_Weakness_Submission.md

## Investigación y cronología

-   Structured Text Amplification --- Marco conceptual.
-   Algorithmic DoS en libminikin.so.
-   libminikin: 10 años de vulnerabilidad en el núcleo de Android.
-   Timeline de la investigación STA.
-   STA --- UTF-16 Serialization Density Experiment.
-   Upstream Android and Chromium Changes in 2026.
-   Unbounded TaskInfo serialization in ActivityTaskManagerService.
-   STA in LLMs.
-   Structured Text Amplification en Gemini.

## Blog oficial

https://lostmon.blogspot.com

## Referencias técnicas externas

-   Android Open Source Project: https://source.android.com/
-   AOSP Minikin:
    https://android.googlesource.com/platform/frameworks/minikin/
-   Chromium: https://chromium.googlesource.com/chromium/src/
-   MITRE CWE: https://cwe.mitre.org/
-   HarfBuzz: https://harfbuzz.github.io/

------------------------------------------------------------------------

**STA RESEARCH**

**Manuel García Peña --- Lostmon**

Independent Security Researcher

Investigación independiente sobre seguridad arquitectónica, propagación
de estado estructurado y resiliencia de sistemas multicapa.

Noviembre de 2026.
