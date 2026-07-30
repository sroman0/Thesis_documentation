# Chapter 3 Policy and Detection Architecture Audit

## Scope and Evidence Baseline

This audit reconstructs the policy and detection architecture that is safe to
describe in Section 3.6 of the thesis. It is based on the committed
`demo_project` baseline at commit `163ad72`, the current rule definitions, and
the relevant Go tests. Code is treated as authoritative where comments or
documentation describe planned rather than implemented behaviour.

The working tree is not clean. In particular,
`rules/policies/test_net.yaml` is untracked and is not part of the committed
policy inventory. Its presence makes
`TestRepositoryPolicyPresetsLoadAndReferenceSupportedEvents` fail because the
test expects eight repository presets and finds nine. This is an inventory
inconsistency in the working tree, not a failure of policy matching. The
targeted policy tests and the detector, YAML, event, output, configuration, and
command integration tests pass.

The architectural evidence supports deterministic, rule-based point detection
and bounded local collective detection. It does not support claims of learned
behaviour, statistical anomaly scoring, general complex-event processing,
cluster-wide correlation, or enforcement.

## 1. Architectural Distinction Between Policy, Detector, and Alert

### Policy

A policy defines the monitoring scope. It is a normalised userspace object
containing a mode, an intent, shared scope constraints, and one or more event
rules. The implemented matching fields are:

- event name;
- process command name;
- UID.

An event is admitted when at least one non-suppress policy matches it and no
matching suppress policy overrides that decision. If no policies are
configured, the policy layer is permissive. Policy modes record whether a
matching policy is intended for monitoring, detection, or suppression, but the
current event path ultimately consumes only the resulting allow-or-drop
decision. Policy intent is retained as configuration metadata and does not
change detector execution.

Policies do not define detector conditions, create detectors, select detector
identifiers, or select MITRE ATT&CK metadata. Their primary implemented role is
to constrain which decoded events continue through the runtime. Under an exact
event allowlist, they can also reduce the eBPF programs selected before object
loading and attachment.

### Detector

A detector is an independently loaded analytical component. It declares:

- a stable identifier and descriptive metadata;
- the events it consumes or requires;
- optional global and per-step predicates;
- whether it is stateful;
- an optional short correlation window and correlation strategy;
- alert metadata, including severity and optional ATT&CK classification.

The dispatcher indexes detectors by event name. A point detector evaluates one
normalised event. A collective detector evaluates an ordered sequence of
normalised events and retains incomplete local evidence until the sequence
completes, expires, is overwritten for the same key, or is evicted by the state
limit.

Detector loading is controlled by detector paths, independently of policy
files. A policy may admit the event stream needed by a detector, but it does
not activate that detector.

### Alert

An alert is the derived result of a successful detector match. It contains the
detector identity, title, description, severity, creation time, ATT&CK metadata,
free-form metadata, and the event evidence that caused the match. A point alert
normally contains one event. A collective alert contains the ordered sequence
retained by the detector.

Alert rendering is separate from detection. Table output presents a compact
summary and a bounded textual sequence, while JSON output preserves the
structured evidence and threat metadata. Operational Zap logs are a third,
separate output path and do not replace event or alert output.

This separation is an adopted architectural pattern. It should not be
presented as the novelty of the thesis. The project-specific contribution lies
in how the pattern is adapted to a constrained enterprise kernel target and
combined with bounded local correlation and a stable event pipeline.

## 2. Policy Scope and Its Effect on Probe Attachment

Policies are loaded and validated before the eBPF runtime is constructed. Their
event references are checked against the userspace event registry. At runtime,
policy matching is performed after binary decoding, configured event
selection, and command-name filtering, but before raw event output and detector
dispatch.

The policy manager can derive an exact event selection only when every
non-suppress rule has an explicit, non-empty include list. In that case:

1. the explicit policy includes are combined into an allowlist;
2. if the CLI also selected events, the runtime computes their intersection;
3. the resulting list replaces the configured event selection before the eBPF
   runtime is created;
4. the probe registry uses that selection to choose the programs that are
   loaded and attached, together with any required support dependencies.

This is an optimisation boundary rather than a complete kernel policy engine.
Command-name and UID policy constraints remain userspace decisions because
they cannot be compiled into this exact event-name allowlist. Suppress policies
also remain runtime filters and are excluded from the attach-time allowlist
calculation.

If a non-suppress rule has an empty include list, it means that the rule may
match every event. The manager therefore refuses to narrow probe attachment and
keeps policy evaluation in userspace.

An uncovered edge case exists when the intersection between explicit CLI
events and an exact policy allowlist is empty. The policy manager returns an
empty list with an exact-selection result, while the probe selector interprets
an empty include list as selecting all supported events. Policies will still
drop non-matching decoded events, but the intended attach-time reduction is
lost. Chapter 3 should not claim that every exact intersection necessarily
produces a minimal non-empty attachment set until this case is explicitly
handled and tested.

Safe Chapter 3 wording:

> Policies constrain the normalised event stream through event, command-name,
> and UID selectors. When all active allow rules provide an explicit event
> list, that list can also be used before object loading to reduce the set of
> eBPF programs attached to the kernel. More specific scope constraints and
> suppression decisions remain in user space.

## 3. Independent Detector Loading

Detector loading is enabled by the presence of one or more detector paths.
Paths may refer to individual YAML files or directories. Directory expansion
is non-recursive, accepts `.yaml` and `.yml` files, and is sorted to keep
loading deterministic.

Each document is parsed into a normalised detector definition. Event
references are validated against the event registry, conditions and sequence
steps are compiled, and the detector is registered in an engine. The
dispatcher then builds an event-name index so an event is offered only to
interested detectors, apart from explicitly wildcard definitions.

Policy paths and detector paths are separate configuration inputs. The runtime
does not:

- derive detector paths from a policy;
- select detectors by policy name;
- select detectors by ATT&CK tactic or technique;
- automatically add every detector-required event to the active event set;
- verify that the final policy and CLI event selection supplies every event
  required by every loaded detector.

Consequently, a syntactically valid detector can be loaded but remain inert if
the selected event stream excludes one of its required events. This should be
presented as a current configuration responsibility and a candidate validation
improvement, not as automatic policy-detector orchestration.

Detectors can also run when alert output is disabled. In that configuration the
engine still evaluates events, but produced alerts are not rendered because no
alert printer is installed. The `alerts-only` option concerns presentation: it
hides raw event output and enables alert output, but it does not change the
detector matching algorithm or act as a kernel filter.

## 4. Point-Detector Lifecycle

The implemented point-detector lifecycle is:

1. **Load and normalise.** A YAML document is decoded into the external schema
   and converted into a detector definition.
2. **Validate.** Required metadata, severity, window bounds, event references,
   and the syntactic form of ATT&CK identifiers are checked.
3. **Compile.** Conditions are converted to internal operators and expected
   values before the event hot path. Event names from requirements, consumes,
   and steps form the detector's event set.
4. **Register and index.** The detector registry enforces valid, unique
   identifiers. The dispatcher indexes the detector by relevant event names.
5. **Initialise.** The engine calls `Init` on every detector. YAML detectors
   currently require no external initialisation.
6. **Dispatch.** After decoding and runtime filters, an event is sent only to
   candidate detectors.
7. **Evaluate.** Global conditions are applied first. If the definition
   contains steps but is not a multi-step stateful sequence, the conditions of
   the applicable step are also evaluated against the same event.
8. **Create evidence.** A successful match creates one alert containing the
   source event and the detector's alert and threat metadata.
9. **Deduplicate.** The engine suppresses an equivalent alert repeated within a
   five-second deduplication window.
10. **Render.** If alert output is enabled, the selected alert printer emits a
    compact table record or structured JSON.

The predicate operators currently include equality, inequality, containment,
membership, prefix, suffix, existence, non-existence, greater-than, and
less-than comparisons.

A detector declared `stateful` with zero or one step does not run the collective
state machine; the collective path requires more than one step. This is a
runtime distinction that should be validated more explicitly at configuration
time if a stateful declaration is intended always to mean a sequence.

## 5. Collective-Detector Lifecycle and Evidence Retention

A collective detector uses an ordered list of at least two steps. Its
correlation strategy is resolved when the parsed definition is validated and
again when the runtime detector is constructed. Strategy names are therefore
not parsed on every event, although describing the plan as being compiled
exactly once would be too literal.

For each relevant event, the detector:

1. applies global conditions;
2. obtains the current userspace clock time;
3. performs periodic expiry pruning when due;
4. constructs correlation lookup keys for the event;
5. searches for one partial sequence awaiting the current step;
6. checks that the partial sequence is still inside its time window;
7. evaluates the next ordered step;
8. appends the event when the step matches;
9. emits an alert and deletes the state when the last step completes;
10. otherwise, if the event matches the first step, starts or restarts one
    partial sequence for its start key.

The resulting alert preserves the ordered event slice accumulated by the state
machine. This is the evidence behind the collective match and is available as
structured data in JSON output. Table output serialises the sequence into a
bounded line for readability.

Only one partial sequence is retained for a detector correlation key. A new
first-step event for the same key replaces the previous incomplete sequence.
The detector therefore does not preserve overlapping candidate sequences for
one key. It also does not reorder events, search backwards, or reconstruct a
general event graph. The sequence follows the order in which events reach the
userspace detector engine.

The implementation uses `time.Now` for sequence windows and alert creation
rather than the timestamp carried by the kernel event context. This makes
windows local to processing time. Because perf events can arrive through
per-CPU streams, Chapter 5 should validate whether event arrival order remains
adequate for the selected multi-event scenarios under load.

## 6. Implemented Correlation Semantics

If `group_by` is omitted, the detector defaults to process correlation. Aliases
such as `pid`, `host_pid`, and their qualified forms are normalised to the
process strategy. Duplicate components, including duplicates created through
alias normalisation, are rejected.

### Process

Process correlation uses the host PID together with the process start time.
Both values must be non-zero. Including start time prevents a later process
that reuses the same PID from advancing an earlier process's partial sequence.

This strategy requires all sequence events to carry the same stable process
identity.

### Process Tree

The first step is stored under the current process identity. Later events are
looked up using two candidates:

- the current process identity;
- the immediate parent's host PID and parent start time.

It therefore supports a sequence that remains in the same process or moves
from a parent to its immediate child. It does not reconstruct arbitrary
ancestry, retain a persistent process graph, match child-to-parent transitions,
or traverse multiple generations.

### Resource

Resource correlation uses the pair `(device, inode)`. Every step event in a
resource-correlated detector must expose both `dev` and `inode` in the event
schema, and the runtime values must be available as numeric arguments.

The pathname is deliberately not part of resource identity and is not used as
a fallback. Two processes may therefore correlate when they act on the same
device and inode even if their path strings differ. Conversely, equal pathname
strings without equal device and inode values do not establish a match.

### Cgroup

Cgroup correlation uses the local numeric cgroup ID carried by the event
context. A zero ID is treated as unavailable, so the event cannot start or
advance a cgroup-correlated sequence.

The identity is local to the monitored host and kernel. It may connect events
from different processes in the same observed cgroup, but it must not be
described as a Kubernetes workload, pod, tenant, or cluster-wide identity.

### Composite

A composite strategy is an ordered list of two or more correlation components,
for example process plus resource. Runtime keys contain every configured
component. If any component is unavailable or differs, the event cannot match
the partial sequence.

Composite correlation is therefore conjunction, not fallback: all components
must match. A `process_tree` component may contribute current-process and
immediate-parent candidates, but each candidate is still combined with every
other configured component.

### Rejected User Session

`user_session` is recognised as a proposed strategy but is rejected during
detector parsing. Normalised events do not expose a stable local session
identifier, and the implementation intentionally does not approximate session
identity with UID. UID alone could merge unrelated logins or processes and
would not satisfy the intended correlation semantics.

## 7. Time, Pruning, and Retained-State Boundaries

Stateful detectors use a two-second default window. The parser accepts an
explicit duration, but validation rejects windows longer than five seconds.
These values are deliberate performance guardrails for the target virtual
machine rather than claims that every meaningful security sequence fits inside
five seconds.

Incomplete state is bounded in two ways:

- expiration after the detector's effective window;
- a hard maximum of 4096 partial correlation keys per detector.

When a new key would exceed the cap, the oldest retained partial sequence is
evicted. The cap is per detector, not global across the engine.

Expiry is checked directly whenever an event addresses a retained key.
Unrelated stale entries are scanned periodically, at half the configured
window with a maximum interval of one second. Pruning occurs when relevant
stateful events invoke the detector.

The detector interface and engine expose `Flush`, and YAML detector flushes
remove expired partial sequences without emitting timeout alerts. The main eBPF
runtime does not currently schedule periodic engine flushes or invoke a final
flush during shutdown. If no further relevant event arrives, stale entries may
therefore remain beyond their nominal time window until another detector event
causes pruning. The hard key cap still limits their count.

The number of retained keys is bounded, but the YAML schema does not impose a
hard maximum on the number of sequence steps or conditions in one detector.
Each partial state retains one event per completed step. Strict whole-engine
memory bounds therefore also depend on configuration size, detector count, and
event payload size. These dimensions require benchmark evidence before the
architecture can be described as having a universal fixed memory bound.

The engine separately retains alert deduplication timestamps for five seconds.
Old deduplication entries are pruned when a non-empty alert batch is processed.
This map has time-based cleanup but no explicit cardinality cap, which is a
distinct state-management concern from the 4096-key sequence limit.

Safe Chapter 3 wording:

> Each collective detector defines how the events in its ordered sequence must
> be related, using one or more local process, parent-child, file-resource, or
> cgroup identities. Incomplete matches are retained only within a short
> configurable window and a per-detector key limit; completed alerts preserve
> the ordered events that satisfied the sequence.

## 8. MITRE ATT&CK Metadata: Role and Validation Limits

ATT&CK metadata belongs to detector definitions, not policies. The committed
detector presets currently declare `MITRE ATT&CK Enterprise` and version
`v19.1`, together with tactic and technique identifiers appropriate to their
intended interpretation.

When a detector matches, its complete threat metadata is copied into the alert.
Table output presents a compact list of tactic and technique identifiers. JSON
output preserves framework, version, tactics, techniques, data sources, and
data components as structured fields.

Validation is intentionally limited:

- tactic identifiers are checked only against the syntactic form `TA` followed
  by four digits;
- technique identifiers are checked only against `T` followed by four digits,
  with an optional three-digit sub-technique suffix;
- empty ATT&CK metadata is accepted;
- the framework and version strings are not constrained;
- identifiers are not checked against a downloaded ATT&CK release;
- names are not checked for consistency with identifiers;
- data-source and data-component values are not semantically validated;
- the runtime does not prove that a detector correctly implements or
  comprehensively covers the declared technique.

The mapping is nevertheless valuable: it gives alerts a standard vocabulary,
supports interpretation and coverage reporting, and makes the intended threat
behaviour explicit. Chapter 5 should validate representative mappings against
the actual detector logic and discuss coverage as observed scope rather than
as complete ATT&CK coverage.

Although the output model can read policy names from alert metadata, YAML
detectors do not populate `policy_names`, and the runtime does not inject the
policies that matched the source event. Automatic policy provenance is
therefore not currently present in emitted alerts.

## 9. Claim-to-Code-and-Test Matrix for Section 3.6

| Architectural claim | Status | Primary code evidence | Test evidence | Safe Chapter 3 interpretation |
|---|---|---|---|---|
| Policies and detectors have separate configuration and runtime roles. | Verified | `pkg/policy/policy.go`; `pkg/cmd/project.go:newRuntimeExtensions`; `pkg/detectors/detector.go` | `TestNewRuntimeExtensionsLoadsDetectorEngine`; policy loader tests | Policies scope events; independently loaded detectors evaluate admitted normalised events. Do not claim this separation as novel. |
| Policy matching supports event name, command name, and UID, with suppress precedence. | Verified | `pkg/policy/policy.go:Policy.Matches`; `pkg/policy/manager.go:Match` and `IsEventSelected` | `TestPolicyMatchesWhenScopeAndRuleAcceptEvent`; `TestManagerSuppressWinsOverOtherModes` | Policy scope is intentionally limited and deterministic. |
| Exact policy event allowlists can influence probe attachment. | Verified with edge case | `pkg/policy/manager.go:EffectiveEventSelection`; `pkg/cmd/project.go:NewProjectRunner`; `pkg/ebpf/probes/probes.go:Select` | `TestEffectiveEventSelectionUsesExplicitPolicyAllowlist`; `TestEffectiveEventSelectionIntersectsCLIEvents`; `TestNewProjectRunnerAppliesExactPolicyEventSelection` | Explicit allowlists can narrow attachment before load; do not claim complete policy execution in kernel space. Empty intersections need explicit handling. |
| Detector files are loaded independently from policy files. | Verified | `pkg/config/config.go:Normalize`; `pkg/cmd/project.go:loadDetectors` and `detectorFiles` | `TestNewRuntimeExtensionsLoadsDetectorEngine`; detector-file tests | Policies do not activate detector IDs or ATT&CK mappings. |
| Detector event references are validated against the event registry. | Verified | `pkg/detectors/yaml/parser.go:validateEventReferences`; `pkg/cmd/project.go:loadDetectors` | `TestParseBytesRejectsUnsupportedEvents`; `TestNewRuntimeExtensionsRejectsDetectorUnknownEvent` | Invalid event names fail at startup. Compatibility between final policy scope and detector requirements is not automatically validated. |
| Dispatch is indexed by event name and detector failures are isolated. | Verified | `pkg/detectors/dispatch.go` | `TestDispatcherRoutesEventToMatchingDetectors`; `TestDispatcherKeepsDetectorErrorsNonFatal`; `TestDispatcherSkipsAllDetectorsForUnmatchedEvent` | Candidate indexing limits unnecessary detector evaluation, while one detector error does not stop others. |
| Point detectors emit one-event evidence after deterministic condition matching. | Verified | `pkg/detectors/yaml/detector.go:OnEvent`, `matches`, and `newAlert` | Point condition, operator, and step-condition tests in `detector_test.go` | A point alert is a rule match over one normalised event, not a learned anomaly score. |
| Collective detectors match ordered, short-window sequences. | Verified | `pkg/detectors/yaml/detector.go:onStatefulEvent` | `TestStatefulDetectorEmitsAlertWhenSequenceCompletes`; `TestStatefulDetectorExpiresSequenceWindow` | Sequence order is userspace arrival order; no out-of-order or general CEP semantics are implemented. |
| Collective alerts preserve the complete matched evidence sequence. | Verified | `pkg/detectors/yaml/detector.go:newAlertFromEvents`; `pkg/output/alert.go` | `TestNewAlertRecordNormalizesDetectorAlert`; `TestFormatAlertSequenceIsBounded` | JSON preserves structured evidence; table output gives a bounded summary. |
| Process identity includes host PID and start time. | Verified | `pkg/detectors/yaml/correlation.go:processIdentity` | `TestProcessCorrelationRejectsPIDReuse`; host-PID grouping test | Start time prevents PID reuse from joining unrelated sequences. |
| Process-tree correlation is limited to same process or immediate parent-to-child. | Verified in code; partially tested | `pkg/detectors/yaml/correlation.go:correlationComponent.values` | `TestStatefulDetectorCanGroupByProcessTree` | Do not claim arbitrary ancestry or a persistent process graph. A negative reverse/multi-generation test is still desirable. |
| Resource correlation requires device and inode and has no pathname fallback. | Verified | `pkg/detectors/yaml/correlation.go:compileCorrelationPlan` and `resourceIdentity` | `TestResourceCorrelationMatchesDifferentProcesses`; `TestResourceCorrelationIgnoresPathnameIdentity`; `TestResourceCorrelationRequiresDeviceAndInode` | Resource identity is a local file-object pair, not a path string. |
| Cgroup correlation can relate different processes using a local cgroup ID. | Verified | `pkg/detectors/yaml/correlation.go` | `TestCgroupCorrelationMatchesAcrossProcesses` | It is a host-local identity, not a Kubernetes workload identity. Explicit zero-ID rejection lacks a dedicated test. |
| Composite correlation requires every configured component. | Verified | `pkg/detectors/yaml/correlation.go:correlationPlan.keys` | `TestCompositeCorrelationRequiresEveryComponent`; duplicate-alias parser test | Composite strategies are conjunctions, not fallback alternatives. |
| `user_session` is rejected rather than approximated with UID. | Verified | `pkg/detectors/yaml/correlation.go:normalizeCorrelationComponent` | `TestParseBytesRejectsUnavailableUserSessionCorrelation` | Session correlation is unsupported because no stable local session identifier is available. |
| Only one partial sequence is retained per detector correlation key. | Verified in code | `pkg/detectors/yaml/detector.go:storeSequence` | Indirect sequence tests; no dedicated overlapping-sequence test | A repeated first step for the same key replaces the earlier partial match. |
| Sequence state has short time windows and a 4096-key per-detector cap. | Verified | `pkg/detectors/definition.go`; `pkg/detectors/yaml/detector.go` | performance-bound tests; expiry, periodic-prune, flush-prune, and `TestCorrelationStateHasHardBound` | State is bounded by time and keys, subject to configuration-size qualifications. |
| Alerts carry ATT&CK metadata from the matching detector. | Verified | `pkg/detectors/yaml/detector.go:newAlertFromEvents`; `pkg/output/alert.go` | `TestDetectorPropagatesThreatMetadataToAlert`; output alert and JSON tests | ATT&CK supports classification and interpretation. Validation is syntactic, not semantic. |
| Alert deduplication reduces repeated identical output. | Verified | `pkg/detectors/engine.go:deduplicateAlerts` | duplicate suppression and post-window tests | Deduplication is a short userspace readability control and is separate from detector correlation. |
| Emitted alerts automatically identify the matching policies. | Unsupported | Output can parse `policy_names`, but runtime policy results are not added to detector alerts | Output metadata parsing is tested only with manually supplied metadata | Do not claim policy provenance in current alerts. |

## 10. Unsupported Behaviours and Chapter 5 Validation Needs

### Unsupported or Incomplete Behaviours

The current implementation does not provide:

- machine-learning, statistical, or baseline-driven anomaly detection;
- arbitrary contextual anomaly detection;
- cluster-wide or cross-node collective correlation;
- a persistent process graph or arbitrary ancestry traversal;
- out-of-order event matching or general complex-event processing;
- more than one overlapping partial sequence for the same detector key;
- stable local user-session correlation;
- pathname fallback for file-resource identity;
- Kubernetes pod or workload identity derived from cgroup IDs;
- policy selection of detector IDs, tags, tactics, or techniques;
- automatic activation of detector-required events;
- automatic policy provenance in alerts;
- detector-based enforcement or prevention;
- periodic runtime calls to detector `Flush`;
- semantic validation against a specific ATT&CK dataset;
- universal proof that the five-per-cent CPU objective is met.

The system is an observation and local detection prototype. These boundaries
should appear explicitly in Section 3.7 rather than being left implicit.

### Chapter 5 Validation Needs

Chapter 5 should evaluate:

1. **Policy selection correctness.** Confirm event, command-name, UID, and
   suppress precedence, including the effect of explicit policy allowlists on
   probe attachment.
2. **Empty event intersections.** Define and test the expected behaviour when
   CLI selection and exact policy selection have no event in common.
3. **Point-detector case studies.** Demonstrate true matches, rejected benign
   cases, alert deduplication, and readable evidence.
4. **Collective-detector case studies.** Demonstrate successful ordered
   sequences, wrong order, expired windows, different correlation keys, and
   repeated first steps.
5. **Process-tree limits.** Test same-process, parent-to-child, reverse, sibling,
   and multi-generation sequences.
6. **Resource identity.** Test equal device/inode across different pathnames,
   equal pathname with different identity, and unavailable arguments.
7. **Cgroup identity.** Test different processes in one cgroup, different
   cgroups, zero or unavailable IDs, and document host-local semantics.
8. **Composite correlation.** Test that every component is required and measure
   the additional per-event cost.
9. **Arrival ordering.** Exercise collective detectors under concurrent,
   cross-CPU workloads to identify ordering-related false negatives.
10. **State bounds.** Measure CPU and memory with increasing detector count,
    key cardinality, step count, and event payload size; verify oldest-state
    eviction at the cap.
11. **Idle expiry.** Determine whether the runtime should schedule engine flush
    operations so expired state is reclaimed without waiting for another
    relevant event.
12. **Deduplication state.** Measure cardinality under many unique alerts and
    decide whether a hard bound is required.
13. **Policy-detector compatibility.** Validate or detect configurations in
    which policies exclude events required by loaded detectors.
14. **ATT&CK mapping.** Review selected tactic and technique mappings against
    detector logic and report coverage without implying completeness.
15. **Performance profiles.** Compare collection-only, point-detection,
    collective-detection, kernel-filtered, and worst-case high-cardinality
    profiles after a defined warm-up period.

## Recommended Narrative for Section 3.6

### 3.6.1 Policy-Based Event Scope

Introduce policies as declarative userspace scope controls over normalised
events. Explain that explicit event allowlists may reduce probe attachment
before loading, while command-name, UID, and suppression decisions remain in
the event path. State that policies do not instantiate detectors.

### 3.6.2 Point and Collective Detectors

Define point detectors as deterministic conditions over one event and
collective detectors as deterministic ordered sequences. Explain independent
detector loading, event-indexed dispatch, and evidence-preserving alerts.
Avoid YAML field names and source-code structures.

### 3.6.3 Bounded Local Correlation

Describe configurable local relationships based on stable process identity,
immediate parent-child identity, file-resource identity, cgroup identity, and
combinations of these dimensions. State that all configured dimensions must
match, and that incomplete sequences are limited by short windows and a
per-detector state cap. Explicitly exclude persistent graphs and cluster-wide
correlation.

Recommended paragraph:

> Each collective detector defines how the events in its ordered sequence must
> be related. The available local relationships distinguish a stable process,
> an immediate parent-child transition, the same file resource, the same
> cgroup, or a conjunction of these dimensions. A partial match is retained
> only within a short configurable interval and a bounded number of correlation
> keys. When the sequence completes, the alert preserves the ordered events
> that satisfied it; otherwise, the incomplete state expires or is evicted.

### 3.6.4 Alert Evidence and MITRE ATT&CK Metadata

Explain that alerts carry the event or sequence that caused the match and copy
the detector's ATT&CK classification. Present ATT&CK as a shared vocabulary for
interpretation and coverage analysis. Reserve exact schema fields, compact
table formatting, and JSON structures for Chapter 4.

## Boundary Between Chapters 3, 4, and 5

Chapter 3 should explain responsibilities, data flow, local correlation
semantics, bounded-state rationale, and the separation between scope, detection,
and presentation. It should not become a walkthrough of Go packages, function
names, YAML tags, map representations, key encodings, or pruning loops.

Chapter 4 should document:

- concrete policy and detector schemas;
- loader, registry, dispatcher, and engine implementation;
- condition operators;
- correlation-key construction;
- the exact 4096-key cap and pruning implementation;
- deduplication-key construction;
- alert table and JSON schemas;
- ATT&CK propagation and syntactic validation.

Chapter 5 should provide behavioural and performance evidence for the claims
listed above, clearly separating completed reproducible tests from future or
exploratory evaluation.
