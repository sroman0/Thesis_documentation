# Chapter 2 Technical Positioning Audit

## Scope and assessment method

This audit checks the proposed positioning for Section 2.5 against the current
implementation and the available technical documentation. The labels have the
following meanings:

- **Verified**: the claimed capability is directly represented in the current
  code and connected to the runtime pipeline.
- **Partially verified**: a bounded implementation exists, but the broader
  interpretation of the claim requires qualification or experimental evidence.
- **Unsupported**: the repository does not currently provide sufficient
  implementation or evaluation evidence for the claim.

The wording below is intentionally conservative. Section 2.5 should position
the work at a high level; detailed architecture belongs in Chapter 3, concrete
symbols and algorithms in Chapter 4, and measured results in Chapter 5.

## Claim-by-claim audit

### 1. Rocky Linux 8.10 and the enterprise 4.18 kernel target

**Assessment: Verified as the declared and implementation-oriented target;
partially verified as a compatibility claim.**

**Evidence**

- `pkg/config/config.go`: `Default` selects `/sys/kernel/btf/vmlinux` and
  documents the Rocky Linux 8.10 target.
- `pkg/cmd/initialize/bpfobject.go`: BTF discovery and fallback logic.
- `pkg/ebpf/c/vmlinux.h`, `vmlinux_flavors.h`, and `vmlinux_missing.h`:
  target-aware kernel types, compatibility flavours, and missing constants.
- `pkg/ebpf/c/common/arch.h`: x86-64 syscall definitions.
- `pkg/ebpf/c/common/capabilities.h`: compatibility branch for the older
  credential layout used by the target kernel.
- `Makefile`: `ARCH = x86`, `LINUX_ARCH = x86`, `-D__TARGET_ARCH_x86`, and
  `-D__x86_64__`.
- `documentation/implementation/hooks.md`: hook-specific target decisions and
  verifier mitigations.
- `Thesis/content/chapters/chapter1.tex`: records the exact target as
  `4.18.0-553.137.1.el8_10.x86_64`.

**Safe academic wording**

> The prototype was developed for Rocky Linux 8.10 on the
> `4.18.0-553.137.1.el8_10.x86_64` enterprise kernel. This target motivated
> explicit attention to vendor backports, available attachment points, BTF
> information, kernel data layouts, and verifier behaviour.

This wording establishes a target, not portability to every Rocky Linux,
RHEL-compatible, 4.18, or upstream kernel.

**Defer to Chapters 3 or 4**

- the exact BTF discovery order;
- generated and compatibility headers;
- x86-64 compilation flags;
- hook-specific fallbacks and verifier-oriented code changes;
- attach-time handling of unavailable symbols or tracepoints.

**Evidence still required for Chapter 5**

- machine and kernel identity captured with reproducible commands;
- successful clean build, load, and attach results on the stated target;
- a compatibility matrix for the evaluated event set;
- explicit failures or unavailable hooks on that kernel;
- optionally, comparison with at least one second kernel if portability is
  evaluated.

### 2. Kernel-to-user-space event contract

**Assessment: Verified.**

**Evidence**

- `pkg/ebpf/c/types.h`: `task_context_t`, `event_context_t`,
  `args_buffer_t`, `event_data_t`, `EVENT_ID_LIST_COMMON`, and
  `MAX_EVENT_SIZE`.
- `pkg/ebpf/c/common/buffer.h`: `events_perf_submit` and argument
  serialization helpers.
- `pkg/ebpf/c/common/context.h`: `init_program_data`, which initializes the
  shared event context before submission.
- `pkg/bufferdecoder/decoder.go`: `EbpfDecoder.DecodeContext`, including the
  explicit 136-byte layout mirrored from C.
- `pkg/bufferdecoder/eventsreader.go`: `DecodeEvent`, `decodeArguments`, and
  typed argument-record decoding.
- `pkg/events/ids.go`: Go-side event identifiers.
- `pkg/events/spec.go`: event names and ordered typed argument specifications.
- `pkg/events/spec_test.go`, `pkg/ebpf/probes/probes_test.go`, and
  `pkg/bufferdecoder/eventsreader_test.go`: registry and decoding checks.

The contract has also exposed real schema-drift failures during development,
including the earlier mismatch caused by extending `event_context_t` without
updating the Go context size. A later long-record regression showed that
masking an argument offset with `ARGS_BUF_SIZE - 1` was invalid because the
32000-byte buffer is not a power of two. Commit `163ad72` replaced that mask
with an explicit bounds check and added a decoder regression test. These fixes
are evidence of a maintained contract, not proof that future drift is
impossible.

**Safe academic wording**

> Kernel programs and the Go runtime share an explicit binary event contract.
> Event identifiers, a fixed context layout, indexed argument records, and
> user-space event specifications are kept aligned so that records emitted
> through the perf buffer can be decoded into typed events.

**Defer to Chapters 3 or 4**

- byte offsets and the 136-byte context layout;
- event-ID generation and registry invariants;
- indexed argument record formats;
- handling of strings, arrays, credentials, socket addresses, and pointers;
- the historical schema and long-record offset mismatches and their fixes.

**Evidence still required for Chapter 5**

- automated contract-test results;
- end-to-end examples across representative scalar and structured arguments;
- malformed, truncated, and unknown-event behaviour;
- perf-buffer lost-event measurements under load;
- evidence that the evaluated object and Go binary were built from the same
  source revision.

### 3. Go and `libbpfgo` runtime

**Assessment: Verified.**

**Evidence**

- `go.mod`: Go 1.22 module, `github.com/aquasecurity/libbpfgo`, and a local
  replacement under `3rdparty/libbpfgo`.
- `pkg/ebpf/project.go`: `Project`, `New`, `Init`, `Run`, `Close`,
  `configureKernelFilters`, and `attachPrograms`.
- `pkg/ebpf/probes/probes.go`: probe registry and attachment through
  `libbpfgo`.
- `pkg/cmd/project.go`: policy, detector, logger, and runtime wiring.
- `Makefile`: static `libbpf` build, CGO flags, eBPF compilation, and Go
  executable build.

**Safe academic wording**

> The user-space runtime is implemented in Go and uses `libbpfgo`, the Go
> bindings to `libbpf`, to load the eBPF object, configure maps, attach selected
> programs, consume perf-buffer records, and coordinate decoding and detection.

**Defer to Chapters 3 or 4**

- runner lifecycle and cleanup;
- selective autoload and probe dependencies;
- CGO and static `libbpf` build details;
- logging, output printers, and perf-buffer channel sizes.

**Evidence still required for Chapter 5**

- clean-build reproducibility;
- startup, shutdown, and failure-path tests;
- CPU, RSS, thread count, and lost-event observations;
- distinction between Go runtime cost, output cost, and kernel-side cost.

### 4. Policies and YAML detectors

**Assessment: Verified, within the current rule language.**

**Evidence**

- `pkg/policy/policy.go`: `Policy`, `Scope`, `Rule`, `EventSelector`,
  `Policy.Matches`, modes, and intents.
- `pkg/policy/loader.go`: `LoadFiles` and YAML conversion.
- `pkg/policy/manager.go`: `Manager.Match`, `IsEventSelected`, and
  `EffectiveEventSelection`.
- `pkg/detectors/yaml/schema.go`: external detector schema.
- `pkg/detectors/yaml/parser.go`: parsing, normalization, event-reference
  validation, and threat-metadata conversion.
- `pkg/detectors/yaml/detector.go`: executable YAML-backed detector.
- `pkg/detectors/registry.go`, `dispatch.go`, and `engine.go`: detector
  registration, event-indexed dispatch, evaluation, deduplication, and metrics.
- `rules/policies/*.yaml` and `rules/detectors/*.yaml`: executable examples.
- `pkg/cmd/project.go`: end-to-end runtime integration.

The policy language currently selects events and applies `comm` and UID scope.
The detector language supports a finite set of deterministic operators. It is
not a general policy language, an enforcement system, or a statistical model.

**Safe academic wording**

> The prototype separates monitoring scope from detection logic. YAML policies
> select relevant event families and optional process constraints, while YAML
> detectors define deterministic predicates or short ordered sequences over
> normalized events.

**Defer to Chapters 3 or 4**

- policy modes and intents;
- exact YAML schemas and operators;
- validation and normalization;
- event allowlists compiled into program autoload and attachment selection;
- registry, dispatcher, engine, and alert deduplication.

**Evidence still required for Chapter 5**

- valid and invalid configuration tests;
- policy-selection and detector-match test cases;
- false-positive analysis for the supplied rule set;
- cost comparison with and without policies and detectors;
- usability assessment of the YAML workflow.

### 5. Point and bounded collective detections

**Assessment: Verified as deterministic point detections and a bounded
collective-sequence MVP.**

**Evidence**

- `pkg/detectors/yaml/detector.go`: `Detector.OnEvent` emits a point alert for a
  stateless match; `onStatefulEvent` advances ordered sequence state.
- `pkg/detectors/definition.go`: `DefaultStateWindow` is two seconds and
  `MaxStateWindow` is five seconds; `ValidatePerformanceBounds` rejects longer
  windows.
- `rules/detectors/root_exec.yaml`,
  `sensitive_file_open.yaml`, `privileged_uid_change.yaml`, and
  `kernel_module_activity.yaml`: point detectors.
- `rules/detectors/privilege_exec_chain.yaml`,
  `privilege_sensitive_file_chain.yaml`, `memfd_exec_chain.yaml`, and
  `kernel_module_kprobe_chain.yaml`: two-step collective detectors with
  five-second windows.
- `pkg/output/alert.go`: `formatAlertSequence` and structured multi-event alert
  output.

**Safe academic wording**

> The current detector layer supports deterministic matches on individual
> events and short ordered multi-event patterns. Collective state is retained
> locally for bounded windows of at most five seconds.

In Chapter 2, “point detection” and “short collective event pattern” are safer
than an unqualified claim of general anomaly detection.

**Defer to Chapters 3 or 4**

- sequence state representation;
- pruning interval;
- step-matching semantics;
- concrete detector definitions and alert formatting.

**Evidence still required for Chapter 5**

- controlled positive and negative tests for every detector;
- timeout, interleaving, restart, and repeated-sequence cases;
- memory growth and state-cardinality measurements;
- precision or false-positive observations on representative benign activity;
- evidence that selected MITRE mappings match the tested behaviours.

### 6. Configurable local short-window correlation

**Assessment: Verified within explicit local bounds.**

**Evidence**

- `pkg/detectors/yaml/correlation.go` compiles the configured `group_by`
  strategies before events reach the hot path.
- `process` uses host PID and start time, while `process_tree` additionally
  checks the immediate parent identity carried by the event context.
- `resource` uses typed `dev` and `inode` arguments and can correlate different
  processes accessing the same local file object.
- `cgroup` uses the local cgroup identifier from the normalised event context.
- Ordered `group_by` lists form composite keys whose components must all match.
- `pkg/detectors/yaml/correlation_test.go` covers PID reuse, immediate
  parent-child matching, cross-process resources, false resource matches,
  cgroups, missing fields and composite keys.
- Time-based pruning and the limit of 4096 incomplete states per detector bound
  local retention.

The implementation does not maintain a complete or persistent process tree.
It does not correlate arbitrary descendants, reconstruct historical ancestry,
or use cluster-wide identity. `user_session` is not silently approximated with
UID and remains unavailable until a stable session identifier is collected.
Resource and cgroup identities are local observations and do not represent
Kubernetes workload identity.

**Safe academic wording**

> Short sequences can be correlated through stable process identity, immediate
> parent relationships, file-resource identity, local cgroup identity, or
> composite combinations of these dimensions. This provides bounded local and
> selected cross-process correlation without maintaining persistent or
> cluster-wide state.

**Defer to Chapters 3 or 4**

- compilation of detector-specific correlation plans;
- process and parent key composition;
- `dev:inode` resource identity and schema validation;
- cgroup and composite-key semantics;
- sequence replacement, pruning and state-cap behaviour.

**Evidence still required for Chapter 5**

- same-process, direct parent-child, unrelated-process, and PID-reuse tests;
- concurrent and overlapping sequence tests;
- measurement of retained keys and memory use;
- explicit demonstration of the boundary beyond immediate parent-child cases.

### 7. MITRE ATT&CK metadata

**Assessment: Verified as metadata validation and propagation; unsupported as
proof of coverage or detector correctness.**

**Evidence**

- `pkg/detectors/definition.go`: `ThreatMetadata`, `AttackTactic`,
  `AttackTechnique`, `ThreatMetadata.Validate`, and `IsMapped`.
- Validation checks the syntactic form of tactic and technique IDs.
- `pkg/detectors/yaml/schema.go` and `parser.go`: YAML threat fields and
  conversion into the internal model.
- `pkg/detectors/yaml/detector.go`: threat metadata is copied into emitted
  alerts.
- `pkg/output/alert.go`: `newThreatRecord` exposes structured JSON metadata and
  `formatMITRESummary` produces compact table output.
- Current detector YAML files declare tactics, techniques, data sources, and
  data components.

The implementation does not query an authoritative ATT&CK dataset, validate
ID/name pairs, calculate coverage, or establish that a detector is a correct
implementation of a technique.

**Safe academic wording**

> Detector definitions can associate alerts with MITRE ATT&CK tactics,
> techniques, data sources, and data components. The runtime validates the
> identifier format and propagates this metadata to table and JSON alert output,
> providing a consistent vocabulary for downstream interpretation.

**Defer to Chapters 3 or 4**

- YAML threat schema;
- identifier regular expressions;
- conversion and output structures;
- the mappings selected for each demonstration detector.

**Evidence still required for Chapter 5**

- manual review of every ID, name, and detector-to-technique mapping against the
  selected ATT&CK version;
- a detector-to-ATT&CK coverage table;
- confirmation that data-source terminology is valid for that version;
- positive test output showing metadata propagation.

### 8. Kernel-side UID filtering

**Assessment: Verified as a minimal optional exact-match filter.**

**Evidence**

- `pkg/config/config.go`: `KernelFilterConfig` with `UID` and `UIDEnabled`.
- `pkg/cmd/cobra/cobra.go`: `--kernel-filter-uid` and
  `--kernel-filter-uid-enabled`.
- `pkg/ebpf/project.go`: `configureKernelFilters` patches the `config_map`
  value after object loading.
- `pkg/ebpf/c/types.h`: `config_entry_t.kernel_uid_filter_enabled` and
  `kernel_uid_filter`.
- `pkg/ebpf/c/common/context.h`:
  `should_drop_by_kernel_uid_filter` compares the current effective UID, and
  `init_program_data` applies the guard before context enrichment, argument
  capture, and submission.

The filter is one exact UID allow condition. It is not a general policy
compiler and does not support UID sets, PID, command name, namespace, container,
or event-specific predicates.

**Safe academic wording**

> An optional kernel-side pre-filter can retain events generated by one
> configured effective UID. The check runs in the common event-initialization
> path before full context enrichment and perf-buffer submission, while richer
> policy and detector logic remains in user space.

**Defer to Chapters 3 or 4**

- raw `config_map` layout offsets;
- CLI wiring and map update;
- placement inside `init_program_data`;
- semantics for events whose subject differs from the current task.

**Evidence still required for Chapter 5**

- functional tests with matching and non-matching UIDs;
- event-count reduction before and after filtering;
- CPU, RSS, and lost-event comparison under the same workload;
- clarification of workloads dominated by root or other UIDs;
- validation that all evaluated event paths pass through the common guard.

### 9. Performance target and current benchmark limitations

**Assessment: Partially verified as a measurement framework and preliminary
result; unsupported as a general “below 5%” result.**

**Evidence**

- `scripts/benchmark_userspace.sh`: samples process CPU from `/proc`, reports
  RSS and thread count, excludes a configurable warm-up, calculates average,
  p95, and peak CPU, and applies a default 5% average threshold.
- `scripts/benchmark_profiles.sh`: defines `raw`, `point`, `collective`,
  `kernel-filter-uid`, and `all-events` profiles.
- `scripts/benchmark_workload.sh`: repeatable local process, file, and
  privileged-operation workload.
- `scripts/benchmark_suite.sh`: starts each profile independently, records the
  exact child PID, captures summaries, and fails profiles whose average exceeds
  the threshold.
- Preliminary summaries under `tmp/benchmarks/20260724T070129Z` and
  `20260724T082845Z` report average CPU below 5% for the completed `raw`,
  `point`, `collective`, and `kernel-filter-uid` runs. Several p95 and peak
  values exceed 5%.
- The available `all-events` attempts are empty or marked `ERROR`.

These measurements are encouraging but not sufficient for a general
performance claim. They are workload-sensitive, include limited repetitions,
measure the Go process rather than total kernel-side overhead, and do not yet
provide a stable all-events result, baseline, confidence intervals, event rate,
or lost-event rate.

**Safe academic wording**

> Keeping steady-state user-space CPU consumption below 5% of one core is an
> evaluation target for the operational profile. Preliminary profiling
> infrastructure separates warm-up from measurement and compares raw, point,
> collective, and UID-filtered configurations; definitive results are reserved
> for the experimental evaluation.

**Defer to Chapters 3 or 4**

- benchmark script architecture may be summarized in Chapter 4;
- the rationale for bounded windows and early filtering belongs in Chapter 3;
- no numerical result should be developed in Chapter 2.

**Evidence still required for Chapter 5**

- a formally selected operational and worst-case profile;
- completed 120-second or longer runs with multiple repetitions;
- baseline system measurements and a no-detector comparison;
- average, p95, peak, variability, and confidence intervals;
- CPU normalized to one core, RSS, threads, event throughput, and lost events;
- separate assessment of output enabled versus discarded;
- working `all-events` measurements or a documented reason for excluding that
  profile;
- total-system or kernel-side overhead methodology, not only process CPU;
- controlled comparison of UID filtering with identical input workloads.

## Overclaim risks in the Chapter 2 material

| Candidate statement or implication | Risk | Safer treatment |
|---|---|---|
| “Target Rocky Linux 8.10 with an enterprise 4.18 kernel and backports” implies broad 4.18 portability | Portability | State the exact tested target. Explain that vendor backports motivated target-aware design; do not generalize to every 4.18 kernel. |
| CO-RE makes the implementation portable | Portability | Say that BTF and CO-RE reduce data-layout coupling. Hook availability, helper support, tracepoints, symbols, and verifier behaviour remain target-dependent. |
| “A coherent kernel-to-Go event contract” implies that mismatch is impossible | Completeness | Describe an explicit mirrored and tested contract. Acknowledge that schema evolution still requires synchronized C and Go changes. |
| Policies are a complete filtering system | Completeness | Current policies select event names and optional UID/command scope; exact event allowlists can also reduce autoload and attachment. Rich content predicates remain detector-side. |
| YAML detectors constitute general anomaly detection | Novelty and scope | Call them deterministic rule-based point detections and short collective event patterns. Do not imply machine learning, learned baselines, or contextual anomaly detection. |
| `process_tree` means complete process-tree correlation | Completeness | Qualify it as same-process and immediate parent-child correlation derived from local event context. |
| Bounded state “controls” or “guarantees low overhead” | Performance | Bounded windows constrain retention by design; their performance effect must be measured in Chapter 5. |
| MITRE ATT&CK mapping validates detections | Correctness | State that metadata supplies a common classification vocabulary. Mapping correctness and coverage require manual evaluation. |
| Kernel-side filtering is policy offload | Completeness | Describe one optional effective-UID exact-match pre-filter. Rich policy evaluation remains in user space. |
| The prototype remains below 5% of one core | Performance | Keep 5% as the evaluation target. Preliminary profile averages are not a universal or final result. |
| The event set provides comprehensive process and host-security monitoring | Completeness | Describe the implemented event families as selected coverage. Define evaluated coverage in Chapter 5 rather than using “comprehensive”. |
| The system is production-ready because it uses `libbpf`, Go, YAML, or structured logging | Production readiness | Call it a research prototype. Production readiness would require broader compatibility, long-duration reliability, operational hardening, and deployment evidence. |
| Similarity to Tracee establishes equivalent capability | Novelty and completeness | Present Tracee as an architectural reference. Compare design boundaries without claiming functional parity. |
| Kubernetes deployment is an implemented contribution | Scope | Mention only that external orchestration is possible future context. Do not claim that Kubernetes deployment, cluster-wide correlation, or centralized contextual detection was implemented or evaluated. |

## Documentation consistency findings

The following internal inconsistencies should not be carried into the thesis:

- `documentation/thesis/terminology-and-style.md` describes the ring buffer as a
  channel retained for versatility. The current implementation does not define
  or initialize a project ring-buffer map: `pkg/ebpf/project.go` opens only the
  `events` perf buffer, and `documentation/implementation/hooks.md` states that
  the ring-buffer map and submission helper were removed. Chapter 2 may explain
  perf buffers and ring buffers as general eBPF primitives, but Chapters 3 and 4
  should identify the perf buffer as the only implemented event transport.
- The opening of `documentation/implementation/overview.md` says that advanced
  stateful correlation remains outside the implementation. Later sections and
  the code show that a bounded sequence MVP is implemented. The accurate
  distinction is between the existing short sequence engine and the absent
  complete, persistent process graph.
- The final “missing” list in the same overview still mentions future minimal
  kernel-side filters, although the exact effective-UID filter is already
  implemented. The missing work concerns richer filters and definitive
  evaluation, not the first UID filter itself.
- Some comments call the detector mapping or policy fields “novelty”. These
  labels are development notes, not research evidence. Chapter 2 must treat
  novelty as a candidate conclusion to be supported by related-work comparison
  and experimental evaluation.

## Recommended Section 2.5 positioning paragraph

The following wording is supported by the present implementation while keeping
the details in later chapters:

> The proposed work occupies a deliberately bounded part of the Linux runtime
> security design space. It targets process and host-security monitoring on
> Rocky Linux 8.10 and its enterprise 4.18-based kernel, where vendor backports
> and target-specific verifier behaviour require explicit compatibility
> decisions. Kernel observations are transported through an explicit binary
> contract and normalized by a Go runtime built on `libbpfgo`. Declarative
> policies select the monitoring scope, while YAML detectors evaluate
> deterministic predicates over individual events or short ordered event
> patterns. The current collective model retains local state for bounded
> windows and supports same-process or immediate parent-child correlation.
> Alerts may carry MITRE ATT&CK metadata as a common classification vocabulary.
> A minimal effective-UID pre-filter can reduce kernel-to-user-space traffic,
> whereas richer selection and detection remain in user space. Performance,
> compatibility breadth, detection quality, and the effect of early filtering
> are evaluated separately rather than assumed from the architecture.

## Final boundary for Chapter 2

Section 2.5 may claim that the listed mechanisms exist and explain how they
differentiate the prototype's scope. It should not include C structure layouts,
Go symbol names, YAML field-by-field descriptions, probe registries, map
offsets, detector algorithms, benchmark numbers, or debugging history. Those
details belong to Chapters 3 and 4, while Chapter 5 must establish compatibility,
functional correctness, detection behaviour, and performance through
reproducible experiments.
