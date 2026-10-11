# CWE New Weakness Submission — Structured Text Amplification (STA)

**Proposed Weakness Name:** Unbounded Structured-State Propagation Across Processing Layers
**Short Name:** STA — Structured Text Amplification
**Submitted by:** Manuel García Peña ("Lostmon") — Independent Security Researcher

---

## 1. Description

Unbounded Structured-State Propagation Across Processing Layers occurs when software accepts or transports structured textual input without enforcing an appropriate security-relevant resource bound, and the input — or state derived from it — is subsequently retained, propagated, materialized, transformed, or serialized across multiple processing layers.

Downstream components may independently process the resulting state without applying limits that account for its origin or cumulative resource impact. Repeated materialization and serialization across independent consumers can cause large inter-process communication (IPC) transactions, excessive resource consumption, transaction failures, incomplete state reconstruction, application failures, or cascading denial of service.

The weakness arises from the interaction between unbounded structured state and resource-constrained processing boundaries. Relevant inputs may include URLs, URIs, deep links, intent data, structured text, serialized payloads, and associated metadata or extras.

The originating component may accept or transport the input without a security-relevant size or complexity bound. The input, or state derived from it, can then become part of persistent or semi-persistent application or framework state. Subsequent components may retrieve that state and construct new representations for task management, lifecycle operations, window management, IPC, user-interface updates, or other consumers.

Each individual component may behave according to its local implementation contract, while the system as a whole fails to constrain the cumulative resource requirements of the propagated state. A representation may be serialized repeatedly or delivered through multiple independent interfaces. The same state, or state derived from the original input, may therefore reach consumers that did not participate in the original validation decision.

When a transaction or processing operation exceeds an implementation-specific resource limit, the result may include `TransactionTooLargeException`, failed IPC delivery, incomplete deserialization or state reconstruction, application termination, repeated component restarts, or cascading denial of service. Recovery mechanisms that repeatedly reconstruct or redistribute the affected state may prolong or intensify the failure.

---

## 2. Defining Characteristics

The weakness is characterized by the combination of:

1. Structured input or derived state that lacks an effective security-relevant resource bound.
2. Propagation of that state across multiple processing layers.
3. Repeated or independent materialization, transformation, or serialization by downstream components.
4. Failure to enforce appropriate resource limits at relevant processing and communication boundaries.
5. Resource exhaustion or cascading failures arising from the interaction between these layers.

The weakness does not require a specific serialization format, programming language, operating system, IPC mechanism, or fixed amplification ratio. It may occur even when no individual component performs an obvious multiplicative expansion. Its defining property is the cross-layer propagation and repeated processing of insufficiently bounded state.

---

## 3. Re-materialization

Re-materialization is the reconstruction or creation of a representation of existing state for a subsequent consumer or processing stage. For example, state originating in a URI or structured input may be incorporated into an Intent or task, later included in task metadata, serialized into an IPC transaction, and then reconstructed or queried by another component.

The same state, or state derived from the original input, may be materialized independently for multiple consumers that did not participate in the original validation decision. This can propagate resource pressure beyond the component that first accepted or transported the input.

Re-materialization is a defining mechanism of this weakness, but its presence alone is not sufficient to establish a vulnerability. The relevant condition is insufficiently bounded state propagation that results in, or can technically substantiate, resource exhaustion or a failure condition.

---

## 4. Relationships to Other CWE Entries

Potential relationships for CWE review:

- **ChildOf** [CWE-400](https://cwe.mitre.org/data/definitions/400.html): Uncontrolled Resource Consumption
- **ChildOf** [CWE-770](https://cwe.mitre.org/data/definitions/770.html): Allocation of Resources Without Limits or Throttling
- **Related to** [CWE-405](https://cwe.mitre.org/data/definitions/405.html): Asymmetric Resource Consumption (Amplification)
- **PeerOf** [CWE-130](https://cwe.mitre.org/data/definitions/130.html): Improper Handling of Length Parameter Inconsistency
- **PeerOf** [CWE-502](https://cwe.mitre.org/data/definitions/502.html): Deserialization of Untrusted Data

These relationships are proposed for evaluation and should not be interpreted as a claim that STA is identical to any of the existing weaknesses.

STA focuses on cross-layer propagation and repeated materialization or serialization of insufficiently bounded structured state. It does not require a measurable byte-amplification ratio, a length-field inconsistency, or unsafe object deserialization.

[CWE-833](https://cwe.mitre.org/data/definitions/833.html) (Deadlock) is not an inherent relationship: the described failure can occur without a demonstrated deadlock.

---

## 5. Security Properties and Consequences

**Confidentiality Impact:**
Not established by the described weakness alone.

**Integrity Impact:**
Not established by the described weakness alone. Recovery or state-management side effects must be demonstrated separately before assigning an integrity impact.

**Availability Impact:**
High, where the weakness causes persistent application failure, repeated component crashes, exhaustion of shared IPC resources, or disruption of system services.

Potential consequences include:
- Denial of service
- Application launch failures or process termination
- Failed IPC transactions
- Incomplete deserialization or state reconstruction
- Repeated component restart loops
- Degradation or failure of shared system-interface components
- Cascading failures across components that consume or reconstruct the affected state

---

## 6. Modes of Introduction

The weakness may arise from architectural or implementation decisions, including:

- Accepting or transporting structured input without an appropriate size or complexity bound.
- Persisting input, or state derived from it, in application or framework state.
- Reusing unbounded state across task, lifecycle, navigation, or window-management operations.
- Serializing state to IPC consumers without accounting for transaction limits.
- Relying on validation performed by an earlier component when later components independently materialize or serialize the state.
- Retrying failed operations or reconstructing state without reducing, bounding, or isolating the problematic data.

The weakness is language-agnostic and can affect operating-system frameworks, applications, browsers, messaging software, and other systems that propagate structured input across processing boundaries.

---

## 7. Affected Technologies and Examples

Potentially affected technologies include:

- Operating-system task and activity frameworks
- Intent and deep-link routing
- IPC and serialized message transport
- Window-management and task-organizer interfaces
- Recent-task and application-lifecycle management
- Browser and application integrations that pass structured state to system components
- Text-processing and rendering paths where structured text is repeatedly measured, shaped, or laid out

Android Binder is one example of an IPC mechanism in which transaction-size limits can make this class of weakness observable. The weakness is not specific to Android or Binder.

---

## 8. Detection and Testing

Detection should examine the full lifecycle of structured input rather than testing only the initial parser or receiving component.

Useful approaches include:

- Instrumenting IPC transaction sizes and identifying repeated large transactions associated with the same input or derived state.
- Tracing structured input through Intent, URI, task, metadata, serialization, and consumer boundaries.
- Testing each entry point and delivery path while holding the underlying test input constant.
- Recording application and system-component crashes, transaction failures, incomplete state reconstruction, and recovery loops.
- Performing static analysis of serialization and materialization paths to identify missing resource bounds.
- Fuzzing structured input across multiple consumers and observing cumulative resource effects.
- Distinguishing correlation from causation by isolating the state fields and transaction paths responsible for observed failures.

A large input, a large transaction, or a crash in isolation is not sufficient to establish STA. Testing should substantiate the propagation mechanism and the resulting resource-exhaustion or failure condition.

---

## 9. Mitigation

Mitigation should apply resource bounds at multiple relevant boundaries:

- Enforce size and, where appropriate, complexity limits when structured input is first accepted.
- Apply independent validation when input-derived state is persisted, retrieved, or materialized for a new consumer.
- Bound fields such as URI data, captured links, extras, metadata, and serialized representations where applicable.
- Avoid repeatedly embedding unbounded state in task, lifecycle, or window-management metadata.
- Enforce serialization and IPC limits before sending transactions, with graceful failure handling.
- Avoid retry loops that repeatedly submit the same oversized state.
- Isolate state and failures between consumers so that one invalid or oversized representation cannot destabilize shared components.
- Add regression tests that exercise multiple delivery paths and downstream consumers, not only the original input handler.
- Ensure recovery paths can discard, truncate safely, or otherwise isolate invalid state without repeatedly reconstructing it.

A single size check may reduce a particular manifestation but may not eliminate the weakness if derived state can be propagated or materialized through other paths. Effective mitigation requires bounds and failure handling at the relevant acceptance, state, materialization, and serialization boundaries.

---

## 10. Observed and Reproduced Android Example

In the Android 16 / HyperOS test environment, structured input delivered through application and deep-link flows was associated with oversized Binder transactions across several framework interfaces. Observed transaction sizes ranged from approximately 365 KB to approximately 1.11 MB in different reproductions.

Observed symptoms included `TransactionTooLargeException`, failed Binder delivery, `BadParcelableException` messages indicating incomplete array retrieval, application launch failures, and repeated SystemUI crashes. In one documented reproduction, a sustained SystemUI crash sequence escalated to a RescueParty-triggered reboot.

A separate IntentResolver reproduction showed an `ACTION_VIEW` flow involving `ResolverActivity` followed by approximately 556 KB transaction failures, Resolver process launch failure/restart, and a SystemUI failure while querying recent tasks. These observations support the involvement of multiple Android-managed processing and IPC layers.

Reproductions have involved different application families and delivery paths, including browser-mediated links, messaging/deep-link handlers, video links, and Android's IntentResolver. These applications are treated as handlers, consumers, or affected components; the observations do not establish that every involved application independently contains the underlying weakness.

The precise contribution of every field to the observed transaction sizes has not been isolated. The evidence supports a cross-layer propagation and resource-boundary failure model, but does not justify attributing all transaction bytes to a single URI, captured-link field, or application component. The observations also do not establish memory corruption or remote code execution.

---

## 11. Distinction Between Entry Points and Root Cause

Different applications or delivery paths may provide independent ways to introduce or transport the same structured input. These paths should not automatically be counted as separate weaknesses.

An application may act as the initial receiver, a deep-link handler, a consumer of derived state, or a point at which the Android framework re-materializes existing state. The fact that a failure becomes visible while one application is active does not, by itself, establish that the application is the root cause.

The architectural question is whether insufficiently bounded state can propagate through multiple processing layers and repeatedly consume constrained resources after the original input-handling decision.

---

## 12. Related Mitigations and References

Related platform mitigations, including selective handling of large payloads in browser or application code, may reduce individual manifestations. Such mitigations should not be treated as proof that every framework-level propagation path has been addressed.

Research materials and references should be supplied with the submission and should identify the relevant public technical reports, reproducible observations, source-code references, and any related platform issue identifiers. Claims should be limited to what the referenced evidence demonstrates.

**Research blog:**
https://lostmon.blogspot.com/

**Additional references:**
- Android platform and framework source code relevant to `Task`, `TaskInfo`, `ActivityTaskManager`, `TaskOrganizer`, and Binder IPC.
- Relevant public Chromium changes concerning large-payload handling, where directly applicable.
- Reproduction logs and technical reports supplied privately to the affected vendors, subject to coordinated-disclosure constraints.

---

## 13. Rationale for a Distinct Weakness

Existing resource-consumption and amplification categories describe important aspects of this problem, but may not clearly express the cross-layer state-propagation mechanism.

STA identifies a class of failures in which insufficiently bounded structured state is accepted or transported, retained or derived into internal state, and then repeatedly materialized or serialized across independent processing layers. The security-relevant condition emerges from the interaction between the propagated state and the resource limits of downstream consumers.

The proposed weakness is intended to help developers and reviewers recognize that local validation or local resource controls may be insufficient when state is reused across multiple framework boundaries. It emphasizes lifecycle tracing, resource bounds at materialization and serialization boundaries, and containment of failures across consumers.

---

**Submitted by:** Manuel García Peña ("Lostmon") — Independent Security Researcher
**Proposed short name:** STA — Structured Text Amplification
**Proposed CWE name:** Unbounded Structured-State Propagation Across Processing Layers
