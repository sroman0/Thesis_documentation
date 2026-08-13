# Chapter 3 Requirements and Constraints Audit

## 0. Audit Scope and Repository State

This audit supports Chapter 3, *Requirements and System Design*. It treats the
implementation as authoritative where narrative documentation is older than
the code. The audit was performed against:

- `demo_project` branch `feature/composite-collective-correlation`;
- committed baseline `163ad72` (`fix(ebpf): preserve argument offsets in long
  event records`);
- parent feature commit `35103ea` (`feat(detectors): add composite local
  correlation strategies`);
- the Chapter 1 and Chapter 2 LaTeX sources and all sources listed in the
  Chapter 3 audit prompt.

The committed baseline passed the selected unit-test suite covering
configuration, command wiring, event registration, probe selection, decoding,
policies, detectors, output and logging. The current worktree did not pass the
same suite because of experimental, uncommitted changes described separately
in Section 6.1.

This document distinguishes:

- **implemented**: present in the committed baseline and supported by code or
  tests;
- **partially implemented**: usable, but narrower than the apparent design
  intention;
- **evaluation pending**: implemented behaviour whose acceptance evidence
  belongs in Chapter 5;
- **aspirational**: documented as future work and not supported by the current
  implementation.

## 1. Functional Requirements

### FR-01: Collect Process and Security-Relevant Kernel Events

**Requirement.** The system shall observe a configurable set of host-level
events relevant to process execution, credentials, files, namespaces, kernel
modules, capabilities and related security activity.

**Status:** Implemented for the event families registered by the prototype.
This is not a claim of complete Linux or Tracee event coverage.

**Evidence:**

- `pkg/ebpf/c/project.bpf.c`: eBPF event producers;
- `pkg/ebpf/probes/probes.go`: `supportedProbes`;
- `pkg/events/spec.go`: userspace event definitions;
- `pkg/ebpf/probes/probes_test.go`:
  `TestDecoderEventsAreExposedByPublicProbes`.

### FR-02: Select Logical Events and Attach Only Necessary Programs

**Requirement.** The user shall be able to include or exclude logical events.
The runtime shall load and attach the programs required by that selection,
including hidden support dependencies, while disabling unrelated registered
programs before object loading.

**Status:** Implemented.

**Evidence:**

- `pkg/config/config.go`: `EventConfig`;
- `pkg/cmd/cobra/cobra.go`: `--events`, `--drop-events`, `--list-events`;
- `pkg/ebpf/probes/probes.go`: `Spec`, `Select`, `ConfigureAutoload`,
  `EmitsEvent`;
- `pkg/ebpf/probes/probes_test.go`:
  `TestSelectIncludeAndExclude`,
  `TestSelectAttachesImpliedProbesWithoutExposingInternalEvents`,
  `TestInternalProbesAreHiddenFromCliSelection`.

### FR-03: Load and Manage the eBPF Runtime

**Requirement.** The userspace runtime shall resolve the eBPF object and BTF
inputs, configure program autoload, load and attach selected programs, consume
records, and release links, buffers, cgroup resources and the module during
shutdown.

**Status:** Implemented.

**Evidence:**

- `pkg/cmd/initialize/bpfobject.go`: `BPFObject`, `readBPFObject`,
  `resolveBPFObjectPath`;
- `pkg/cmd/project.go`: `ProjectRunner.Run`;
- `pkg/ebpf/project.go`: `Project.Init`, `Project.Run`, `Project.Close`;
- `embedded-ebpf.go`: `BPFBundle`.

### FR-04: Transfer Events Through a Perf Buffer and Report Loss

**Requirement.** Kernel events shall be transferred to userspace through a
transport supported by the target kernel. Lost records shall be made visible as
operational diagnostics.

**Status:** Implemented with a perf-event array. A BPF ring buffer is not part
of the current runtime.

**Evidence:**

- `pkg/ebpf/c/maps.h`: `events` as `BPF_MAP_TYPE_PERF_EVENT_ARRAY`;
- `pkg/ebpf/c/common/buffer.h`: `events_perf_submit`;
- `pkg/ebpf/project.go`: `Project.Init` calls `InitPerfBuf`; `Project.Run`
  consumes both event and lost-event channels.

### FR-05: Decode a Stable Typed Event Contract

**Requirement.** The consumer shall transform binary records into normalised
events containing a common context and typed, named arguments. Unknown event
identifiers and malformed or inconsistent argument records shall fail
explicitly rather than being silently interpreted.

**Status:** Implemented.

**Evidence:**

- `pkg/ebpf/c/types.h`: `event_context_t`, `event_data_t`;
- `pkg/bufferdecoder/protocol.go`: `EventContext`, fixed
  `eventContextSize = 136`;
- `pkg/bufferdecoder/eventsreader.go`: `DecodeEvent`, `decodeArguments`,
  `decodeUnknownArguments`;
- `pkg/events/spec.go`: event names, identifiers and argument schemas;
- `pkg/bufferdecoder/eventsreader_test.go`:
  `TestDecodeEventPreservesContextAndArguments`,
  `TestDecodeStringArrayRecord`,
  `TestDecodeArgsArrayRecord`,
  `TestDecodeSecurityBprmCheckWithArgumentsBeyond256Bytes`.

### FR-06: Apply User-Space Event and Process Filters

**Requirement.** Before output and detector evaluation, the runtime shall
enforce the effective logical-event selection, command-name filters and active
policy scope.

**Status:** Implemented.

**Evidence:**

- `pkg/cmd/cobra/cobra.go`: `--comms`;
- `pkg/ebpf/project.go`: `Project.handleRawEvent`, `eventEnabled`,
  `commEnabled`, `policyEnabled`.

### FR-07: Load and Evaluate Configurable Policies

**Requirement.** The system shall load YAML policies from configured files or
directories, validate their event references, and use their event, command and
UID selectors to determine whether an event continues through the userspace
pipeline. Explicit event allowlists should narrow the eBPF selection before
load and attachment when this can be done without changing policy semantics.

**Status:** Implemented, with qualifications.

**Evidence:**

- `pkg/policy/loader.go`: `LoadFiles`;
- `pkg/policy/policy.go`: `Policy`, `Scope`, `Rule`, `EventSelector`,
  `Policy.Matches`;
- `pkg/policy/manager.go`: `Manager.Match`, `IsEventSelected`,
  `EffectiveEventSelection`;
- `pkg/cmd/project.go`: `newRuntimeExtensions`, `validatePolicyEvents`,
  `NewProjectRunner`;
- `pkg/policy/manager_test.go`:
  `TestManagerSuppressWinsOverOtherModes`,
  `TestEffectiveEventSelectionUsesExplicitPolicyAllowlist`,
  `TestEffectiveEventSelectionIntersectsCLIEvents`.

**Qualification.** `monitor`, `detect` and `suppress` are represented in the
policy model, but the current event pipeline mainly uses the aggregate
allow/suppress result. A `detect` policy does not select detector identifiers,
and a `monitor` policy does not independently prevent detector evaluation.

### FR-08: Load and Dispatch Point Detectors

**Requirement.** The runtime shall load detector definitions independently
from policies, validate their consumed events, dispatch each normalised event
only to relevant detectors, and emit alerts for deterministic single-event
matches.

**Status:** Implemented.

**Evidence:**

- `pkg/cmd/project.go`: `loadDetectors`, `detectorFiles`,
  `newRuntimeExtensions`;
- `pkg/detectors/detector.go`: `Detector`;
- `pkg/detectors/registry.go`: `Registry`;
- `pkg/detectors/dispatch.go`: `Dispatcher`;
- `pkg/detectors/engine.go`: `Engine.ProcessEvent`;
- `pkg/detectors/yaml/parser.go`: `ParseFile`, `ParseBytes`;
- `pkg/detectors/dispatch_test.go`:
  `TestDispatcherRoutesEventToMatchingDetectors`,
  `TestDispatcherSkipsAllDetectorsForUnmatchedEvent`.

### FR-09: Evaluate Bounded Local Collective Detectors

**Requirement.** A detector shall be able to describe an ordered local
sequence, define how its events must be related, and retain incomplete matches
for only a short, finite interval.

**Status:** Implemented for bounded, detector-specific local correlation.

**Supported correlation dimensions:**

- stable process identity based on host process identity and start time;
- the current process or its immediate parent-child relationship;
- file-resource identity based on device and inode;
- local cgroup identity;
- composite keys requiring all configured dimensions to match.

**Boundaries:**

- no persistent process graph or recursive ancestry reconstruction;
- no cluster-wide identity or Kubernetes workload correlation;
- no approximation of `user_session` through UID;
- one partial state per correlation key rather than arbitrary overlapping
  sequence instances;
- a maximum five-second detector window and 4096 partial states per detector.

**Evidence:**

- `pkg/detectors/definition.go`: `DefaultStateWindow`,
  `MaxStateWindow`, `Definition.EffectiveWindow`;
- `pkg/detectors/yaml/correlation.go`: `compileCorrelationPlan`,
  `correlationPlan.keys`;
- `pkg/detectors/yaml/detector.go`: `Detector.OnEvent`,
  `maxSequenceStates`, `pruneExpiredStateIfDue`, `evictOldestState`;
- `pkg/events/spec.go`: argument-schema checks used by resource correlation;
- `pkg/detectors/yaml/correlation_test.go`:
  `TestProcessCorrelationRejectsPIDReuse`,
  `TestResourceCorrelationMatchesDifferentProcesses`,
  `TestCompositeCorrelationRequiresEveryComponent`,
  `TestCgroupCorrelationMatchesAcrossProcesses`,
  `TestCorrelationStateHasHardBound`;
- `pkg/detectors/yaml/parser_test.go`:
  `TestParseBytesRejectsUnavailableUserSessionCorrelation`.

### FR-10: Preserve Alert Evidence and MITRE ATT&CK Metadata

**Requirement.** Every emitted alert shall identify the detector and preserve
the event or ordered event sequence that satisfied it. Detector definitions
may attach MITRE ATT&CK tactics, techniques, data sources and data components,
which shall be propagated to alert output.

**Status:** Implemented for detector evidence and ATT&CK metadata.

**Evidence:**

- `pkg/detectors/detector.go`: `Alert`;
- `pkg/detectors/definition.go`: `ThreatMetadata`, `AttackTactic`,
  `AttackTechnique`, `ThreatMetadata.Validate`;
- `pkg/detectors/yaml/detector.go`: alert construction;
- `pkg/output/alert.go`: `AlertRecord`, `newThreatRecord`,
  `formatMITRESummary`, `formatAlertSequence`;
- `pkg/detectors/yaml/detector_test.go`:
  `TestDetectorPropagatesThreatMetadataToAlert`;
- `pkg/output/alert_test.go`: `TestNewAlertRecordNormalizesDetectorAlert`,
  `TestFormatAlertSequenceIsBounded`.

**Qualification.** ATT&CK identifiers are syntax-validated; the runtime does
not consult an authoritative ATT&CK catalogue or establish semantic
correctness and coverage automatically.

### FR-11: Separate Raw Event, Alert and Diagnostic Output

**Requirement.** The system shall distinguish normalised event telemetry,
detector alerts and operational diagnostics. It shall support JSON and compact
table rendering, including an alerts-only presentation mode.

**Status:** Implemented.

**Evidence:**

- `pkg/output/printer.go`: `Printer`, `NewPrinter`;
- `pkg/output/json.go`, `pkg/output/table.go`, `pkg/output/alert.go`;
- `pkg/config/config.go`: `OutputConfig`, `AlertConfig`;
- `pkg/cmd/cobra/cobra.go`: `--output`, `--alerts`,
  `--alerts-only`, `--alerts-output`;
- `pkg/logging/logger.go`: Zap logger construction;
- `pkg/ebpf/project.go`: `Project.handleRawEvent`, `processDetectors`,
  `WithLogger`.

**Qualification.** Raw events and alerts are written to standard output;
diagnostic logs are emitted separately through Zap to standard error.
Alerts-only changes presentation, not collection or detector evaluation.

### FR-12: Provide a Minimal Kernel-Side UID Filter

**Requirement.** For focused profiles, the user shall be able to discard
events from all but one configured UID before context enrichment,
serialisation and perf-buffer submission.

**Status:** Implemented as one optional equality filter.

**Evidence:**

- `pkg/config/config.go`: `KernelFilterConfig`;
- `pkg/cmd/cobra/cobra.go`: `--kernel-filter-uid-enabled`,
  `--kernel-filter-uid`;
- `pkg/ebpf/project.go`: `configureKernelFilters`;
- `pkg/ebpf/c/common/context.h`:
  `should_drop_by_kernel_uid_filter`, `init_program_data`;
- `pkg/config/config_test.go`: `TestDefaultDisablesKernelUIDFilter`,
  `TestValidateAcceptsKernelUIDFilterForRoot`.

**Qualification.** This is not a complete kernel policy engine and does not
replace user-space policy evaluation.

### FR-13: Package the Executable and eBPF Object Together

**Requirement.** A normal build shall produce a self-contained executable in
which the compiled eBPF object is embedded; a container image may package that
executable for future operational use.

**Status:** Implemented as build and packaging support. Kubernetes deployment
is not implemented or evaluated as a thesis contribution.

**Evidence:**

- `embedded-ebpf.go`: `BPFBundle`;
- `pkg/cmd/initialize/bpfobject.go`: embedded-object fallback;
- `Dockerfile`: builder invokes `make all`, runtime stage copies the executable;
- `README.md`: build and Docker procedures.

## 2. Non-Functional and Operational Requirements

### NFR-01: Target-Kernel Compatibility

The prototype shall be built and validated for Rocky Linux 8.10 with kernel
`4.18.0-553.137.1.el8_10.x86_64`. Enterprise backports mean that capability
decisions must be based on the actual target kernel rather than upstream
version numbers alone.

### NFR-02: CO-RE Portability Within Explicit Limits

BTF and CO-RE shall be used to reduce dependence on kernel structure layouts.
They shall not be presented as providing absent hooks, helpers, program types,
attachment mechanisms, permissions or verifier behaviour.

### NFR-03: Deterministic Detection

For identical normalised events, detector definitions and local state, the
detector path shall apply explicit rule conditions. The system shall not be
described as learning a baseline, producing a statistical anomaly score or
performing general anomaly discovery.

### NFR-04: Bounded Detector State

Collective detection state shall be bounded by short per-detector windows and a
hard capacity. Current implementation limits are five seconds and 4096 partial
matches per detector. Alert deduplication is also bounded by a fixed five-second
engine window.

### NFR-05: Profile-Specific Performance Objective

Steady-state userspace CPU consumption should remain below 5% of one core for
explicitly defined representative profiles. This is an evaluation target, not
a universal property for every event combination, workload or instantaneous
sample. Kernel-side eBPF cost must be evaluated separately.

### NFR-06: Observable Failure Boundaries

Configuration errors, unknown events, detector parse failures, decode errors,
attach failures and lost perf-buffer records shall be surfaced as explicit
errors or structured diagnostic logs.

### NFR-07: Contract Consistency

The C producer and Go consumer shall remain aligned on event identifiers,
header dimensions and argument schemas. Public probe entries shall correspond
to decoder events; support probes shall remain hidden from the public event
API.

### NFR-08: Clear Output Semantics

Event telemetry, alerts and diagnostics shall remain conceptually separate.
Collective alerts shall retain bounded sequence evidence, and JSON shall remain
the complete machine-oriented representation.

### NFR-09: Graceful Lifecycle

The runtime shall stop on context cancellation and attempt ordered cleanup of
perf buffers, links, managed cgroup resources and the loaded module.

### NFR-10: Maintainable Configuration Boundary

CLI options shall populate one validated configuration model. Policies and
detectors shall be loaded from external files without embedding rule logic in
individual eBPF programs.

## 3. Target Environment Assumptions and Compatibility Constraints

| Assumption or constraint | Current design consequence | Evidence |
|---|---|---|
| Rocky Linux 8.10, kernel `4.18.0-553.137.1.el8_10.x86_64` | Hook, helper, BTF and verifier compatibility must be tested on this kernel. | `documentation/report.md`; `documentation/implementation/overview.md` |
| `/sys/kernel/btf/vmlinux` is normally available | This is the default BTF path, but it can be overridden by CLI or environment. | `pkg/config/config.go:Default`; `pkg/cmd/initialize/bpfobject.go:BPFObject` |
| Root or equivalent capabilities are available | Loading eBPF, opening perf events and attaching kprobes/tracepoints require elevated privileges. | runtime commands in `README.md`; `pkg/ebpf/project.go:Init` |
| Vendor backports are possible | Upstream kernel version alone is insufficient evidence for a feature. | Chapter 2 compatibility discussion; `documentation/report.md` |
| BTF/CO-RE are layout-compatibility mechanisms | They do not create missing kernel facilities or bypass permissions and verifier limits. | `chapter2.tex`; CO-RE reads in `pkg/ebpf/c` |
| Perf-event array is the operational transport | Ring-buffer claims must remain background or future-work discussion. | `pkg/ebpf/c/maps.h`; `pkg/ebpf/project.go:Init` |
| Detection is host-local | Process, resource and cgroup correlation keys describe local observations only. | `pkg/detectors/yaml/correlation.go` |
| Cgroup identity is local | A cgroup ID must not be called a Kubernetes workload identity. | `correlationPlan.keys`; `chapter2.tex` |
| Process-tree correlation is shallow | Only the current process and immediate parent relationship are considered. | `correlationPlan.keys`; correlation tests |
| Resource correlation requires stable file fields | Every step using `resource` must expose both `dev` and `inode`; pathnames are not identity keys. | `compileCorrelationPlan`; parser and correlation tests |
| Session identity is unavailable | `user_session` definitions are rejected; UID is not used as a substitute. | `normalizeCorrelationComponent`; `TestParseBytesRejectsUnavailableUserSessionCorrelation` |
| Kubernetes deployment is future context | It must not be presented as implemented, evaluated or novel work in this thesis. | Chapter 1 scope; Chapter 3 dossier |

## 4. Requirement-to-Evidence Matrix

| ID | Requirement | Status | Exact implementation evidence | Existing verification |
|---|---|---|---|---|
| FR-01 | Collect registered process/security events | Implemented, bounded coverage | `pkg/ebpf/c/project.bpf.c`; `pkg/ebpf/probes/probes.go:supportedProbes`; `pkg/events/spec.go` | Probe registry and event registry tests; manual hook procedures in `README.md` |
| FR-02 | Select events and attach dependencies only | Implemented | `probes.Select`; `probes.ConfigureAutoload`; `Spec.impliedBy`; `Spec.internal` | `probes_test.go` selection, dependency and visibility tests |
| FR-03 | Manage eBPF lifecycle | Implemented | `ProjectRunner.Run`; `Project.Init`; `Project.Run`; `Project.Close` | Unit coverage for selection/configuration; target integration deferred |
| FR-04 | Perf-buffer transport and loss diagnostics | Implemented | `maps.h:events`; `events_perf_submit`; `Project.Run` lost channel | Code inspection; loss-under-load evaluation deferred |
| FR-05 | Decode typed event records | Implemented | `EventContext`; `DecodeEvent`; `events.GetSpec` | Decoder tests, including arrays, credentials and long records |
| FR-06 | Apply event, comm and policy filters | Implemented | `Project.handleRawEvent`; `eventEnabled`; `commEnabled`; `policyEnabled` | Command/config and policy unit tests |
| FR-07 | Load and evaluate policies | Implemented, modes partially operational | `policy.LoadFiles`; `Policy.Matches`; `Manager.Match`; `EffectiveEventSelection` | Policy loader, validation, suppress-wins and selection tests |
| FR-08 | Load and dispatch point detectors | Implemented | `loadDetectors`; `NewEngine`; `Dispatcher.Dispatch`; `Detector.OnEvent` | Detector registry, dispatcher and YAML condition tests |
| FR-09 | Evaluate bounded local sequences | Implemented with stated limits | `compileCorrelationPlan`; `Detector.OnEvent`; `maxSequenceStates`; `MaxStateWindow` | Stateful, expiry, process, resource, cgroup, composite and capacity tests |
| FR-10 | Preserve evidence and ATT&CK metadata | Implemented | `detectors.Alert`; `ThreatMetadata`; `AlertRecord`; `formatAlertSequence` | Detector propagation and JSON/table alert tests |
| FR-11 | Separate event, alert and diagnostic channels | Implemented conceptually | `output.Printer`; `WithLogger`; `handleRawEvent`; `logging.New` | Output and logging tests |
| FR-12 | Optional early UID equality filter | Implemented | `KernelFilterConfig`; `configureKernelFilters`; `should_drop_by_kernel_uid_filter` | Config tests and profile benchmarks |
| FR-13 | Embed and package eBPF object | Implemented build support | `embedded-ebpf.go`; `BPFObject`; `Dockerfile` | Build instructions; deployment validation deferred |
| NFR-01 | Target-kernel compatibility | Evaluation pending | CO-RE code and target-specific BTF configuration | Manual target runs documented; systematic matrix deferred |
| NFR-03 | Deterministic rule evaluation | Implemented | YAML condition compiler and detector engine | Operator and sequence tests |
| NFR-04 | Bounded state | Implemented | `MaxStateWindow`; `maxSequenceStates`; dedup window | Bound, expiry and pruning tests |
| NFR-05 | Average CPU below 5% for defined profiles | Partially evidenced | benchmark scripts and stored profile summaries | Several profile runs pass; general and kernel-cost conclusions deferred |
| NFR-06 | Observable errors and losses | Implemented in runtime paths | wrapped errors and Zap logging in `pkg/cmd`, `pkg/ebpf`, decoder and loaders | Error-path unit tests; stress testing deferred |
| NFR-07 | C/Go registry and layout consistency | Implemented with regression checks | C `event_context_t`; Go `EventContext`; probe/event registries | registry tests and long-record regression test |

## 5. Acceptance Evidence

### 5.1 Evidence Already Available

#### Committed unit-test baseline

The following areas pass in a clean archive of commit `163ad72`:

- configuration and command integration;
- event registry and probe registry consistency;
- buffer decoding and typed arguments;
- policy loading, validation, matching and exact event narrowing;
- detector registry, dispatch, deduplication and metrics;
- YAML point and collective evaluation;
- process, immediate process-tree, resource, cgroup and composite correlation;
- state-window and 4096-entry capacity guardrails;
- JSON/table event and alert rendering;
- MITRE ATT&CK metadata propagation;
- Zap logger configuration.

The audit used the relevant package test suite rather than claiming that every
kernel attachment was exercised on the target host.

#### Event-contract regression evidence

Commit `163ad72` replaced an invalid bit-mask treatment of the
non-power-of-two 32000-byte argument buffer with explicit bounds checks in
`pkg/ebpf/c/common/buffer.h`. It also added
`TestDecodeSecurityBprmCheckWithArgumentsBeyond256Bytes`, which verifies a
record whose arguments extend beyond the formerly corrupted offset range.

The shared event context is explicitly represented as 136 bytes in
`pkg/ebpf/c/types.h` and `pkg/bufferdecoder/protocol.go`. Decoder tests verify
the expected wire prefix. Chapter 4 should describe this as an explicit
cross-language contract, not as automatic ABI generation.

#### Stored userspace benchmark evidence

The benchmark suite separates warm-up from steady-state samples and reports
average, P95, peak CPU, peak RSS and thread count. Relevant stored results
include:

| Run/profile | Average CPU | P95 CPU | Peak CPU | Peak RSS | Result |
|---|---:|---:|---:|---:|---|
| 2026-07-24 raw | 4.36% | 7.14% | 9.25% | 46100 KiB | PASS by average |
| 2026-07-24 point | 0.33% | 1.79% | 1.80% | 40724 KiB | PASS |
| 2026-07-24 collective | 3.52% | 5.19% | 8.49% | 44920 KiB | PASS by average |
| 2026-07-24 UID filter | 2.92% | 4.32% | 5.15% | 46560 KiB | PASS by average |
| 2026-07-29 collective | 3.40% | 4.53% | 5.64% | 58772 KiB | PASS by average |

These measurements establish that the userspace target is testable and has
been met by these particular runs. They do not establish a universal
below-5% result, and peak samples may exceed 5%.

### 5.2 Evidence Deferred to Chapter 5

Chapter 5 should provide:

1. a controlled functional test for representative probe families on the exact
   Rocky Linux kernel;
2. repeatable point- and collective-detector scenarios, including negative
   cases and expired sequences;
3. precision-oriented tests for noisy legitimate activity, rather than only
   confirmation that an alert can be produced;
4. repeated benchmark runs with defined workload, event selection, policy and
   detector set, warm-up, duration and CPU topology;
5. separate accounting for Go userspace cost and kernel-side eBPF overhead;
6. event-rate, perf-buffer loss and malformed-record behaviour under load;
7. controlled comparison with and without the UID kernel filter using
   equivalent workloads;
8. memory and thread behaviour over longer runs, especially for stateful
   detectors;
9. validation of container execution only if packaging is evaluated; no
   Kubernetes deployment claim is required;
10. an explicit statement that all-events stress measurements are not the
    representative profile used for the 5% target.

No valid all-events result is currently stored: the available attempts are
empty or marked `ERROR`.

## 6. Conflicts, Obsolete Claims and Aspirational Requirements

### 6.1 Uncommitted Worktree Changes

The current worktree contains:

- modifications to `Makefile`, `README.md`,
  `scripts/benchmark_profiles.sh` and `scripts/benchmark_suite.sh` adding an
  `all-events` benchmark profile;
- modifications to `pkg/ebpf/probes/probes.go` removing `internal: true` from
  six support probes;
- untracked `rules/policies/test_net.yaml`.

These changes are not part of commit `163ad72`.

The current worktree fails:

- `TestPublicProbesHaveDecoderSpecs`, because `sock_alloc_file` becomes public
  without a decoder schema;
- `TestRepositoryPolicyPresetsLoadAndReferenceSupportedEvents`, because the
  untracked policy changes the expected repository preset count from eight to
  nine.

Therefore Chapter 3 must use the committed registry model: support probes are
hidden dependencies unless they are promoted together with a stable event ID
and decoder schema. The all-events profile is experimental and has no valid
acceptance result yet.

### 6.2 Policy Semantics Need Qualification

The policy model distinguishes `monitor`, `detect` and `suppress`, but only
suppression currently changes continuation semantics distinctly. Any matching
non-suppress policy permits the event to reach both output and detector
processing. Detector selection is path-based and separate from policies.

Safe claim: policies define event scope and may reduce probe selection when
their allowlists are exact.

Unsafe claim: detect policies directly enable particular detectors or monitor
policies bypass detection.

### 6.3 Policy Provenance Is Not Fully Wired Into Alerts

`AlertRecord` can display policy names when detector metadata contains
`policy_names` or `policy`. The runtime does not currently inject the matched
policy result into detector alerts. Policy provenance in alert output is
therefore a prepared output capability, not an end-to-end implemented
requirement.

### 6.4 Detector Requirements Do Not Automatically Select Probes

Detector definitions declare consumed and required events, but their event
requirements are not automatically merged into CLI/policy probe selection.
Operators must configure a compatible policy/event scope. Chapter 3 should not
claim automatic dependency planning from detectors to eBPF probes.

### 6.5 Runtime Flush Is Not Periodically Scheduled

The detector interface, dispatcher and engine expose `Flush`, and YAML
detectors prune stale state while processing incoming events. The main runtime
does not currently schedule periodic engine flushes. Expired partial state is
therefore removed on later detector traffic or explicit flush, not by an
independent runtime timer.

### 6.6 Correlation Is Deliberately Narrow

The following would be obsolete or incorrect descriptions:

- "process-tree correlation reconstructs process ancestry";
- "resource correlation follows files by pathname";
- "cgroup correlation identifies Kubernetes workloads";
- "collective detection is a general complex-event-processing engine";
- "UID is used as a user-session identity".

The implementation provides local keys and short ordered sequences only.

### 6.7 ATT&CK Mapping Is Classification, Not Automatic Validation

The implementation validates ATT&CK identifier formats and preserves metadata.
It does not verify names against an ATT&CK release, prove detector correctness,
or establish complete technique coverage. This does not reduce its utility:
the mapping provides a consistent security vocabulary and preserves
traceability from a detector to the behaviour it is intended to represent.

### 6.8 Documentation Claims That Are Stale

- `documentation/implementation/overview.md` says minimal kernel-side filters
  are still missing. A single-UID early filter is now implemented; richer
  filtering is still future work.
- `README.md` roadmap text referring to adding the UID filter and initial
  process-tree grouping is obsolete. Both are present.
- Some older report/roadmap passages describe collective detectors as a next
  phase. Point and bounded collective detectors are now implemented.
- Documentation that implies policy names are already attached to runtime
  alerts should be qualified as described in Section 6.3.
- Detailed Kubernetes deployment language is operational context only and
  should not enter Chapter 3 as an implemented contribution.

### 6.9 Features That Remain Aspirational

- enforcement or event blocking;
- machine-learning or statistical anomaly detection;
- persistent process graphs and recursive ancestry;
- cluster-wide contextual correlation;
- Kubernetes workload attribution;
- a general complex-event-processing engine;
- a complete kernel-side policy engine;
- operational BPF ring-buffer transport;
- automatic detector-to-probe dependency planning;
- universal below-5% CPU behaviour;
- complete ATT&CK or Linux event coverage.

## 7. Safe Academic Wording

### 7.1 Proposed Wording for Section 3.1

> The system was designed for a constrained and explicitly identified
> execution environment: Rocky Linux 8.10 with the enterprise kernel
> 4.18.0-553.137.1.el8_10.x86_64. This target combines an older upstream kernel
> lineage with vendor backports; consequently, compatibility cannot be inferred
> from the version number alone. The availability of BTF, individual hooks,
> eBPF helpers, attachment mechanisms and verifier behaviour must be established
> on the deployed kernel. CO-RE is used to reduce dependence on kernel data
> structure layouts, but it cannot supply kernel facilities that are absent
> from the target.
>
> Functionally, the prototype is required to collect a selectable set of
> process- and security-relevant kernel events, transport them to a Go
> userspace runtime, decode them through a typed event contract, and apply
> configurable event policies and deterministic detectors. Policies constrain
> the monitoring scope, while detectors evaluate either individual events or
> short ordered sequences of related local events. Alerts retain the evidence
> that satisfied a detector and may include MITRE ATT&CK metadata to express
> the intended security interpretation in a shared vocabulary. Event
> telemetry, detector alerts and operational diagnostics are treated as
> separate outputs.
>
> The design also imposes operational constraints. Collective state must remain
> bounded, malformed records and lost transport samples must be observable, and
> probe selection should avoid loading and attaching programs that are not
> required by the active event scope. A steady-state userspace CPU target below
> five per cent of one core is evaluated for explicitly defined representative
> profiles; it is not assumed to hold for every workload, every event
> combination or every instantaneous sample. The prototype performs
> observation and detection rather than enforcement, and its correlation scope
> is local to one monitored host.

### 7.2 Proposed Wording for Section 3.7

> Several boundaries follow directly from the target environment and the
> intended experimental scope. Kernel-space programs are responsible for
> observation, minimal extraction, optional early UID filtering and event
> serialisation. The Go runtime owns lifecycle management, decoding, event
> selection, policy evaluation, detector dispatch, local sequence correlation,
> alert generation, presentation and diagnostic logging. This division keeps
> rule evolution outside individual eBPF programs and limits the amount of
> verifier-sensitive logic executed in the kernel.
>
> Perf buffers are used as the operational transport because they are supported
> by the target environment; the BPF ring buffer is not part of the implemented
> runtime. Public logical events are represented through explicit probe and
> decoder registries, whereas internal programs may be attached only as
> dependencies of selected events. Policies and detectors remain separate:
> policies define the event domain, and detectors independently express the
> conditions that can produce alerts.
>
> Collective detection is intentionally local and bounded. A detector may
> relate events through stable process identity, an immediate parent-child
> relationship, file-resource identity, local cgroup identity, or a composite
> of these dimensions. Partial matches are retained only within short windows
> and under a fixed per-detector capacity. This design does not construct a
> persistent process graph, infer cluster-wide context or implement a
> general-purpose complex-event-processing engine. Likewise, the prototype
> reports observed behaviour but does not block it. Container packaging and a
> possible future deployment in a wider orchestration environment are
> operational considerations rather than contributions evaluated by this
> thesis.
