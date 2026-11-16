# STRUCTURED TEXT AMPLIFICATION (STA)

## Propagación de estado estructurado y amplificación de recursos a través de capas de procesamiento

**Tesis de investigación --- Versión 3.0**\
**Autor:** Manuel García Peña (Lostmon)\
**Fecha:** noviembre de 2026\
**Repositorio:**
[Lostmon/STA-Structured-Text-Amplification](https://github.com/Lostmon/STA-Structured-Text-Amplification)\
**Blog:** [lostmon.blogspot.com](https://lostmon.blogspot.com)

------------------------------------------------------------------------

## Resumen

Esta tesis desarrolla **Structured Text Amplification (STA)** como un
modelo arquitectónico para estudiar fallos de disponibilidad que
aparecen cuando una entrada textual estructurada atraviesa varias capas
de software y su coste de procesamiento, tamaño serializado o alcance de
propagación crece de forma desproporcionada respecto de la entrada
original.

La investigación se apoya en el whitepaper técnico STA v6.0, que reúne
evidencias en el ecosistema Android: propagación de estado mediante
`Intent`, `Bundle`, `SavedState`, Binder IPC y mecanismos de
persistencia; procesamiento síncrono de texto mediante `TextView`,
`StaticLayout`, `LineBreaker` y `libminikin`; y fallos observados en
aplicaciones, navegadores, servicios del sistema y capas de
personalización de fabricantes.

La tesis diferencia dos mecanismos empíricos principales:

-   **Class A --- State / IPC Amplification:** propagación,
    serialización, restauración o redistribución de estado estructurado
    hasta superar límites de transporte o recuperación.
-   **Class B --- Synchronous Text-Processing Amplification:**
    procesamiento de texto en rutas síncronas de interfaz, donde el
    coste de layout o medición puede bloquear el hilo principal y
    producir ANR.

Ambas clases se estudian bajo un marco común, pero no se consideran el
mismo mecanismo ni se reducen a una única causa raíz. Se incorpora
STA-027, relativo al deep link de Edge Ask Copilot, como extensión
contextual cuya asignación causal a Class A y Class B debe verificarse
por separado. También se explicitan las limitaciones de la muestra, la
discrepancia del catálogo entre 32 y 33 entradas y la diferencia entre
observación, reproducción, interpretación y atribución causal.

**Tesis central:** la disponibilidad de un sistema puede verse
comprometida no solo por una entrada malformada o por un defecto
aislado, sino también por la propagación de contenido estructurado a
través de componentes que aplican límites, presupuestos y estrategias de
recuperación incompatibles entre sí.

------------------------------------------------------------------------

## 1. Introducción

Las aplicaciones modernas rara vez procesan una entrada textual en un
único componente. Una URL, un enlace compartido, un borrador, un texto
seleccionado o un parámetro de deep link puede atravesar parsers,
adaptadores de interfaz, componentes de navegación, APIs del framework,
mecanismos IPC, sistemas de guardado de estado, motores de layout y
servicios privilegiados.

Cada componente puede aceptar el dato de acuerdo con sus propias reglas.
Sin embargo, la composición de esas decisiones puede producir un
comportamiento que no resulta evidente al observar cada componente de
forma aislada:

1.  Una entrada estructurada se acepta en un punto de entrada.
2.  El dato se transforma, codifica, encapsula o copia.
3.  Otros componentes lo conservan como estado, lo serializan o lo
    vuelven a procesar.
4.  El tamaño del estado o el coste computacional aumenta, o el dato
    alcanza una superficie de mayor impacto.
5.  Se rebasa un límite de transporte, memoria, tiempo de respuesta o
    recuperación.
6.  El fallo puede manifestarse lejos del origen y persistir después de
    cerrar o reiniciar una aplicación.

STA propone estudiar este fenómeno como un problema de **presupuestos y
límites a través de capas**, en lugar de limitar el análisis al tamaño
inicial del texto o al componente que finalmente falla.

### 1.1 Problema de investigación

¿De qué manera la propagación de texto estructurado a través de capas de
procesamiento independientes puede amplificar el coste computacional o
el estado serializado hasta provocar fallos de disponibilidad, y qué
controles arquitectónicos reducen ese riesgo sin depender exclusivamente
de validaciones en el punto de entrada?

### 1.2 Hipótesis

-   **H1.** La ausencia o inconsistencia de límites por etapa permite
    que una entrada estructurada alcance componentes posteriores con un
    coste o tamaño superior al previsto por los controles locales.
-   **H2.** En rutas de estado e IPC, la serialización, la anidación y
    la restauración pueden convertir una entrada relativamente pequeña
    en un estado agregado que excede los límites operativos de
    transporte o recuperación.
-   **H3.** En rutas de procesamiento síncrono de texto, determinadas
    entradas pueden provocar un coste de layout desproporcionado y
    bloquear el hilo de interfaz, incluso sin transferencia IPC de gran
    tamaño.
-   **H4.** Los controles aplicados únicamente en la aplicación de
    origen son insuficientes cuando el mismo dato puede alcanzar
    componentes compartidos del framework, bibliotecas nativas o
    servicios del sistema.

Son hipótesis de investigación: su validez debe evaluarse por vector y
mediante evidencia reproducible.

### 1.3 Objetivos

-   Definir STA mediante criterios técnicos verificables.
-   Diferenciar Class A y Class B.
-   Formalizar un modelo de propagación, amplificación y presupuesto por
    etapa.
-   Organizar las evidencias del whitepaper STA v6.0 sin confundir
    observación con causalidad.
-   Proponer controles en aplicación, framework, bibliotecas y
    recuperación.
-   Delimitar resultados confirmados, inferencias y trabajo pendiente.

------------------------------------------------------------------------

## 2. Definición de Structured Text Amplification

Se define **Structured Text Amplification (STA)** como:

> Un patrón arquitectónico de amplificación de recursos en el que una
> entrada textual estructurada atraviesa múltiples capas de software y,
> como resultado de sus transformaciones, conservación, serialización,
> redistribución o procesamiento, provoca un crecimiento
> desproporcionado del coste computacional, del tamaño del estado o del
> alcance de los recursos comprometidos, hasta poder superar los límites
> operativos de uno o más componentes posteriores.

La definición incluye entradas como URLs, deep links, texto de
documentos, contenido seleccionado, borradores, extras de `Intent` y
otros datos textuales estructurados. No exige que la entrada sea
sintácticamente inválida ni que exista corrupción de memoria.

STA no es una vulnerabilidad única ni afirma que todos los casos
compartan una implementación defectuosa. Es un **marco de análisis**
para identificar patrones recurrentes de propagación y amplificación
entre componentes.

### 2.1 Criterios de inclusión

Un caso puede considerarse candidato a STA cuando se documentan varios
de estos elementos:

1.  **Entrada estructurada identificable:** puede determinarse qué dato
    inicia la cadena.
2.  **Propagación entre etapas:** el dato atraviesa más de un componente
    o capa.
3.  **Transformación o conservación:** se copia, encapsula, serializa,
    restaura, renderiza, mide o redistribuye.
4.  **Desproporción:** el coste o tamaño resultante no guarda una
    relación operativamente segura con la entrada.
5.  **Límite alcanzado:** se identifica un límite de tamaño, tiempo,
    memoria, IPC o recuperación.
6.  **Impacto reproducible:** se observa un fallo de disponibilidad o
    degradación atribuible a la cadena.

No todos los casos documentados demuestran todos los criterios con el
mismo nivel de detalle. La clasificación probatoria debe reflejarlo.

### 2.2 Exclusiones

No todo texto largo constituye STA. Tampoco lo es automáticamente
cualquier ANR, `TransactionTooLargeException`, fallo de navegación o
agotamiento de memoria.

Un caso aislado de texto costoso, sin propagación ni amplificación
demostrable, puede ser un problema de complejidad algorítmica o de
validación de entrada, pero no necesariamente STA en sentido estricto.

La inyección de prompts, las instrucciones ocultas Unicode y los ataques
contra sistemas LLM son líneas relacionadas cuando interviene texto
estructurado, pero **no son STA por definición**. Solo deben
incorporarse cuando se demuestre la cadena específica de propagación y
amplificación de recursos.

------------------------------------------------------------------------

## 3. Modelo arquitectónico

### 3.1 Cadena de procesamiento

``` text
Entrada estructurada
       |
       v
Punto de entrada / parser
       |
       v
Transformación / normalización
       |
       v
API de aplicación o framework
       |
       +-------------------------------+
       |                               |
       v                               v
Estado / IPC / persistencia       Renderizado / medición
       |                               |
       v                               v
Restauración / redistribución     Layout / hilo de interfaz
       |                               |
       +---------------+---------------+
                       |
                       v
             Límite operativo alcanzado
                       |
                       v
         ANR / excepción / crash loop /
           degradación / pérdida de disponibilidad
```

El diagrama es conceptual. No implica que todos los vectores recorran
todas las etapas ni que el orden sea idéntico en todos los productos.

### 3.2 Modelo de transformación

Sea (X_0) la entrada estructurada y (T_i) la transformación aplicada por
la etapa (i):

\[ X\_{i+1}=T_i(X_i,S_i) \]

donde (S_i) representa el contexto de ejecución: estado previo,
metadatos, estructura de contenedores, configuración de UI, historial,
número de objetos y demás condiciones relevantes.

El tamaño observable del estado puede expresarse como:

\[ M_i=`\operatorname{size}`{=tex}(X_i) \]

El coste de procesamiento de una etapa puede representarse como:

\[ C_i=f_i(X_i,S_i) \]

El coste no tiene por qué crecer linealmente con (M_i). Puede depender
de la estructura, la distribución de caracteres, la profundidad de
anidación, el número de elementos y las operaciones de layout.

Para una ruta (P), el coste agregado puede aproximarse como:

\[ C_P=`\sum`{=tex}\_{i`\in `{=tex}P}C_i \]

Es un marco contable, no una afirmación de que las etapas sean
independientes o estrictamente aditivas en una implementación real.

### 3.3 Factor de amplificación

Cuando existe una medida de referencia adecuada:

\[ A_M=`\frac{M_{\text{observed}}}{M_{\text{input}}}`{=tex} \]

para amplificación de tamaño, o:

\[ A_C=`\frac{C_{\text{observed}}}{C_{\text{baseline}}}`{=tex} \]

para amplificación de coste.

El valor solo es significativo si se especifican unidad, referencia,
punto de medición y procedimiento. No deben compararse factores
calculados con métricas o condiciones distintas.

### 3.4 Presupuesto por etapa

Cada componente debería imponer un presupuesto explícito:

\[ B_i=(M_i,T_i,D_i,N_i,R_i) \]

-   (M_i): tamaño máximo aceptable.
-   (T_i): tiempo máximo de procesamiento.
-   (D_i): profundidad máxima de anidación.
-   (N_i): número máximo de elementos u operaciones.
-   (R_i): presupuesto de recursos o política de recuperación.

La propiedad deseable no es simplemente que la entrada inicial sea
pequeña, sino que **cada etapa valide el coste que realmente va a
asumir**.

------------------------------------------------------------------------

## 4. Mecanismos empíricos: Class A y Class B

### 4.1 Class A --- State / IPC Amplification

Class A comprende rutas donde el dato estructurado se conserva,
encapsula, serializa, restaura o redistribuye como estado de aplicación
o del sistema.

Componentes y mecanismos estudiados:

-   `Intent` extras y `Intent.EXTRA_TEXT`.
-   `Bundle` y `SavedState`.
-   `SavedStateRegistry` y `FragmentManager`.
-   `TaskInfo`, `TaskPersister` y observadores del estado de tareas.
-   Binder IPC y servicios del sistema.
-   Restauración de estado después de la muerte de un proceso.
-   SystemUI y componentes OEM que vuelven a consultar el estado
    persistido.

#### 4.1.1 Secuencia típica

1.  Una entrada llega a una aplicación o componente.
2.  Se incorpora a un `Bundle`, argumento, borrador o estado de
    navegación.
3.  La estructura se anida o se replica entre componentes.
4.  El estado agregado se serializa o transmite mediante IPC.
5.  Se supera un límite operativo o falla la restauración.
6.  El estado problemático puede conservarse y provocar fallos repetidos
    al reabrir la aplicación o reiniciar un componente.

Un ejemplo del whitepaper describe un argumento de fragmento de **5.812
bytes** que aparece dentro de un estado agregado de **1.661.088 bytes**
tras varios niveles de anidación de `childFragmentManager`. El factor
aproximado reportado es de 286× para ese caso concreto; no debe
extrapolarse como factor universal de `FragmentManager`.

Otros ejemplos incluyen excepciones `TransactionTooLargeException` en
operaciones de contexto y Share Intent, y cadenas de fallo que alcanzan
SystemUI. El impacto puede ir desde el fallo de una acción puntual hasta
un ciclo de caída persistente. La persistencia depende de dónde se
conserva el estado y de cómo se comporta la restauración.

#### 4.1.2 Binder y límites de transporte

Binder es una infraestructura IPC compartida. La investigación documenta
transacciones que fallan por tamaño y agotamiento de recursos, pero el
comportamiento real depende del buffer compartido, concurrencia, objetos
transferidos y condiciones del sistema.

Por ello, Binder no debe describirse como si ofreciera un límite aislado
y universal de un megabyte por transacción en cualquier contexto. El
whitepaper utiliza 1.048.576 bytes como referencia en determinadas
evidencias y rutas; la interpretación debe conservar el contexto de cada
captura.

### 4.2 Class B --- Synchronous Text-Processing Amplification

Class B comprende rutas donde texto estructurado alcanza operaciones de
procesamiento síncrono que consumen un tiempo desproporcionado,
especialmente en el hilo principal de una interfaz.

Ruta técnica destacada:

``` text
TextView / componente de interfaz
        |
        v
StaticLayout / DynamicLayout
        |
        v
LineBreaker
        |
        v
libhwui / libminikin
        |
        v
LineBreakOptimizer::computeBreaks
        |
        v
Bloqueo del hilo principal / ANR
```

El whitepaper documenta trazas nativas donde `libminikin.so`, en
particular `LineBreakOptimizer::computeBreaks`, aparece en el stack del
hilo principal durante ANR observados en aplicaciones basadas en
Chromium y Gecko.

También identifica rutas adicionales de medición y renderizado de
sugerencias de omnibox, así como `breakLineGreedy` en otra ruta.
Sustituir un algoritmo por otro no debe asumirse como solución
suficiente si el nuevo camino carece de límites de longitud o tiempo.

#### 4.2.1 Características

-   El dato puede ser una URL sintácticamente válida.
-   El fallo se produce durante layout, medición o renderizado.
-   El hilo principal queda bloqueado y deja de responder a eventos.
-   La manifestación habitual documentada es un ANR, no necesariamente
    una excepción IPC.
-   La reproducción puede depender del punto de entrada, del patrón del
    texto y del componente que solicita el layout.

Las duraciones observadas deben atribuirse a las capturas concretas; no
constituyen una garantía para cualquier dispositivo.

#### 4.2.2 Complejidad algorítmica

El documento relaciona determinadas rutas de `libminikin` con el
algoritmo de optimización de saltos de línea y describe un
comportamiento de coste elevado. No debe afirmarse que todo
`libminikin`, todo procesamiento de texto Android o todas las rutas de
Minikin sean (O(n\^2)).

La complejidad debe atribuirse a la función, versión, ruta y condiciones
analizadas. La tesis propone medir coste frente a longitud y estructura
del texto, y verificar por separado las rutas óptima y greedy.

### 4.3 Comparación

  -----------------------------------------------------------------------
  Dimensión               Class A                 Class B
  ----------------------- ----------------------- -----------------------
  Recurso dominante       Tamaño y propagación de Tiempo de CPU / bloqueo
                          estado                  síncrono

  Ruta representativa     Bundle, SavedState,     TextView, StaticLayout,
                          Binder, persistencia    LineBreaker, libminikin

  Límite                  IPC, serialización,     Presupuesto temporal de
                          restauración            UI y coste de layout

  Manifestación           Excepción, crash, crash ANR, congelación de
                          loop, fallo de servicio interfaz

  Persistencia            Puede persistir si el   Normalmente
                          estado se guarda        transitoria, salvo
                                                  interacción con
                                                  historial/estado

  Mitigación principal    Límites de estado y     Límites previos al
                          recuperación segura     layout y degradación
                                                  segura
  -----------------------------------------------------------------------

**Conclusión de clasificación:** Class A y Class B son mecanismos
diferentes bajo una misma hipótesis arquitectónica. No se debe inferir
uno a partir del otro ni sumar sus evidencias como si demostraran una
única ruta causal.

------------------------------------------------------------------------

## 5. Extensión contextual: STA-027 y deep links

El whitepaper v6.0 incorpora **STA-027 --- Edge Ask Copilot Deep Link
Amplification**, relativo a Microsoft Edge Ask Copilot
(`com.microsoft.copilotn`). El vector utiliza un deep link cuyo
parámetro `initialText` contiene una entrada extensa que llega al
manejador de deep links y se incorpora a una ruta de Jetpack Compose
Navigation.

La evidencia descrita incluye una `IllegalArgumentException` durante la
resolución del destino del `NavGraph`: la ruta expandida no se reconoce
como hija directa del grafo. El documento identifica el expediente MSRC
como **VULN-196348 / Case 122975** y reporta una estimación CVSS 3.1 de
6.5 realizada por el investigador.

Conviene separar tres afirmaciones:

1.  **Observación reportada:** `initialText` llega a la ruta y se
    produce un fallo de resolución del `NavGraph`.
2.  **Interpretación de propagación:** la expansión del dato dentro de
    la ruta puede exceder las expectativas estructurales del componente
    de navegación.
3.  **Amplificación de recursos:** para sostener Class B se necesita
    evidencia de trabajo costoso de layout o medición; para Class A se
    debe demostrar que el mecanismo corresponde a propagación de
    estado/IPC/SavedState y no solo a una ruta inválida.

El whitepaper lo etiqueta como comportamiento combinado Class A + Class
B. Esta tesis conserva el registro, pero recomienda tratar esa
asignación como **hipótesis que requiere trazabilidad causal por
separado**. Una excepción de NavGraph no prueba por sí sola un ANR de
`libminikin` ni un desbordamiento Binder.

El expediente **VULN-196348 / Case 122975** debe mantenerse separado de
otros avisos o expedientes de Ask Copilot, salvo que documentación
primaria confirme que se trata del mismo caso.

STA-027 no convierte STA en una teoría general de prompt injection. La
seguridad de prompts, el contenido Unicode oculto y la inyección
indirecta son riesgos relacionados, pero su inclusión en STA exige
demostrar la cadena específica de propagación y amplificación de
recursos.

------------------------------------------------------------------------

## 6. Evidencia y metodología

### 6.1 Base empírica

El whitepaper STA v6.0 describe investigación independiente con
observaciones desde 2022 y formalización ampliada durante 2026. La
metodología incluye:

-   Reproducción directa en dispositivos físicos de producción.
-   Captura de `adb bugreport`, `logcat`, trazas ANR y tombstones.
-   Análisis de tamaños de `Bundle` y excepciones Binder.
-   Inspección de rutas públicas de AOSP y AndroidX.
-   Comparación entre aplicaciones, navegadores y fabricantes.
-   Revisión de notificaciones y respuestas de proveedores.

La muestra declarada comprende siete dispositivos físicos, cuatro capas
OEM (HyperOS, One UI, ColorOS y OxygenOS), dispositivos Pixel y una
tableta Lenovo, con Android 13--16. El Redmi Note 14 5G se probó a
través de tres configuraciones de software, desde Android 14 hasta
Android 16. La muestra no equivale a una prueba exhaustiva de todos los
fabricantes, modelos o versiones.

### 6.2 Taxonomía probatoria

  -----------------------------------------------------------------------
  Nivel                   Criterio del whitepaper Interpretación
  ----------------------- ----------------------- -----------------------
  **Tier A ---            Evidencia de stack      Reproducción con
  Confirmed**             completo, excepción y/o evidencia técnica
                          análisis de Bundle      directa
                          según el vector         

  **Tier B ---            Crash o ANR observado,  Caso observado;
  Behavioral**            con mecanismo inferido  causalidad pendiente de
                          por analogía o          completar
                          evidencia incompleta    

  **Tier C --- Mitigated  Escenario probado sin   Control negativo o
  / control**             reproducir el fallo     comportamiento mitigado
  -----------------------------------------------------------------------

Tier A no significa que todos los detalles de causa raíz hayan sido
auditados por el proveedor. Significa que el vector dispone del tipo de
evidencia técnica definido por el whitepaper.

### 6.3 Separación de afirmaciones

Cada resultado debería etiquetarse como:

-   **Observación:** evento visto en dispositivo, log o interfaz.
-   **Reproducción:** procedimiento repetible bajo condiciones
    documentadas.
-   **Evidencia de ejecución:** stack trace, excepción, tamaño de estado
    o traza nativa.
-   **Correlación:** coincidencia entre el evento y una ruta o cambio de
    código.
-   **Interpretación arquitectónica:** explicación propuesta para
    conectar etapas.
-   **Atribución causal:** evidencia suficiente para afirmar que una
    operación concreta produce el fallo.
-   **Confirmación externa:** validación explícita del proveedor o
    tercero.

Una correlación temporal o coincidencia de CWE no demuestra que un
commit corrija un vector concreto. La confirmación de un fabricante
sobre un caso tampoco valida automáticamente todos los vectores de la
misma familia conceptual.

### 6.4 Catálogo de vectores

El whitepaper contiene una discrepancia editorial: la portada y el
resumen ejecutivo hablan de **32 vectores**, mientras que el índice y la
sección del catálogo anuncian **33**. Hasta auditar identificadores,
variantes y duplicados, esta tesis usa la expresión **"catálogo
documentado de STA v6.0"** y no presenta 32 o 33 como recuento
definitivo de causas raíz independientes.

Un vector es una instancia documentada; varias entradas pueden compartir
causa raíz, componente o cadena de activación. El número de entradas no
equivale al número de vulnerabilidades independientes.

------------------------------------------------------------------------

## 7. Evidencias arquitectónicas destacadas

### 7.1 FragmentManager y SavedState

Los casos de estado guardado muestran cómo argumentos y datos de
interfaz pueden propagarse por jerarquías de fragmentos y terminar
serializados como un `Bundle` agregado. La investigación documenta
tamaños de estado superiores a los esperados y excepciones durante
transacciones o restauraciones.

La mitigación debe considerar el tamaño del dato inicial, la
multiplicación por anidación, el número de componentes y la posibilidad
de restaurar un estado previamente persistido.

### 7.2 SystemUI y estado de tareas

El whitepaper describe una cadena en la que un dato asociado a una tarea
llega a estructuras de estado observadas por componentes del sistema. En
determinados casos de HyperOS, la relectura del estado por un componente
OEM después del reinicio de SystemUI puede volver a activar el fallo.

En la captura del 16 de septiembre de 2026 se informa de un error Binder
`-28` ("No space left on device", asociado a `ENOSPC`) durante una
transacción de `IActivityTaskManager`, seguido de eventos
`DeadObjectException`/`DeadSystemException` y nuevos fallos de SystemUI.
La interpretación debe limitarse a esa captura y secuencia; no prueba
que todos los fallos Binder tengan la misma causa.

### 7.3 libminikin y motores de navegador

Las trazas del whitepaper sitúan
`libminikin.so::LineBreakOptimizer::computeBreaks` en el hilo principal
durante ANR de navegadores y otras aplicaciones. Su presencia en
Chromium y Gecko es relevante porque el componente señalado pertenece a
la plataforma Android y no exclusivamente al motor del navegador.

La presencia de una biblioteca compartida no significa que todos los
productos que la utilizan sean vulnerables bajo cualquier entrada.

### 7.4 Evidencia AOSP y AndroidX

El whitepaper recoge cambios relacionados con el transporte de payloads
grandes mediante mecanismos alternativos a Binder, incluido el trabajo
denominado `LargePayloadSupport` y cambios de AndroidX SavedState. Son
corroboración arquitectónica de que el tamaño de datos transportados por
IPC y la separación entre estado y Binder son problemas reales de
ingeniería.

No constituyen por sí solos prueba de que Google haya reconocido,
corregido o atribuido cada vector STA. La correspondencia entre un
commit y un vector concreto requiere comparar código afectado, versión,
ruta de ejecución y condición de fallo.

------------------------------------------------------------------------

## 8. Modelo de amenazas

### 8.1 Capacidades del atacante

Según el vector, el atacante podría controlar o influir en:

-   Un enlace o URL que se abre en una aplicación.
-   Contenido HTML o documento que incorpora enlaces o texto.
-   Un deep link con parámetros.
-   Texto copiado, pegado o compartido.
-   Un borrador o contenido que la aplicación conserva como estado.
-   Datos que posteriormente se muestran, miden, serializan o restauran.

No se presupone ejecución de código, corrupción de memoria ni
privilegios elevados.

### 8.2 Interacción requerida

Los vectores pueden requerir clic, long-press, pegar texto, abrir una
conversación, enfocar una barra de direcciones o activar
compartir/traducir. La interacción exacta debe figurar en cada ficha; no
debe generalizarse a "cero interacción" si el escenario requiere que la
víctima siga un enlace.

### 8.3 Activos afectados

-   Disponibilidad de una aplicación.
-   Disponibilidad de una interfaz o servicio del sistema.
-   Capacidad de restaurar sesiones o tareas.
-   Historial, borradores o datos de conversación.
-   Recursos de CPU y memoria durante el procesamiento.

Confidencialidad e integridad no se consideran impactos demostrados por
STA salvo que un vector aporte evidencia independiente.

------------------------------------------------------------------------

## 9. Mitigaciones y diseño defensivo

La defensa debe ser distribuida. Un único límite en el punto de entrada
no basta si los componentes posteriores transforman, duplican, anidan o
vuelven a procesar el dato.

### 9.1 Aplicación

-   Establecer límites de longitud y tamaño para URLs, deep links,
    borradores y extras.
-   Validar parámetros antes de incorporarlos a rutas, argumentos o
    estado persistente.
-   Evitar copiar íntegramente contenido no confiable a múltiples capas.
-   Truncar de forma segura textos de previsualización y sugerencias.
-   No persistir contenido que exceda el presupuesto definido.
-   Ofrecer recuperación que no requiera borrar todos los datos cuando
    sea técnicamente posible.

### 9.2 Framework y serialización

-   Definir límites de tamaño y profundidad antes de serializar.
-   Evitar que el estado de UI replique datos grandes en jerarquías
    anidadas.
-   Diseñar rutas de restauración que descarten de forma segura estado
    inválido o excesivo.
-   Evitar que una excepción de restauración provoque un ciclo de
    persistencia y caída.
-   Usar mecanismos adecuados para payloads grandes ---por ejemplo,
    referencias o descriptores de archivo cuando corresponda--- en vez
    de transportar indiscriminadamente grandes bloques por Binder.
-   Instrumentar el tamaño agregado del estado, no solo el de cada
    argumento individual.

### 9.3 Procesamiento de texto y renderizado

-   Aplicar límites antes de ejecutar algoritmos costosos de layout o
    medición.
-   Establecer presupuestos temporales y cancelación cuando sean
    viables.
-   Evitar que el hilo principal procese entradas sin límite.
-   Verificar todas las rutas de line breaking, no solo una
    implementación concreta.
-   Degradar de forma segura: truncamiento, renderizado parcial o
    presentación diferida.
-   Añadir pruebas con entradas estructuradas y distribuciones de
    caracteres variadas.

### 9.4 Navegación y deep links

-   Tratar los parámetros de deep link como datos no confiables.
-   Separar el identificador de ruta del contenido textual extenso.
-   Evitar concatenar parámetros arbitrarios sin codificación y
    validación.
-   Limitar el tamaño de cada parámetro y el tamaño total de la ruta.
-   Manejar fallos de resolución del grafo sin cerrar el proceso ni
    conservar indefinidamente un estado inválido.

### 9.5 Recuperación y observabilidad

-   Detectar ciclos de restauración y limitar intentos repetidos.
-   Mantener una vía de arranque limpio que permita descartar estado
    corrupto o excesivo.
-   Registrar tamaños, profundidad, tiempo de procesamiento y etapa
    donde se supera el presupuesto.
-   Conservar trazas que distingan ANR de UI, agotamiento IPC y errores
    de navegación.
-   Evitar registros que expongan contenido sensible; medir tamaño y
    metadatos cuando sea suficiente.

------------------------------------------------------------------------

## 10. Programa de validación experimental

Cada vector debería documentarse con una ficha normalizada:

  Campo                Contenido requerido
  -------------------- ----------------------------------------------------------
  Identificador        ID estable y único
  Producto y versión   Aplicación, versión, sistema y build
  Dispositivo          Modelo, OEM y RAM si se conoce
  Entrada              Tipo, tamaño, estructura y método de entrega
  Precondiciones       Estado de la app, historial, sesión y permisos
  Pasos                Secuencia exacta de reproducción
  Etapas               Componentes atravesados en orden
  Métrica              Tamaño, tiempo, memoria, profundidad o número de objetos
  Resultado            ANR, excepción, crash, persistencia o degradación
  Evidencia            Logs, bugreport, stack, captura y hashes
  Repetibilidad        Número de intentos y tasa de éxito
  Control negativo     Entrada equivalente que no activa el fallo
  Clasificación        Class A, Class B, extensión o fuera de alcance
  Confianza            Tier y limitaciones
  Mitigación           Cambio propuesto y resultado tras aplicarlo

### 10.1 Experimentos prioritarios

1.  **Curvas de tamaño:** medir entrada inicial y tamaño serializado en
    cada etapa de Class A.
2.  **Profundidad de anidación:** variar niveles de fragmentos y
    observar crecimiento del estado.
3.  **Curvas temporales:** medir tiempo de layout frente a longitud y
    estructura para cada ruta Class B.
4.  **Distribución de caracteres:** comparar entradas de igual longitud
    con patrones distintos, sin asumir que un carácter concreto es
    universalmente costoso.
5.  **Persistencia:** comprobar si el estado sobrevive a cierre, muerte
    de proceso, reinicio de componente o reinicio de dispositivo.
6.  **Controles negativos:** repetir con texto truncado, codificación
    alternativa o límites activados.
7.  **Pruebas de mitigación:** demostrar que el límite se aplica antes
    del coste y que el sistema se recupera sin bucle.

Toda afirmación cuantitativa debe incluir dispositivo, build, versión de
aplicación, configuración, tamaño exacto de entrada, procedimiento,
número de repeticiones y método de medición. Las estimaciones CVSS deben
etiquetarse como valoraciones del investigador, no como puntuaciones
asignadas por el proveedor.

------------------------------------------------------------------------

## 11. Limitaciones

1.  **Muestra finita:** siete dispositivos físicos no representan todo
    el ecosistema Android.
2.  **Variación de firmware:** fabricantes y versiones pueden alterar
    las rutas.
3.  **Evidencia heterogénea:** no todos los vectores disponen de stack
    completo o de la misma calidad de medición.
4.  **Discrepancia del catálogo:** el whitepaper alterna entre 32 y 33
    entradas; debe auditarse antes de fijar el total.
5.  **Causalidad:** una secuencia temporal o correlación entre logs no
    basta para atribuir la causa raíz.
6.  **Complejidad:** no debe generalizarse una complejidad algorítmica a
    toda una biblioteca sin análisis por función y ruta.
7.  **Límites Binder:** el comportamiento depende de recursos
    compartidos y condiciones del sistema; no debe reducirse a un límite
    universal aislado.
8.  **CVSS:** las puntuaciones publicadas son estimaciones del
    investigador y pueden cambiar tras evaluación independiente.
9.  **STA-027:** la excepción de navegación está documentada; la
    asignación simultánea a Class A y Class B requiere demostrar cada
    mecanismo por separado.
10. **Corroboración upstream:** un cambio de código cercano
    conceptualmente no demuestra que el proveedor haya corregido o
    reconocido un vector concreto.

Estas limitaciones no invalidan las reproducciones documentadas, pero
impiden extrapolar cada resultado a todos los dispositivos, aplicaciones
o versiones.

------------------------------------------------------------------------

## 12. Trabajo futuro

-   Auditar el catálogo v6.0 y reconciliar las 32/33 entradas, variantes
    y duplicados.
-   Publicar una matriz vector → evidencia → ruta → causa raíz →
    mitigación.
-   Separar causas raíz compartidas de variantes de activación.
-   Completar mediciones cuantitativas de `libminikin`,
    `breakLineOptimal` y `breakLineGreedy`.
-   Verificar el comportamiento de Binder bajo carga concurrente y
    distintas condiciones de memoria.
-   Analizar la restauración de `TaskInfo`, `TaskPersister` y servicios
    OEM en builds adicionales.
-   Completar una ficha causal de STA-027, incluyendo el tratamiento de
    `initialText`, la ruta de navegación y cualquier coste de
    renderizado medido.
-   Mantener una cronología verificable de notificaciones, respuestas de
    proveedores y cambios públicos.
-   Desarrollar pruebas automatizadas para presupuestos de tamaño,
    profundidad y tiempo.
-   Evaluar la aplicabilidad del marco fuera de Android sin asumir
    equivalencia entre arquitecturas.

------------------------------------------------------------------------

## 13. Implicaciones arquitectónicas

STA apunta a una debilidad recurrente en sistemas por capas: los
presupuestos suelen definirse localmente, mientras que el riesgo emerge
de la composición de varias etapas.

Una aplicación puede aceptar una entrada dentro de su límite local y,
sin embargo, desencadenar un coste elevado cuando el dato se copia a un
estado jerárquico, se transmite por IPC o se presenta en una superficie
de UI. El componente final puede desconocer el tamaño original, la
profundidad de propagación y las transformaciones previas.

Los sistemas robustos necesitan:

-   **Presupuestos end-to-end:** límites que acompañen al dato a través
    de la cadena.
-   **Límites locales:** cada etapa valida el trabajo que va a realizar.
-   **Control de amplificación:** evitar duplicaciones y anidaciones
    innecesarias.
-   **Degradación segura:** ante exceso de tamaño o coste, descartar,
    truncar o diferir sin bloquear el sistema.
-   **Recuperación independiente del estado fallido:** un dato que
    provoca un fallo no debe convertirse en requisito para volver a
    arrancar.
-   **Observabilidad causal:** métricas que permitan localizar dónde
    crece el estado o el coste.

La contribución de STA no consiste en afirmar que todas estas
condiciones constituyan una única vulnerabilidad, sino en ofrecer un
lenguaje y un procedimiento para estudiar sus interacciones.

------------------------------------------------------------------------

## 14. Conclusión

Structured Text Amplification propone un marco para estudiar cómo
entradas textuales estructuradas pueden comprometer la disponibilidad
cuando atraviesan múltiples capas que no comparten límites ni
estrategias de recuperación coherentes.

La evidencia reunida en STA v6.0 muestra dos familias especialmente
relevantes en Android: la amplificación de estado e IPC (Class A) y el
procesamiento síncrono de texto con bloqueo de interfaz (Class B). Los
casos abarcan aplicaciones, navegadores, bibliotecas compartidas,
componentes del framework y servicios del sistema. Algunos muestran
fallos transitorios; otros describen persistencia y ciclos de caída. La
severidad y la causalidad deben establecerse individualmente.

La defensa eficaz requiere límites antes de cada operación costosa,
control de tamaño y profundidad del estado, procesamiento de UI acotado,
serialización adecuada y mecanismos de recuperación capaces de descartar
datos excesivos. Las mitigaciones deben situarse tanto en las
aplicaciones como en las capas compartidas que reciben y procesan el
contenido.

STA debe entenderse como un modelo arquitectónico contrastable, no como
una etiqueta que sustituya el análisis de causa raíz. Su valor depende
de que cada vector mantenga trazabilidad entre entrada, etapas,
mediciones, fallo y mitigación.

------------------------------------------------------------------------

## 15. Terminología y propuesta de taxonomía

**Nombre del patrón:** Structured Text Amplification (STA)\
**Descripción breve:** propagación no acotada de estado estructurado y
amplificación de recursos a través de capas de procesamiento.\
**Nombre CWE propuesto en la investigación:**
`Unbounded Structured-State Propagation Across Processing Layers`.

La propuesta CWE es una iniciativa de clasificación del investigador, no
una asignación o aprobación oficial. El whitepaper identifica la
referencia `ES2610-0b2ea35d` y el Issue CWE-CAPEC `#228` en el estado de
tramitación allí documentado. El estado debe verificarse antes de cada
publicación externa.

### 15.1 Taxonomía de trabajo

-   **STA-A --- State / IPC Amplification**
    -   A1: SavedState y restauración.
    -   A2: Intent y Binder IPC.
    -   A3: Persistencia de tareas y estado del sistema.
    -   A4: Redistribución entre servicios y componentes privilegiados.
-   **STA-B --- Synchronous Text-Processing Amplification**
    -   B1: Line breaking y medición de texto.
    -   B2: Layout de sugerencias, historial y superficies de
        interacción.
    -   B3: Rutas de renderizado que bloquean el hilo principal.
-   **STA-X --- Contextual propagation extensions**
    -   X1: Deep links y parámetros de navegación.
    -   X2: Otras rutas estructuradas que requieren demostrar
        amplificación antes de clasificarse como A o B.

Esta taxonomía organiza la investigación; no sustituye la taxonomía CWE
ni constituye una clasificación oficial.

------------------------------------------------------------------------

## Referencias y materiales de investigación

### Material primario del proyecto

-   [Repositorio STA ---
    Lostmon](https://github.com/Lostmon/STA-Structured-Text-Amplification)
-   [Blog de investigación --- Lostmon](https://lostmon.blogspot.com)
-   [Structured Text Amplification
    (STA)](https://lostmon.blogspot.com/2026/06/structured-text-amplification-sta.html)
-   [Algorithmic DoS en
    libminikin](https://lostmon.blogspot.com/2026/06/algorithmic-dos-en-libminikinso.html)
-   [libminikin --- 10 años de
    vulnerabilidad](https://lostmon.blogspot.com/2026/07/libminikin-10-anos-de-vulnerabilidad-en_0671440346.html)
-   [Cronología de la investigación
    STA](https://lostmon.blogspot.com/2026/07/timeline-investigacion-sta-estilos.html)

### Fuentes técnicas citadas en el whitepaper

-   [AOSP ---
    Minikin](https://android.googlesource.com/platform/frameworks/minikin/)
-   [Chromium --- commit
    3e62ece6](https://chromium.googlesource.com/chromium/src/+/3e62ece6)
-   [Google Issue Tracker
    40879254](https://issuetracker.google.com/issues/40879254)
-   [Google Issue Tracker
    40268979](https://issuetracker.google.com/issues/40268979)

Las referencias identifican materiales y rutas de investigación. La
relación de cada fuente con un vector concreto debe comprobarse en el
texto técnico y en la versión del código correspondiente.

------------------------------------------------------------------------

## Historial de versiones

  -----------------------------------------------------------------------
  Versión                             Cambios
  ----------------------------------- -----------------------------------
  1.x                                 Primeras observaciones sobre
                                      amplificación estructural, menús
                                      contextuales, Share Intent y estado
                                      de fragmentos.

  2.0                                 Formalización inicial del modelo y
                                      consolidación de mecanismos IPC y
                                      procesamiento de texto.

  **3.0**                             Reescritura a partir del whitepaper
                                      STA v6.0; separación explícita
                                      Class A/Class B; incorporación de
                                      STA-027 como extensión contextual;
                                      metodología probatoria; límites de
                                      causalidad; discrepancia pendiente
                                      del catálogo 32/33; mitigaciones y
                                      validación ampliadas.
  -----------------------------------------------------------------------

------------------------------------------------------------------------

**© 2026 Manuel García Peña (Lostmon).**\
Investigación de seguridad independiente. Las afirmaciones técnicas y
puntuaciones de severidad deben interpretarse conforme a la evidencia y
limitaciones expresadas en cada sección.
