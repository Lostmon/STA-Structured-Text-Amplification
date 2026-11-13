
<!--
English translation of STA Research Thesis v2.0.
GitHub-ready Markdown.
Original Spanish version remains the authoritative source.
-->

# STRUCTURED TEXT AMPLIFICATION (STA)

## Unbounded Structured-State Propagation and Resource Amplification Across Processing Layers

**English title:** Structured Text Amplification (STA): Unbounded Structured-State Propagation and Resource Amplification Across Processing Layers

**Author:** Manuel García Peña — Lostmon  
**Research:** STA Research  
**Profile:** Independent Security Researcher  
**Official repository:** https://github.com/Lostmon/STA-Structured-Text-Amplification  
**Research blog:** https://lostmon.blogspot.com  
**Classification proposal:** Unbounded Structured-State Propagation Across Processing Layers  
**Research areas:** Android AOSP, Binder IPC, SavedState, libminikin, Chromium, Gecko, text-processing architectures, and artificial intelligence.

**Version:** 2.0 — Consolidated Research Thesis  
**Date:** November 2026  
**Status:** Independent technical research and taxonomic proposal under evaluation.

---

# ABSTRACT

The evolution of modern software architectures has increased dependence on textual structures as a universal mechanism for communication, representation, persistence, and information transfer.

URLs, URIs, Intents, HTML documents, metadata, serialized states, rich content, and inputs intended for language models cross multiple components and processing boundaries. However, the security guarantees applied during the initial acceptance of this data are not always preserved during subsequent transformation, materialization, persistence, and serialization operations.

This research presents **Structured Text Amplification (STA)**, an architectural resource-amplification pattern in which a structured input, or the state derived from it, acquires a computational, memory, or transport cost disproportionate to its propagation across different software layers without sufficiently restrictive resource budgets.

The research identifies two main manifestations in the Android ecosystem:

- **Class A — State and IPC Amplification:** propagation and rematerialization of structured states through persistence, serialization, and inter-process communication mechanisms.
- **Class B — Computational and Rendering Amplification:** disproportionate increase in processing cost during parsing, measurement, glyph shaping, and typesetting operations.

The research corpus brings together 32 documented vectors, with technical evidence from different input and processing paths, including execution traces, build identifiers, resource measurements, code analysis, and experimental reproductions.

The research also incorporates results from different Android versions and manufacturers, as well as analysis of Chromium- and Gecko-based browsers.

A complementary line examines behaviors related to structured-input amplification in artificial intelligence systems, including ingestion interfaces, tokenization, context processing, and generation. These results are studied as an experimental extension of the model, without assuming that all share a root cause or that they automatically constitute the same CWE weakness.

The thesis proposes a containment model based on computational budgets, independent validation at each processing boundary, control of state rematerialization, and safe degradation mechanisms.

Finally, it proposes a specific taxonomy for describing unbounded structured-state propagation, whose purpose is to complement existing classifications of uncontrolled resource consumption.

**Main contribution:** demonstrate that the security of a structured input cannot be evaluated only at the point where it is received. It must also be guaranteed throughout its entire lifecycle and in every component that rematerializes or processes it.

---

# 1. INTRODUCTION

## 1.1. Context

Modern architectures are built through the composition of multiple independent subsystems.

In Android, a URL may begin as content received by a browser, be transformed into an Intent, be incorporated into a task state, be persisted, be serialized via Parcel, and be transmitted to other processes through Binder.

In a browser, the same content may traverse the DOM, selection mechanisms, preview generation, sharing interfaces, and native rendering components.

In artificial intelligence systems, an input may be transformed into internal documents, token sequences, context structures, and representations used by different inference stages.

In all these scenarios, a common characteristic appears: the original datum ceases to be a passive representation and begins to participate in operations whose cost depends on its structure, composition, and subsequent treatment.

STA research starts from a question:

**What happens when an input that appears small or manageable at its origin becomes a disproportionate computational or state load after crossing different layers of the system?**

The working hypothesis is that certain architectures lack cumulative controls capable of limiting the real cost of structured data throughout its entire lifecycle.

## 1.2. Research problem

Traditional validation strategies usually focus on:

- Input length.
- Size of the received object.
- Syntactic validity.
- Parser limits.
- Memory limits of a component.
- Transport mechanism restrictions.

However, these checks may be insufficient when an initially validated datum is transformed into different representations and processed repeatedly by independent consumers.

An input may respect one component's limit and subsequently exceed the resources available in another.

The problem does not necessarily reside in an individual operation, but in the absence of a coherent resource policy across the entire processing chain.

## 1.3. Main hypothesis

The central hypothesis of STA states that:

> The absence of effective limits on the size, complexity, and propagation of structured state allows initially acceptable inputs to generate disproportionate resource consumption in subsequent components, causing local or systemic failures.

The hypothesis is considered supported when the following can be established experimentally:

1. An identifiable source input or state.
2. A sequence of transformations or propagations.
3. One or more relevant processing boundaries.
4. An increase in resources or cost attributable to that propagation.
5. A reproducible or technically demonstrable failure condition.

---

# 2. FORMAL DEFINITION OF STRUCTURED TEXT AMPLIFICATION

## 2.1. Definition

**Structured Text Amplification (STA)** is an architectural resource-amplification pattern in which a structured textual input, or the state derived from it, is accepted, transported, or persisted without sufficiently effective resource limits and is subsequently propagated, transformed, rematerialized, or serialized across multiple processing layers.

As a consequence, one or more subsequent components may experience disproportionate consumption of memory, CPU, transport capacity, or execution time, to the point of exceeding their operational limits or compromising system stability.

STA does not require:

- A specific serialization format.
- A specific operating system.
- A specific IPC mechanism.
- Exponential expansion.
- A fixed relationship between input bytes and resources consumed.
- A memory-corruption vulnerability.

Its differential characteristic is the interaction between insufficiently bounded structured state, propagation across layers, and resource consumption in subsequent components.

## 2.2. Mathematical model

Let an initial structured input be:

$$
X_0
$$

Each component of the architecture applies a transformation:

$$
X_{i+1}=T_i(X_i)
$$

Where:

- \(X_i\) represents the state at a given layer.
- \(T_i\) represents a transformation, serialization, parsing, persistence, or materialization operation.
- \(X_{i+1}\) represents the new representation of the state.

The cost of processing each representation is defined as:

$$
C_i=f_i(X_i,S_i)
$$

Where \(S_i\) represents the component's internal state and execution conditions.

The total cost of the chain can be expressed as:

$$
C_{\text{total}}=\sum_{i=0}^{n} C_i+\sum_{j=1}^{m} R_j
$$

Where \(R_j\) represents the additional cost associated with rematerializations or independent deliveries to subsequent consumers.

For a real architecture, cost may also depend on concurrency, object reuse, shared storage, and processing frequency.

We define the resource amplification factor:

$$
A_R=\frac{R_{\text{observed}}}{R_{\text{reference}}}
$$

Where both values must be measured under comparable experimental conditions.

The factor may be calculated separately for:

- Memory.
- CPU.
- Execution time.
- Serialized size.
- Number of operations.
- Inference cost.

There is no universal STA amplification factor. The comparison must specify which resource is measured, at which layer, and with respect to which reference.

## 2.3. Concept of computational budget

Each processing boundary should maintain an explicit budget:

$$
B_i=(M_i,T_i,S_i,F_i)
$$

Where:

- \(M_i\): maximum allowed memory.
- \(T_i\): maximum processing time.
- \(S_i\): maximum state or representation size.
- \(F_i\): allowed frequency or number of operations.

An operation is admissible when it remains within the limits defined for the component and the execution context.

The fundamental property is that validation at one layer must not be considered unlimited authorization for all subsequent layers.

## 2.4. State rematerialization

Rematerialization is one of the central mechanisms of the STA model.

It occurs when a component reconstructs a representation of an existing state in order to deliver it to another consumer.

Example:

```

URI → Intent → TaskInfo → Parcel → Binder

```

The same logical content may appear in different representations and be serialized independently for multiple consumers.

Rematerialization is not in itself a vulnerability. It becomes a risk factor when there are no effective limits on the propagated state and its processing ends up causing resource exhaustion or demonstrable failures.

---

# 3. FUNDAMENTAL PRINCIPLES OF THE STA MODEL

## 3.1. Local validation versus systemic security

A component may correctly fulfill its local contract and still participate in a chain that ultimately compromises system stability.

Therefore:

$$
\text{Local validation} \neq \text{Global security}
$$

Security must be evaluated across the complete data lifecycle.

## 3.2. Asymmetry between representation and cost

The byte size of an input does not necessarily determine the cost of processing it.

Two strings of similar length may generate different costs depending on:

- Encoding.
- Unicode composition.
- Associated metadata.
- Number of segments.
- Algorithmic complexity.
- Number of transformations.
- Structure of derived objects.

In Android, for example, a string represented in UTF-8 may occupy a different amount of memory when converted into a UTF-16 representation.

In addition, Spanned objects may incorporate metadata associated with styles and ranges.

Encoding conversion by itself does not imply amplification by orders of magnitude. The additional cost must be measured including the representations and structures actually built.

## 3.3. The resource boundary as a trust boundary

Each transition between components must be considered a new validation opportunity.

A consumer should not assume that the previous component has guaranteed all the limits it needs to process the datum.

## 3.4. Recovery is also part of the attack surface

An initial failure may become a persistent incident when recovery mechanisms repeatedly reconstruct or redistribute the same problematic state.

Recovery must be able to discard, isolate, or degrade the state that causes the failure.

---

# 4. TAXONOMY OF STA MANIFESTATIONS

The research distinguishes two main families in Android and other complementary lines of study.

## 4.1. Class A — State and IPC Amplification

**Amplification of state, persistence, and inter-process communication.**

Mechanism:

- Structured input.
- Incorporation into internal objects.
- Persistence or temporary storage.
- Rematerialization.
- Serialization.
- Transport via IPC.
- Failure due to resource limits.

Components investigated:

- Intent.
- Bundle.
- SavedState.
- FragmentManager.
- TaskInfo.
- ActivityTaskManagerService.
- TaskOrganizerController.
- TransitionController.
- RecentTasksController.
- Binder IPC.
- SystemUI.

Representative vectors:

STA-001, STA-003, STA-005, STA-006/007, STA-009/010/018, and STA-012.

The observable effect may include TransactionTooLargeException, restoration failures, process termination, repeated restarts, and cascading denial of service.

## 4.2. Class B — Computational and Rendering Amplification

**Algorithmic amplification during text processing.**

Mechanism:

- Structured textual input.
- Parsing and segmentation.
- Unicode and directionality analysis.
- Line-break resolution.
- Glyph shaping.
- Measurement and typesetting.
- UI thread blocking.

Components investigated:

- TextView.
- StaticLayout.
- LineBreaker.
- Minikin.
- HarfBuzz.
- Paint.measureText().
- Chromium.
- Gecko.

Representative vectors:

STA-017, STA-020, and STA-031.

In these scenarios, resource consumption may be related to composition complexity and the algorithmic paths used.

An O(n²) complexity should not be attributed to all Minikin or HarfBuzz operations. Complexity must be justified through code analysis, identification of the specific path, and reproducible measurements.

## 4.3. Class C — Interface and Ingestion Amplification

**Experimental line in artificial intelligence ingestion interfaces.**

This category studies the transformations that occur before content reaches the model.

Among the phenomena investigated:

- Conversion of text into internal documents.
- Fragmentation and expansion of content.
- Duplication of segments.
- Format transformations.
- Mismatches between input size and the representation delivered to the model.

The documented Copilot case describes an input of 61,560 characters converted into an internal representation of approximately 1,002,682 characters, with an observed ratio close to 16.28.

The cause identified in that experiment lies in the ingestion interface, not necessarily in the language model.

## 4.4. Class D — Model and Context Processing Amplification

**Experimental line on context processing and inference.**

Phenomena such as the following are investigated:

- Disproportionate increase in processing cost.
- Sequence fragmentation.
- Repetitive low-entropy inputs.
- Anomalous prolongation of operations.
- Loss of task convergence.
- Parsing and generation errors.

Published experiments include tests with DeepSeek, Grok, Gemini, Copilot, Leo, and Qwen VL, with different results and symptoms.

This category should be understood as an experimental extension of the general model, not as an automatic equivalence with Android Class A and Class B mechanisms.

Tokenization, attention cost, and generation are different mechanisms. A token amplification ratio does not by itself demonstrate a state-propagation defect.

---

# 5. ANDROID FORENSIC RESEARCH

## 5.1. Native text engine: libminikin.so

Android typographic processing incorporates different analysis, composition, and rendering operations.

Among them:

- BiDi directionality resolution.
- Text segmentation.
- Line breaking.
- Glyph shaping.
- Text measurement.
- HarfBuzz integration.

The research documents scenarios in which certain textual compositions cause a disproportionate increase in processing time.

The relevant characteristic is that apparently valid inputs can reach costly native paths before the system applies a sufficiently effective limitation.

In certain reproduced cases, synchronous processing blocks the main thread and causes an ANR.

### 5.1.1. Sixth entry point

One of the research expansions identifies the path:

```

Paint.measureText() → text processing → HarfBuzz shaping

```

The case documented in Google Docs extends the research surface beyond traditional line-breaking mechanisms.

This demonstrates the importance of studying the different entry points into native engines, instead of limiting analysis to a single presentation API.

### 5.1.2. Architectural consequence

Mitigations implemented exclusively in individual applications may reduce exposure, but they do not necessarily protect other consumers that use the same native libraries.

For this reason, the research considers a policy of limits and complexity control at the platform layers necessary.

## 5.2. Binder IPC and state serialization

Binder uses an inter-process transport mechanism with capacity restrictions.

The reference limit commonly associated with transactions is approximately 1 MiB, but it should not be interpreted as an independent and guaranteed budget for each object or transaction: shared-buffer conditions and other system factors are involved.

The research studies how structures derived from external inputs can reach serialization paths without a sufficiently early evaluation of their real size.

Relevant operations include:

- writeToParcel().
- Intent serialization.
- Bundle serialization.
- SavedState restoration.
- TaskInfo transfer.
- Queries to task-management services.

### 5.2.1. Propagation chain

An initial representation may traverse several stages:

```

External input → Intent → internal state → TaskInfo → Parcel → Binder → consumer

```

The problem is aggravated when the same state is rematerialized for independent consumers.

### 5.2.2. Exception and recovery

TransactionTooLargeException may cause a transport or restoration operation to fail.

When the receiving component is a shared or persistent service, the impact may extend to other processes.

The existence of an exception does not by itself demonstrate a systemic crash loop. The latter requires documenting the failure sequence, restarts, dependencies, and recovery behavior.

---

# 6. VECTORS AND CROSS-PLATFORM EVIDENCE

## 6.1. Research corpus

STA research brings together 32 documented vectors in the consolidated corpus.

The repository classifies cases mainly into the IPC/SavedState and native text-processing families.

Among the vectors with notable technical documentation:

| Vector | Surface | Manifestation |
|---|---|---|
| STA-001 | Context menu / URI | Contextual processing and failure |
| STA-003 | Share Intent / Binder | TransactionTooLargeException |
| STA-005 | WhatsApp / SavedState | State amplification |
| STA-006/007 | Transfer and documents | ANR and IPC failures |
| STA-009/010/018 | Print Preview | Cascading system failures |
| STA-012 | Deep links / SavedState | Persistent failure |
| STA-017 | libminikin | ANR during processing |
| STA-020 | Omnibox | Interface blocking |
| STA-031 | Paint.measureText | Shaping path |

The identifiers represent documented vectors or input paths, not necessarily independent CWE weaknesses.

## 6.2. Reproducibility across manufacturers

The research includes reproductions on devices from different manufacturers and Android generations.

The test set provided by the researcher comprises:

- Xiaomi Redmi Note 14 5G, Android 16, HyperOS 3.0.
- Samsung Galaxy A13 5G, Android 14, One UI 6.1.
- OPPO A6 5G, Android 15, ColorOS 15.

The existence of manifestations across different manufacturers allows investigating whether the behavior originates from shared Android components or from manufacturer-specific implementations.

However, reproduction on three OEMs does not automatically demonstrate a common root cause. To establish it, traces, execution paths, AOSP versions, and affected components must be correlated.

## 6.3. TaskInfo and SystemUI case

One of the most relevant research lines examines the propagation of state associated with Android tasks.

The following are especially studied:

- Task.fillTaskInfo().
- TaskInfo.
- TaskInfo.writeToParcel().
- ActivityTaskManagerService.
- TaskOrganizerController.
- TransitionController.
- RecentTasksController.
- SystemUI.

The analysis provided describes transactions of approximately 365 KB to 1.11 MB in different reproductions and failures associated with several Binder interfaces.

In one documented reproduction, a sustained sequence of SystemUI failures was observed that eventually activated device recovery mechanisms.

The research also documents a reproduction in which RescueParty requested a restart.

This case must be analyzed as an impact chain and not as a universal consequence of any excessive transaction.

### 6.3.1. Difference between symptom and root cause

The SystemUI crash constitutes the final symptom.

STA research must locate the earlier operations that allow a state of external origin to reach critical consumers without sufficient controls.

It must also distinguish which fields effectively contribute to transaction size. All volume must not be attributed to Intent.data or capturedLink without an isolated measurement demonstrating it.

## 6.4. Convergence of upstream mitigations

During 2026, independent changes in AOSP, AndroidX, and Chromium related to the containment of states and bulky payloads have been identified.

Examples included in the repository:

- AOSP InputMethod: a438ce17.
- AndroidX Credential Manager: 393e20ae.
- Chromium PDF Selection: 84b615a0.
- AndroidX NotificationCompat: 90ffa6a7.
- AndroidX PdfView: 8882927e.
- Chromium Oversized Clipboard: 4751a769.
- Chromium Native Messaging: f5c51669.
- Chromium TLE Telemetry: 931ee1ab.

The changes represent different strategies:

**Constrain:** impose limits before the state grows.

**Redirect:** move bulky payloads to alternative mechanisms.

**Replace:** replace complete data with identifiers or references.

**Observe:** measure and record operations that exceed limits.

This convergence is compatible with the STA model and demonstrates the relevance of state-containment strategies. It should not be presented as official confirmation that the parties responsible for those changes recognize a single STA vulnerability.

---

# 7. EXTENSION OF THE MODEL TO ARTIFICIAL INTELLIGENCE

## 7.1. Motivation

Artificial intelligence systems incorporate information ingestion and transformation pipelines that present architectural similarities with other multilayer systems.

Text received by an assistant may traverse:

1. Input interface.
2. Parser.
3. Normalization.
4. Conversion to internal documents.
5. Tokenization.
6. Context construction.
7. Inference.
8. Generation.
9. Result rendering.

Each stage introduces its own costs and representations.

The research explores whether certain structured inputs can cause unforeseen amplifications during these operations.

## 7.2. Ingestion experiments

Published experiments document significant differences between the original input and the representations built by certain interfaces.

The Copilot case is an example of amplification in the document-ingestion layer.

A distinction must be made between:

- Original text size.
- Internal representation size.
- Number of tokens.
- Inference time.
- Generation cost.

These metrics are not interchangeable.

## 7.3. Model behavior

Tests performed with different assistants have observed symptoms such as:

- Excessively long responses.
- Prolonged processing.
- Parsing errors.
- Fragmentation.
- Loss of task convergence.
- Generation of irrelevant content.

In particular, published experiments describe different behaviors among DeepSeek, Grok, Gemini, Copilot, Leo, and Qwen VL.

It should not be stated that all use the same attention architecture, that all present effective quadratic complexity, or that all share an identical vulnerability.

The research must treat these differences as experimental results of each implementation.

## 7.4. Context security and Instruction Drift

Complementary research on Gemini studies changes in instruction fidelity and generation of insecure code under certain input conditions.

A distinction must be established:

- Resource amplification belongs to the STA domain.
- Instruction Drift belongs to the domain of behavior and context security.
- Generation of insecure code constitutes an additional security outcome.
- Exploitation of a vulnerability in the generated code requires independent validation in the consumer system.

The existence of one behavior does not automatically demonstrate the others.

## 7.5. Limits of attribution

The research has also explored risks of hidden content, Markdown rendering, external image requests, and possible interactions with memory systems.

These scenarios must be documented as independent lines of research in interface security and privacy.

An external HTTP request, an instruction injection, or a memory modification does not by itself constitute STA.

To integrate them into the model, a causal relationship with unbounded propagation and the resource amplification defined in this thesis would need to be demonstrated.

---

# 8. CWE TAXONOMY: PROPOSAL FOR A NEW WEAKNESS

## 8.1. Limitations of existing classifications

Current categories describe important aspects of the problem, but they do not always express the complete architectural mechanism.

**CWE-400 — Uncontrolled Resource Consumption**

Describes uncontrolled resource exhaustion, but does not specifically characterize state propagation and rematerialization across layers.

**CWE-770 — Allocation of Resources Without Limits or Throttling**

Describes the absence of limits on resource allocation, but not necessarily the lifecycle of structured state.

**CWE-405 — Asymmetric Resource Consumption (Amplification)**

Is related to asymmetric amplification of resource consumption.

**CWE-130 — Improper Handling of Length Parameter Inconsistency**

Describes inconsistencies in length parameters.

**CWE-502 — Deserialization of Untrusted Data**

Describes problems related to deserialization of untrusted data.

These categories may describe mechanisms or consequences present in certain STA manifestations, but they should not be considered equivalent.

## 8.2. Proposed name

**Unbounded Structured-State Propagation Across Processing Layers**

Short name: STA — Structured Text Amplification.

## 8.3. Proposed definition

A weakness in which software accepts or transports structured textual inputs without imposing adequate resource limits and subsequently preserves, propagates, transforms, rematerializes, or serializes the state derived from those inputs across multiple processing layers.

Subsequent components may process that state without applying limits that take into account its origin or its cumulative impact on resources.

Repetition of these operations may cause excessive IPC transactions, disproportionate resource consumption, transport failures, incomplete state reconstruction, process termination, or cascading denial of service.

## 8.4. Proposed relationships

| Relationship | CWE | Justification |
|---|---|---|
| ChildOf | CWE-400 | Uncontrolled resource consumption |
| ChildOf | CWE-770 | Absence of allocation limits |
| Related to | CWE-405 | Asymmetric amplification |
| PeerOf | CWE-130 | Length management |
| PeerOf | CWE-502 | Deserialization of untrusted data |

These relationships are proposals for CWE evaluation, not formally assigned relationships.

CWE-833 is not considered an inherent relationship: an STA manifestation may produce resource exhaustion and failures without a demonstrated deadlock.

## 8.5. Identification criteria

To identify a possible STA weakness, the following must be jointly demonstrated:

1. A structured input or derived state that is insufficiently bounded.
2. Its propagation across multiple layers.
3. One or more transformation, materialization, or serialization operations.
4. The absence of adequate limits at relevant boundaries.
5. Resource exhaustion or a failure condition attributable to that interaction.

A large input, a large transaction, or an isolated crash is not sufficient to classify a case as STA.

---

# 9. MITIGATION ARCHITECTURE

Mitigating STA requires a defense-in-depth approach.

It is not enough to fix the entry point where the failure manifests if other components can reconstruct and process the same state.

## 9.1. Resource budgets per boundary

Each transition must define independent limits for:

- Memory.
- Representation size.
- Processing complexity.
- Execution time.
- Operation frequency.

Limits must adapt to the characteristics of the component and the threat model.

## 9.2. Validation before transformation

Data must be validated before entering operations that may multiply its cost.

Examples:

- Before complex parsing.
- Before layout.
- Before shaping.
- Before serialization.
- Before persistence.
- Before an IPC transaction.

Validation must consider both size and composition when the operation depends on both properties.

## 9.3. Secure serialization in Android

The following is recommended:

1. Estimate the cost of the actual serialized representation.
2. Apply limits before sending transactions.
3. Avoid storing complete documents in SavedState.
4. Replace bulky payloads with identifiers.
5. Use references to persistent storage.
6. Use shared-memory transfer or alternative mechanisms when appropriate.
7. Implement controlled failures.
8. Avoid retry loops that reproduce the same problematic state.

### 9.3.1. TaskInfo and Task.fillTaskInfo()

The research proposes studying and limiting the variable-size fields incorporated into TaskInfo, including Intent.data and capturedLink.

The intervention point must be evaluated together with downstream consumers and serialization paths.

The solution should not be limited to arbitrarily truncating a string: it must preserve the semantic coherence of the state or replace bulky content with a secure reference.

## 9.4. Native text engine protection

The following is proposed:

- Early input limits.
- Complexity budgets.
- Incremental processing.
- Time limits.
- Avoid costly operations on the UI thread.
- Degradation to a safe representation when feasible.
- Regression testing with different Unicode compositions.

Indiscriminately removing legitimate Unicode characters, including ZWJ, combining marks, or bidirectional controls, is not recommended.

Protection must balance security, accessibility, and linguistic correctness.

## 9.5. Isolation and protection of AI interfaces

For experimental AI lines, complementary measures are proposed:

- Limit the size and complexity of ingested documents.
- Avoid duplications during conversion to internal documents.
- Measure the relationship between original input and effective representation.
- Apply token limits and inference budgets.
- Separate external data from control instructions.
- Treat external content as untrusted data.
- Require authorization for sensitive persistent operations.
- Control external resource requests generated during rendering.

These measures are complementary security controls and do not replace the need to demonstrate the STA mechanism in each implementation.

---

# 10. EXPERIMENTAL AND FORENSIC METHODOLOGY

The validity of the research depends on each claim being relatable to observable evidence.

## 10.1. Identification of the entry point

Document:

- Origin of the content.
- Original format.
- Initial size.
- Composition.
- Operation that introduces the datum.
- Interaction required to trigger the behavior.

## 10.2. Propagation tracing

Follow the datum across the different layers.

Identify:

- Participating components.
- Invoked functions.
- Transformations.
- Intermediate representations.
- Serialization boundaries.
- Independent consumers.

## 10.3. Instrumentation

When possible, record:

- Input size.
- Size of each representation.
- Memory used.
- CPU.
- Execution time.
- Transaction size.
- Number of materializations.
- Stack traces.
- Build identifiers.
- System versions.

## 10.4. Reproducibility

Each experiment must indicate:

- Device model.
- Manufacturer.
- Android version.
- Application version.
- Build ID.
- Initial conditions.
- Reproduction procedure.
- Expected result.
- Observed result.
- Number of repetitions.
- Attached evidence.

## 10.5. Separation of evidence and attribution

Four levels are established:

**Level 1 — Observation:** an anomalous behavior is recorded.

**Level 2 — Reproduction:** the behavior is reproduced under documented conditions.

**Level 3 — Architectural correlation:** a chain of components and transformations compatible with the failure is identified.

**Level 4 — Causal attribution:** the responsible mechanism is isolated through instrumentation, code analysis, or controlled experiments.

A root cause must not be attributed solely based on the temporal coincidence of two failures.

## 10.6. Responsible disclosure

The research follows a coordinated disclosure policy.

Reports addressed to manufacturers and vendors must include sufficient evidence to facilitate their analysis, avoiding the unnecessary publication of operational payloads that could facilitate exploitation.

Public materials must differentiate between:

- Confirmed vulnerabilities.
- Independent reproductions.
- Reported issues.
- Partial mitigations.
- Hypotheses still pending validation.

---

# 11. LIMITATIONS AND SCOPE

This thesis recognizes several limitations.

First: the 32 documented vectors do not necessarily represent 32 independent root causes. Some are different input paths toward shared mechanisms.

Second: the presence of the same behavior on several devices does not by itself demonstrate that all use exactly the same vulnerable implementation.

Third: amplification measurements depend on execution conditions, versions, components, and available resources.

Fourth: behaviors observed in LLMs must not be automatically extrapolated between vendors or confused with memory defects or code-execution vulnerabilities.

Fifth: identified upstream changes must be analyzed individually. Their existence does not by itself certify the correction or confirmation of all STA vectors.

Sixth: impacts on integrity and confidentiality require specific tests. The central definition of STA focuses on propagation and resource exhaustion; it does not presuppose access to private information, modification of persistent memory, or code execution.

These limitations are part of the research's methodological rigor and allow precise delimitation of what has been demonstrated and what remains under study.

---

# 12. RESEARCH CONTRIBUTIONS

STA research provides the following contributions:

**1. Formalization of an architectural pattern.**

It identifies unbounded propagation of structured state as a source of disproportionate resource consumption.

**2. Multilayer analysis model.**

It establishes rematerialization, serialization, and processing boundaries as central elements for identifying the problem.

**3. Technical corpus of vectors.**

It brings together 32 documented vectors, with evidence from different input and processing surfaces.

**4. Android framework research.**

It relates IPC, SavedState, TaskInfo, Binder, and native text-processing paths.

**5. Cross-platform research.**

It extends analysis to browsers and artificial intelligence systems through differentiated experimental lines.

**6. Taxonomy proposal.**

It formulates a new weakness category to facilitate recognition of state-propagation failures across layers.

**7. Mitigation architecture.**

It proposes resource budgets, per-boundary validation, controlled rematerialization, and safe recovery.

**8. Research tools.**

The repository incorporates documentation and analysis tools, including STA Lab, Android Weight Analyzer, Pattern Generator, and Pattern Detector.

---

# 13. CONCLUSIONS

Structured Text Amplification raises a fundamental question about the security of modern architectures: a datum does not necessarily preserve the same cost or the same properties when it crosses different components.

The research demonstrates, through different vectors and documented scenarios, that structured inputs can acquire processing and transport costs that are not sufficiently controlled by initial validation.

In Android, the investigated manifestations cover two especially relevant surfaces:

- The propagation and serialization of state through IPC and persistence mechanisms.
- Algorithmic text processing in shared native components.

Both share an architectural concern: the absence of adequate limits during certain operations after the entry point.

Research on browsers and artificial intelligence broadens the field of observation and raises new questions about the ingestion, transformation, and processing of structured information.

However, the main value of STA is not in claiming that all text failures are the same, but in providing a model that allows identifying when the propagation of an insufficiently bounded state becomes a resource-exhaustion condition.

Security cannot depend exclusively on the first component correctly validating an input. Every component that receives, transforms, preserves, or rematerializes it must assume its own responsibility for the resources it consumes.

Resilience must be designed for the entire chain.

**The datum that enters through one door need not be the same one that reaches the next. Nor does the cost of processing it.**

This is the architectural premise that gives rise to Structured Text Amplification.

---

# 14. REFERENCES AND RESEARCH MATERIALS

## Main repository

Lostmon — STA Structured Text Amplification:

https://github.com/Lostmon/STA-Structured-Text-Amplification

## Theoretical documentation

- STA Theoretical Framework.
- Theoretical framework of Structured Text Amplification.
- Whitepaper v6: Resilience Gaps in Android IPC, SavedState and Text Layout.
- README and vector catalog.
- Mitigations for Developers.

## Taxonomic proposal

CWE New Weakness Submission — Structured Text Amplification.

File:

CWE_New_Weakness_Submission.md

## Research and timeline

- Structured Text Amplification — Conceptual framework.
- Algorithmic DoS in libminikin.so.
- libminikin: 10 years of vulnerability in the Android core.
- STA research timeline.
- STA — UTF-16 Serialization Density Experiment.
- Upstream Android and Chromium Changes in 2026.
- Unbounded TaskInfo serialization in ActivityTaskManagerService.
- STA in LLMs.
- Structured Text Amplification in Gemini.

## Official blog

https://lostmon.blogspot.com

## External technical references

- Android Open Source Project: https://source.android.com/
- AOSP Minikin: https://android.googlesource.com/platform/frameworks/minikin/
- Chromium: https://chromium.googlesource.com/chromium/src/
- MITRE CWE: https://cwe.mitre.org/
- HarfBuzz: https://harfbuzz.github.io/

---

**STA RESEARCH**

**Manuel García Peña — Lostmon**

Independent Security Researcher

Independent research on architectural security, structured-state propagation, and multilayer system resilience.

November 2026.
