# Editorial Synthesis - Chapter 3

## Status

This synthesis is complete and binding for Wave 2 unless the user requests a
change. It consolidates the four first-wave reports and resolves their
differences against the audited implementation baseline.

The Chapter 3 writer must treat this file as the primary editorial authority.
The first-wave reports remain evidence dossiers: they may clarify a claim, but
they must not override a decision recorded here.

## Audited Baseline

The implementation baseline for Chapter 3 is:

- repository: `demo_project`;
- branch inspected: `feature/composite-collective-correlation`;
- committed baseline: `163ad72`;
- parent correlation feature: `35103ea`;
- target: Rocky Linux 8.10 on
  `4.18.0-553.137.1.el8_10.x86_64`, x86-64.

The following working-tree changes are excluded from the stable design:

- the networking support-probe reclassification in
  `pkg/ebpf/probes/probes.go`;
- the experimental `all-events` benchmark changes;
- the untracked `rules/policies/test_net.yaml`.

They currently break registry or policy-inventory tests and must not be used as
Chapter 3 evidence. The stable registry model keeps support probes internal
unless they are promoted together with a public event ID and decoder schema.

## Role Of Chapter 3

Chapter 3 translates the problem established in Chapter 1 and the foundations
and related work established in Chapter 2 into requirements and architectural
decisions.

It must fulfil the exact promise made in Chapter 1:

- define the target environment;
- state functional and operational requirements;
- identify architectural components;
- explain the end-to-end event flow;
- define the boundary between kernel-space collection and user-space
  processing.

The chapter answers **what the system must do**, **which constraints shape it**,
and **why responsibilities are divided in this way**. Chapter 4 answers how
those decisions are implemented in concrete C and Go code. Chapter 5 provides
functional and performance evidence.

## Central Architectural Argument

The chapter must develop one continuous argument:

1. The enterprise-kernel target constrains available hooks, helpers,
   attachment mechanisms, verifier behaviour and transport choices.
2. These constraints favour bounded eBPF programs focused on observation,
   minimal extraction, optional early filtering and serialisation.
3. A Go runtime manages the more changeable responsibilities: lifecycle,
   decoding, normalisation, policy evaluation, detection, alert generation,
   presentation and operational diagnostics.
4. An explicit producer-consumer event contract connects the two domains.
5. Public logical events are resolved into the physical probes and internal
   dependencies required to produce them.
6. Policies constrain the monitored event domain, while independently loaded
   detectors evaluate admitted normalised events.
7. Point detectors operate on one event; collective detectors retain bounded
   local evidence for short ordered sequences.
8. Alerts preserve matching evidence and may carry MITRE ATT&CK metadata.
9. The result is an observation and local detection prototype, not an
   enforcement engine, cluster-wide correlator or general CEP system.

This argument should connect the chapter sections. They must not read as seven
independent subsystem descriptions.

## Canonical Structure And Narrative Plan

### 3.1 Design Goals and Constraints

Open by connecting the Chapter 1 problem statement to the concrete design
environment.

#### 3.1.1 Target Environment and Compatibility Assumptions

Explain:

- the concrete Rocky Linux and kernel target;
- why an enterprise kernel with vendor backports cannot be characterised only
  by its upstream version;
- that BTF and CO-RE reduce coupling to structure layouts;
- that CO-RE cannot provide missing hooks, helpers, program types, BTF,
  attachment mechanisms, permissions or verifier features;
- that target validation remains necessary.

Keep architecture-specific workarounds, kernel structure flavours, register
access and individual hook availability for Chapter 4.

#### 3.1.2 Functional Requirements

Use a compact table based on the approved requirement groups below. Avoid
reproducing the full thirteen-item agent audit.

| ID | Binding functional requirement |
|---|---|
| FR-01 | Collect a selectable set of process- and security-relevant host events. |
| FR-02 | Resolve selected logical events into the required producer and support probes, excluding unrelated registered programs from loading and attachment. |
| FR-03 | Transfer records through the implemented perf-buffer channel and decode them into normalised typed events through an explicit shared contract. |
| FR-04 | Load configurable policies that constrain the monitored event scope and may narrow attachment when their event allowlists are exact. |
| FR-05 | Load detectors independently from policies and evaluate deterministic point conditions or bounded ordered collective sequences. |
| FR-06 | Produce alerts that preserve the evidence responsible for a match and propagate available MITRE ATT&CK metadata. |
| FR-07 | Keep raw event output, alert output and operational diagnostics as separate paths. |
| FR-08 | Manage startup, attachment, event consumption, observable failures and orderly resource cleanup. |

The optional exact-UID kernel guard belongs under FR-02 as an early reduction
mechanism, not as a separate general policy engine.

Container packaging is implemented build support but is not a central Chapter
3 functional requirement. Its concrete design belongs in Chapter 4, and
Kubernetes deployment is not evaluated thesis work.

#### 3.1.3 Non-Functional and Operational Requirements

Use a second compact table:

| ID | Binding non-functional or operational requirement |
|---|---|
| NFR-01 | Operate on the identified enterprise-kernel target with explicit compatibility assumptions and target validation. |
| NFR-02 | Keep kernel execution and data extraction bounded and compatible with the target verifier. |
| NFR-03 | Maintain a consistent kernel-to-user-space event contract and fail visibly on unknown or malformed records. |
| NFR-04 | Keep detection deterministic, configuration-driven and separate from event presentation. |
| NFR-05 | Bound incomplete collective state through short windows and a per-detector key limit. |
| NFR-06 | Expose transport loss, decoding failures, detector failures and setup errors through appropriate diagnostic paths. |
| NFR-07 | Evaluate a steady-state user-space CPU objective below five per cent of one core for explicitly defined profiles. |

NFR-07 is an evaluation objective, not an unconditional property. Do not put
benchmark numbers or PASS/FAIL conclusions in Chapter 3.

### 3.2 Overall System Architecture

#### 3.2.1 Kernel-Space and User-Space Responsibilities

State the responsibility boundary clearly:

- kernel space observes selected operations, captures bounded context,
  performs optional exact-UID rejection, constructs logical records and
  serialises them;
- Go user space validates configuration, manages libbpfgo lifecycle, resolves
  probe dependencies, receives records, decodes and normalises events, applies
  policy scope, dispatches detectors and produces outputs.

This division is not absolute. Attach-time selection is controlled by user
space, while limited kernel state may join entry and exit observations into one
logical event. Nevertheless, there is no complete kernel-resident policy or
detector engine.

#### 3.2.2 End-to-End Event Lifecycle

Present two related flows.

Control path:

```text
configuration
  -> policy and detector validation
  -> effective event selection
  -> probe and dependency resolution
  -> selective object loading and attachment
```

Data path:

```text
kernel hook
  -> eBPF capture and serialisation
  -> perf buffer
  -> decoder and event registry
  -> configured event/command filters
  -> policy admission
  -> optional raw event output
  -> detector dispatch
  -> optional alert output
```

Operational logging is a side channel for lifecycle, attachment, transport,
loss and decoding diagnostics. It does not serialise raw events or alerts.

The implemented combined-output order is raw event output followed by detector
evaluation and alert output. Chapter 3 need not emphasise this ordering unless
it is required to explain the diagram, but it must not show detector evaluation
before every raw event output as though that were the current implementation.

### 3.3 Kernel-Space Collection Design

#### 3.3.1 Hook Selection and Logical Events

Explain hook choice using four criteria:

- semantic position of the observation;
- need for an operation result;
- availability on the concrete target;
- expected cost and robustness.

Distinguish:

- physical hook;
- eBPF program;
- internal support dependency;
- public logical event.

These concepts are not one-to-one. A logical event may require entry and exit
programs or another support probe, while only the resulting logical event is
exposed to users and the decoder.

Do not enumerate the complete hook catalogue. Use at most one generic example
of paired observations when needed.

#### 3.3.2 Early Event Selection and Filtering

Describe:

- logical event selection before object loading;
- dependency expansion through the probe registry;
- disabling autoload for unrelated registered programs;
- attaching only selected programs and required dependencies;
- the optional exact-UID guard before expensive event construction.

Qualify the UID guard as one minimal early filter. It is not a complete
kernel-side policy system and does not replace user-space policy evaluation.

### 3.4 Event Contract and Transport Design

#### 3.4.1 Stable Binary Event Contract

Explain conceptually:

- a common event context followed by indexed typed arguments;
- shared event identifiers and schemas on the C and Go sides;
- registry-based interpretation and normalisation;
- the contract is compact rather than fully self-describing;
- unknown IDs or malformed records are observable contract failures and are
  dropped before policy and detector evaluation.

Do not include:

- the exact 136-byte layout;
- byte offsets;
- argument buffer size;
- numeric event or type identifiers;
- decoder branch details;
- the long-record defect and correction.

These belong in Chapter 4.

#### 3.4.2 Perf-Buffer Transport and Failure Boundaries

State that the operational transport is a perf buffer because it is available
on the target and integrated with the selected runtime stack.

Discuss:

- asynchronous kernel-to-user-space delivery;
- explicit loss notifications;
- lack of retransmission or durable queuing;
- separation between transport loss, submission failure and decoder failure.

The BPF ring buffer may be named only as a non-implemented alternative already
introduced in Chapter 2.

### 3.5 User-Space Runtime Design

#### 3.5.1 Loading, Attachment and Runtime Lifecycle

Describe the lifecycle at component level:

1. configuration and rule validation;
2. eBPF object and BTF resolution;
3. effective event selection;
4. module creation and selective loading;
5. filter-map configuration, perf-buffer startup and attachment;
6. signal-aware event processing;
7. orderly cleanup.

Safe claim: cleanup is attempted in an ordered way.

Unsafe claim: every cleanup or logger-synchronisation error is propagated to
the process result. The current runner ignores some deferred return values.

Do not claim that the runtime removes the memlock limit; the audited source does
not contain the older documented call.

#### 3.5.2 Decoding, Normalisation and Event Registration

Explain:

- structural validation before semantic processing;
- ID-to-schema resolution through an explicit registry;
- typed argument decoding;
- conversion to one normalised internal event model used by policy, detector
  and output components;
- dropping malformed or unknown records while keeping the event loop alive.

Avoid package, function and field names.

#### 3.5.3 Event Output, Alert Output and Operational Logging

Keep three responsibilities distinct:

- raw event presentation;
- derived alert presentation;
- structured operational diagnostics.

Alerts-only behaviour is a presentation decision: it does not disable
collection, decoding, policy admission or detector execution. Do not name its
CLI flag in Chapter 3. Do not describe Zap configuration here; that belongs in
Chapter 4.

### 3.6 Policy and Detection Architecture

#### 3.6.1 Policy-Based Event Scope

Binding semantics:

- policies match normalised events using the currently supported scope;
- suppress matches take precedence;
- exact non-suppress event allowlists can narrow the effective event selection
  before attachment;
- other policy checks remain in the user-space event path;
- policy files do not instantiate or select detector IDs;
- policy mode and intent metadata must not be described as distinct detector
  routing pipelines;
- matched-policy provenance is not currently injected into emitted detector
  alerts.

Do not imply that policies are evaluated completely in kernel space.

#### 3.6.2 Point and Collective Detectors

Explain:

- detector definitions are loaded independently from policies;
- event references are validated during startup;
- event-indexed dispatch avoids evaluating unrelated detectors;
- point detectors evaluate deterministic predicates over one event;
- collective detectors evaluate deterministic ordered sequences;
- detector failures are isolated from other detectors;
- alerts preserve one event or the completed sequence.

Detector requirements do not automatically select missing eBPF probes.
Operators must configure compatible event or policy scope.

#### 3.6.3 Bounded Local Correlation

Use the following conceptual wording as the basis for the section:

> Each collective detector defines how the events in its ordered sequence must
> be related. The available local relationships distinguish a stable process,
> an immediate parent-child transition, the same file resource, the same local
> cgroup, or a conjunction of these dimensions. A partial match is retained
> only within a short configurable interval and a bounded number of
> correlation keys. When the sequence completes, the alert preserves the
> ordered events that satisfied it; otherwise, the incomplete state expires,
> is replaced for the same key, or is evicted.

Required qualifications:

- sequence order is user-space arrival order;
- process identity distinguishes PID reuse;
- process-tree correlation is same process or immediate parent-to-child only;
- resource correlation uses stable local file identity and no pathname
  fallback;
- cgroup identity is host-local and not Kubernetes workload identity;
- composite correlation requires every configured dimension;
- `user_session` is rejected rather than approximated with UID;
- only one partial sequence is retained for a detector correlation key;
- state is bounded by time and a per-detector key cap, but total memory also
  depends on detector count, sequence length and event payload size;
- pruning occurs when relevant detector traffic is processed;
- the runtime does not schedule periodic engine flush operations.

The exact key encoding, concrete resource fields, default and maximum windows,
4096-key cap and pruning implementation belong in Chapter 4.

#### 3.6.4 Alert Evidence and MITRE ATT&CK Metadata

Explain:

- an alert is a derived detector result;
- point alerts preserve one source event;
- collective alerts preserve the ordered matching evidence;
- ATT&CK metadata comes from the detector definition and is copied into the
  alert;
- ATT&CK supplies a shared interpretation and reporting vocabulary.

Validation is syntactic, not semantic. Do not claim automatic validation
against an ATT&CK release, detector correctness or complete technique coverage.
Do not use unnecessarily dismissive wording: the metadata has concrete value
for traceability and communication.

### 3.7 Design Decisions and Scope Boundaries

Conclude by synthesising, rather than repeating, the main trade-offs:

- bounded kernel collection versus complex verifier-sensitive logic;
- Go rule processing versus recompiling kernel programs;
- selective attachment versus loading every registered probe;
- explicit event contract versus ad hoc payload interpretation;
- perf-buffer compatibility versus a non-implemented ring-buffer alternative;
- policy scope separated from detector logic;
- bounded local correlation versus general CEP or persistent graphs;
- observation and detection versus enforcement.

State explicitly that cluster-wide contextual correlation, Kubernetes workload
attribution, enforcement, machine-learning anomaly discovery, complete event
coverage and universal performance compliance are outside the implemented
scope.

Container packaging may be acknowledged as an implementation concern deferred
to Chapter 4. Kubernetes must not be presented as implemented or evaluated
thesis work.

## Approved Figure Plan

One figure is recommended for the first draft:

### Figure 3.1 - Responsibility Boundary And End-To-End Flow

The figure should contain three zones.

**Kernel space**

- selected hook;
- eBPF capture;
- optional UID rejection;
- binary event construction;
- perf-buffer submission.

**Go user space**

- configuration and probe selection;
- perf-buffer reader;
- decoder and event registry;
- policy scope;
- detector dispatch.

**Presentation**

- raw event output;
- alert output with evidence and ATT&CK metadata;
- separate operational logging lane.

Show the control path for configuration and probe selection separately from the
event data path. Do not route raw events or alerts through operational logging.

The second proposed physical-probe/dependency diagram is deferred to Chapter 4,
where the concrete registry implementation can be explained. It is not
required in Chapter 3.

The writer may omit Figure 3.1 from the first draft if no compilable,
professionally readable asset can be produced without adding an unapproved
graphics dependency. Mermaid must not be inserted in the LaTeX source.

## Citation Plan

Chapter 3 primarily describes the developed architecture. It should not cite
internal Markdown documents as external authority.

Reuse existing verified references only where a general claim needs support:

- `redhat2025backporting` for enterprise-kernel backport implications;
- `linux_btf`, `linux_bpf_llvm_relocations` and `linux_libbpf_overview` for BTF,
  CO-RE and libbpf boundaries;
- `linux2026ebpfverifier` for verifier constraints;
- `linux_perf_ring_buffer` for perf-buffer concepts;
- `aquasecurity_libbpfgo` for the Go/libbpfgo integration rationale;
- `cugola2012processing` and `agrawal2008eventstreams` only when distinguishing
  bounded local sequence matching from broader event-processing systems;
- `strom2020attack` or the already verified MITRE references for ATT&CK as a
  shared vocabulary.

Do not repeat the full literature discussion from Chapter 2. Do not add a
citation to every implementation statement. New bibliography entries require
independent verification before insertion.

## Binding Conflict Resolutions

| Topic | Binding decision |
|---|---|
| Repository state | Use committed baseline `163ad72`; exclude current networking and benchmark experiments. |
| Requirements format | Use the compact eight-FR and seven-NFR tables in this synthesis. |
| Ring buffer | Background or future alternative only; runtime uses perf buffers. |
| Probe registry | Public logical events may depend on hidden support probes. |
| Policies | Scope admitted events; exact allowlists may narrow attachment. They do not select detectors. |
| Detector dependencies | Validated as event names but not automatically merged into probe selection. |
| Policy modes | Do not describe monitor and detect as different detector-routing pipelines. |
| Policy provenance | Output capability exists, but runtime alerts do not automatically contain matched policy names. |
| Collective ordering | User-space arrival order, not reconstructed causal order. |
| Process tree | Same process or immediate parent-to-child only. |
| Resource identity | Stable local file identity; no pathname fallback. Exact fields deferred to Chapter 4. |
| Cgroup identity | Host-local correlation key, not Kubernetes workload identity. |
| Composite identity | Conjunction of all configured dimensions. |
| User session | Unsupported and rejected; UID is not a substitute. |
| Partial sequences | One retained partial sequence per detector correlation key. |
| State expiry | Pruned by detector activity or explicit flush; no periodic runtime flush scheduler. |
| State boundedness | Key count and time are bounded; total memory still depends on configuration and payload size. |
| ATT&CK | Metadata propagation and syntactic validation are implemented; semantic validation and complete coverage are not. |
| Logging | Operational diagnostics are separate from event and alert output. Zap details belong in Chapter 4. |
| Cleanup | Ordered cleanup is attempted; not every deferred error is propagated. |
| Memlock | Do not claim an explicit memlock-removal call in the audited runtime. |
| Performance | Below 5% is a profile-specific evaluation objective, not a universal achieved property. |
| Deployment | Container implementation belongs in Chapter 4; Kubernetes deployment is future context. |

## Material Reserved For Chapter 4

- package, file, function and structure walkthroughs;
- complete hook and event catalogues;
- concrete probe-registry entries and dependency representation;
- CLI flags and environment precedence;
- map names, keys and byte offsets;
- exact 136-byte common context;
- numeric event IDs and argument type constants;
- the 32,000-byte argument buffer;
- long-record offset defect and fix;
- exact decoder implementations;
- entry/exit state maps;
- precise correlation key encoding and resource fields;
- two-second default, five-second maximum and 4096-key cap;
- deduplication implementation;
- Zap configuration and libbpf log mapping;
- Dockerfile stages and packaged artifacts;
- target-specific verifier and compatibility workarounds.

## Material Reserved For Chapter 5

- representative functional hook validation;
- policy and detector positive and negative cases;
- noisy benign workloads and false-positive discussion;
- wrong-order, expired and mismatched collective sequences;
- process-tree direction and PID reuse tests;
- resource, cgroup and composite correlation case studies;
- arrival-order limitations under concurrent workloads;
- state and deduplication cardinality measurements;
- repeated benchmark profiles with warm-up and workload definition;
- separate user-space and kernel-side cost;
- UID-filter comparisons;
- event rate, loss and malformed-record behaviour under load;
- final ATT&CK coverage analysis;
- threats to validity.

## Prohibited Claims

The Chapter 3 draft must not claim:

- universal kernel portability;
- automatic support for missing kernel features through CO-RE;
- complete Linux or Tracee event coverage;
- production readiness;
- kernel-resident detector or policy execution;
- policy-driven detector activation;
- automatic detector-to-probe dependency planning;
- automatic matched-policy provenance in alerts;
- learned, statistical or general anomaly discovery;
- arbitrary process ancestry or persistent process graphs;
- pathname-based resource identity;
- user-session correlation;
- Kubernetes pod or workload attribution;
- cluster-wide contextual correlation;
- general complex-event processing;
- enforcement or prevention;
- universal CPU use below five per cent;
- semantic validation or complete coverage of MITRE ATT&CK.

## Wave 2 Acceptance Checklist

The Chapter 3 writer may proceed when:

- this file is present and unchanged unless explicitly revised by the user;
- the canonical structure in `definitive-outline.md` matches this synthesis;
- Chapter 1 and Chapter 2 remain the narrative baseline;
- uncommitted implementation experiments remain excluded;
- no blocking claim depends on missing evidence.

The resulting draft is acceptable for Wave 3 only if:

- every promised Chapter 1 topic is covered;
- requirements map clearly to architecture;
- Chapter 2 background is not repeated;
- Chapter 4 implementation details are deferred;
- Chapter 5 evaluation results are not pre-empted;
- terminology remains consistent;
- the internal project name does not appear;
- the abstract and existing chapters are not modified;
- the Thesis compiles without new unresolved references or citations.
