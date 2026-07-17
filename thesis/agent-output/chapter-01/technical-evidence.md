# Chapter 1 Technical Evidence Audit

## Audit scope

This audit checks the project-specific claims proposed for Chapter 1 against the
current `demo_project` implementation and the supporting documentation. The
audited repository state is commit `91de509` plus an uncommitted working tree.
The uncommitted changes include the first kernel-side UID filter, registry
consistency tests, detector-dispatch metrics, and benchmark tooling. These
features exist in the inspected implementation, but should not be described as
part of a stable release until they are committed and tagged.

The following verification command completed successfully:

```text
GOCACHE=/tmp/go-build go test ./pkg/logging ./pkg/config ./pkg/events \
  ./pkg/bufferdecoder ./pkg/policy ./pkg/detectors/... ./pkg/output
```

This is a source and unit-test audit. It does not replace live validation on the
target kernel, a complete performance experiment, or a Kubernetes deployment
test.

## Evidence matrix

| Claim | Status | Exact evidence | Safe wording for Chapter 1 | Details for Chapters 3-5 | Conflicts or caveats |
|---|---|---|---|---|---|
| Vesuvius targets Rocky Linux 8.10 on `4.18.0-553.109.1.el8_10.x86_64`, x86_64. | **Verified as the declared and tested target** | `demo_project/README.md`, **Requirements** table; `documentation/implementation/performance.md`, **Result of 10 June 2026**; `documentation/daily/2026-05-19.md`, recorded `uname -a`. Build defaults are in `pkg/config/config.go`, `Default`. | “The prototype was developed for and evaluated primarily on Rocky Linux 8.10 with kernel `4.18.0-553.109.1.el8_10.x86_64` on x86_64.” | Chapter 3 should explain RHEL backports, BTF availability, CO-RE, and target-driven hook adaptations. Chapter 5 should state the exact test environment. | Do not generalize this into compatibility with every Rocky Linux, RHEL, 4.18, kernel, or architecture. The target is documented rather than enforced by a runtime compatibility gate. |
| The project implements an eBPF-to-userspace event pipeline. | **Verified** | `pkg/ebpf/project.go`: `Project`, `New`, `Init`, `Run`, `handleRawEvent`, `processDetectors`; `pkg/bufferdecoder/eventsreader.go`: `DecodeEvent`; `pkg/output`: printer implementations. | “Vesuvius implements a pipeline that attaches selected eBPF programs, transports event records to Go userspace, decodes them, applies event and policy selection, evaluates configured detectors, and emits events or alerts.” | Chapter 3: component architecture and data flow. Chapter 4: object loading, map/buffer initialization, attach lifecycle, decoding, filtering order, and output. | The pipeline is implemented, but “complete pipeline” is ambiguous and should not imply production completeness, universal event coverage, or guaranteed losslessness. |
| Perf buffer is the current event transport, while ring buffer remains available. | **Verified, with stale comments** | `pkg/ebpf/project.go`, `Init`: opens `events_ringbuf` and `events`; comments at `Init` identify perf as the path used by currently attached event hooks. `Run` consumes both. `pkg/ebpf/c/maps.h` defines both maps. | “The current hooks submit through a perf buffer; ring-buffer support is retained in the runtime for compatibility and future experiments.” | Chapter 4 should document map types, channel sizes, lost-event reporting, and why both consumers are initialized. | `pkg/ebpf/project.go`, the `Project` field comment near lines 31-33, still says simpler hooks use ring buffer and networking hooks use perf buffer. This conflicts with the later implementation comment and transport decision documentation. Do not repeat the stale claim. |
| The tool covers process and security hook families. | **Verified as broad, selected coverage** | `pkg/ebpf/c/project.bpf.c` contains 131 `SEC` declarations: 66 tracepoints, 16 raw tracepoints, 43 kprobes, 4 kretprobes, and 2 cgroup-skb programs. Public attachment metadata is in `pkg/ebpf/probes/probes.go`, `supportedProbes`; event schemas are in `pkg/events/spec.go`. | “The prototype provides selected coverage across process lifecycle and execution, credentials and privilege changes, filesystems, memory operations, namespaces and mounts, kernel modules and instrumentation, eBPF-related operations, cgroups, and selected networking activity.” | Chapter 4 or an appendix should list public events, support probes, attach type, payload, and target-kernel adaptations. | Program count is not event count: some programs are internal dependencies, entry/return pairs, or alternate producers for one logical event. Do not call coverage complete or equivalent to Tracee. Networking attribution also remains a thesis-scoping decision. |
| A coherent event/probe registry exists. | **Verified in the current working tree** | `pkg/ebpf/probes/probes.go`: `Spec`, `EmitsEvent`, `supportedProbes`, `Select`, `SupportedEventNames`, `Attach`; `pkg/events/ids.go`; `pkg/events/spec.go`. Bidirectional checks are in `pkg/ebpf/probes/probes_test.go` and `pkg/events/spec_test.go`. | “A static registry aligns public event names, eBPF program attachment metadata, event identifiers, and userspace decoder schemas, while hiding support-only probes from the CLI event contract.” | Chapter 3: rationale for public events versus internal dependencies. Chapter 4: registry invariants and attach routing. Chapter 5: registry test results. | The stronger bidirectional registry tests are uncommitted. The C event IDs and Go definitions are still maintained manually rather than generated from one source, so drift is detected by tests rather than made impossible by construction. |
| The decoder normalizes the custom C event format into typed Go events. | **Verified** | `pkg/bufferdecoder/protocol.go`: `EventContext`, `Event`; `pkg/bufferdecoder/eventsreader.go`: `DecodeEvent`, argument-record decoders; `pkg/bufferdecoder/event_args.go`: typed argument model; `pkg/events/spec.go`: event argument schemas. | “Userspace decodes a fixed event context and schema-driven argument records into a common Go event model used by output, policy matching, and detectors.” | Chapter 4 should describe the 136-byte context contract, argument record format, supported payload types, bounds checks, and unknown-ID handling. | The decoder is tested and supports scalar, string, string-array, argument-array, credential, and socket-address payloads, but “robust” should be reserved for Chapter 5 evidence such as malformed-input, fuzz, and compatibility testing. `MatchedPolicies` in the wire context still carries a TODO and is not the active policy path. |
| YAML policies select relevant events and scopes. | **Verified, but narrower than a full policy control plane** | `pkg/policy/policy.go`: `Policy`, `PolicyMode`, `Scope`, `EventRules`; `pkg/policy/loader.go`; `pkg/policy/manager.go`: `Match`, `IsEventSelected`; `pkg/ebpf/project.go`: `policyEnabled`; policy examples in `rules/policies`. | “YAML policies provide userspace event selection and limited scoping by event name, command name, and UID before detector evaluation.” | Chapter 3: policy model and precedence. Chapter 4: loading, validation, monitor/detect/suppress matching, and runtime filter order. | The runtime calls `IsEventSelected` only. A matched policy does not currently select a detector, configure kernel maps, or reliably propagate policy provenance into alerts. Therefore, do not say that policies create detectors or form a Tracee-equivalent control plane. |
| YAML detectors can produce point alerts. | **Verified** | `pkg/detectors/yaml/parser.go`; `pkg/detectors/yaml/detector.go`: stateless `OnEvent`, condition matching, `newAlertFromEvents`; rules `root_exec.yaml`, `sensitive_file_open.yaml`, `privileged_uid_change.yaml`, and `kernel_module_activity.yaml`. | “Stateless YAML detectors evaluate conditions on individual decoded events and produce structured alerts.” | Chapter 4 should document the schema, operators, field resolution, validation, and examples. Chapter 5 should report controlled positive and negative tests and false-positive observations. | These are deterministic rule matches, not learned anomaly models or proof that an event is malicious. Some point rules, especially root execution and sensitive-file access, have shown benign alert volume and require tuning. |
| The detector engine supports local collective anomalies. | **Verified for ordered short-window sequences** | `pkg/detectors/definition.go`: stateful mode and 2-second default/5-second maximum windows; `pkg/detectors/yaml/detector.go`: `onStatefulEvent`, `pruneExpiredState`, sequence state; collective rules `privilege_exec_chain.yaml`, `privilege_sensitive_file_chain.yaml`, `memfd_exec_chain.yaml`, and `kernel_module_kprobe_chain.yaml`. | “The local detector engine can correlate ordered event sequences within short time windows to represent selected collective anomaly patterns.” | Chapter 3: rationale for local collective detection. Chapter 4: state lifecycle and grouping. Chapter 5: sequence tests, expiry behavior, memory/CPU effects, and false positives. | This is not general collective-anomaly discovery, statistical anomaly detection, or distributed correlation. One active sequence is retained per key and state is deleted after completion or expiry. Contextual anomalies and cluster-wide state are explicitly out of scope. |
| Collective detectors perform process-aware correlation. | **Partially verified** | `pkg/detectors/yaml/detector.go`: `groupKeys`, `processTreeGroupValues`, `processTreeKey`; collective YAML rules use `group_by: process_tree`. Context fields come from `pkg/bufferdecoder/protocol.go`, `EventContext`. | “For selected sequence detectors, events can be grouped by a process identity and matched across the same process or an immediate parent-child relationship.” | Chapter 4 should explain the key `host PID + start time`, parent lookup, PID-reuse protection, and grouping limitations. | “Process-aware” is acceptable only with qualification. The engine does not maintain a complete or persistent process tree, correlate arbitrary descendants, or use Kubernetes workload identity. |
| The engine deduplicates repeated alerts. | **Verified, with narrow semantics** | `pkg/detectors/engine.go`: default dedup window, alert-key construction, suppression state, `AlertsSuppressed`; tests in `pkg/detectors/engine_test.go`. | “The engine applies a short temporal deduplication window to suppress alerts with the same computed alert key.” | Chapter 4 should define the key and cleanup behavior. Chapter 5 should measure suppression effects and discuss whether configurability is needed. | This is not semantic incident aggregation. The key is derived from detector/alert fields and source-event attributes, so similar but non-identical activity can still produce multiple alerts. The window is not exposed as a CLI option. |
| Detector alerts carry MITRE ATT&CK metadata. | **Verified as declared metadata propagation; semantic correctness only partially verified** | `pkg/detectors/definition.go`: `ThreatMetadata` and format validation; `pkg/detectors/yaml/parser.go`; `pkg/detectors/yaml/detector.go`, `newAlertFromEvents`; `pkg/output/alert.go`: `threatRecord`, `formatMITRESummary`; all eight detector YAML files declare ATT&CK metadata. | “Detector definitions may declare MITRE ATT&CK tactic and technique metadata, which is validated syntactically and propagated to JSON and table alert output.” | Chapter 4: schema and propagation. Chapter 5: review of mappings and ATT&CK coverage. | Validation checks identifier shape, not existence in a specific ATT&CK release or semantic correctness of each mapping. MITRE fields do not currently drive policy selection, detector activation, or coverage reporting. Do not claim ATT&CK compliance. |
| The tool implements a minimal kernel-side UID filter. | **Partially verified; prototype in uncommitted working tree** | `pkg/ebpf/c/types.h`: `config_entry_t.kernel_uid_filter_enabled` and `kernel_uid_filter`; `pkg/ebpf/c/common/context.h`: `should_drop_by_kernel_uid_filter`, call in `init_program_data`; `pkg/ebpf/project.go`: `configureKernelFilters`; `pkg/config/config.go`: `KernelFilterConfig`; `pkg/cmd/cobra/cobra.go`: CLI flags. | “An experimental single-UID allow filter can reject non-matching events in eBPF before common context enrichment and perf-buffer submission.” | Chapter 4: map layout, initialization order, effective UID semantics, and early-return location. Chapter 5: controlled comparison with and without the filter. | It supports one UID only, not ranges, lists, PID, namespace, container, or policy semantics. Go patches raw map bytes using fixed offsets, which is layout-sensitive. It applies to hooks using `init_program_data`; coverage of every event path must be tested before claiming universal filtering. The feature is uncommitted. |
| Runtime logging is implemented with Zap and separated from security output. | **Verified** | `pkg/logging/logger.go`: `Config`, `New`, level/format parsers; `pkg/cmd/project.go`: logger construction; `pkg/ebpf/project.go`: `WithLogger`, structured diagnostics, `configureLibbpfLogging`; `pkg/output` remains the event/alert path. | “Zap is used for runtime and libbpf diagnostics, with console or JSON encoding and level control, while events and detector alerts retain dedicated output schemas.” | Chapter 3: separation of diagnostics and data plane. Chapter 4: stderr/stdout behavior and libbpf callback mapping. | Do not say Zap formats eBPF events or alerts. It deliberately does not. Structured logging does not by itself imply production observability, persistence, rotation, or remote export. |
| The application is containerized. | **Verified at image/build level** | `demo_project/Dockerfile`: multi-stage builder and slim runtime; the source is copied, `make all` is run, and `/src/dist/project` is copied to `/usr/local/bin/project`. `Makefile`: `docker-image`, `docker-run`. | “A multi-stage Docker build produces a runtime image containing the compiled Vesuvius binary and its embedded eBPF object.” | Chapter 4 should describe build dependencies, runtime libraries, embedding, and required host mounts/capabilities. Chapter 5 may validate image startup on the target node. | The runtime image is Debian-based, while the observed host target is Rocky Linux. This is valid for a containerized userspace but should be tested and explained. No image hardening, SBOM, signing, non-root mode, or production release pipeline is demonstrated. |
| The tool is ready for Kubernetes/DaemonSet deployment. | **Unsupported as a completed capability; verified only as a design direction** | `README.md`, Docker section; `documentation/implementation/docker.md`, **Kubernetes implications**, contains an illustrative DaemonSet fragment and explicitly says it is not a complete production manifest. No deployable Kubernetes manifest exists under `demo_project`. | “The container packaging is designed with a future privileged, node-level Kubernetes deployment in mind; a DaemonSet is the anticipated deployment model.” | Chapter 3: intended node-agent architecture and trust assumptions. Chapter 4: required `hostPID`, BTF, tracefs/debugfs, bpffs, cgroup namespace, and privileges. Chapter 5 or future work: real cluster deployment and security evaluation. | Do not write that Vesuvius has been integrated with Kubernetes, is DaemonSet-ready, or is production-ready. The Docker Make target uses `--privileged`, `--pid=host`, and host mounts, but equivalent Kubernetes behavior has not been demonstrated. |
| Runtime overhead is below 5% of one CPU core. | **Unsupported as a general result** | `documentation/implementation/performance.md`: results from 10 June, 14 July, and 16 July; `scripts/benchmark_userspace.sh`; `scripts/benchmark_profiles.sh`. The 14 July collective profile was often `5.9%-7.7%`; a later collective run remained around `6%-7.7%`. Several runs were interrupted before summary generation. | “A target of less than 5% of one core was adopted, but current measurements are workload-dependent and do not yet establish that the target is met. Some exploratory profiles remained below it, whereas collective-detector profiles exceeded it.” | Chapter 5 must define workload, enabled events/detectors, warm-up, duration, repetitions, userspace and eBPF accounting, lost events, baseline, CPU topology, and uncertainty. | The 10 June estimate of 3.39% and later above-target runs are not contradictory because configurations and workloads differ. They cannot support “low-overhead” without a controlled protocol. Current scripts measure userspace only; kernel-side automation remains incomplete. Benchmark scripts and recent filter changes are uncommitted. |
| Vesuvius is complete, robust, innovative, low-overhead, or production-ready. | **Unsupported as a factual Chapter 1 claim** | The dossier itself marks novelty as requiring related-work comparison and excludes production-grade completeness. Current code and performance documentation list open limitations. | “Vesuvius is a research prototype whose implemented mechanisms and trade-offs are evaluated on a constrained target environment.” | Chapter 5 may support narrower claims with measured evidence. Chapter 6 may discuss novelty only after comparison with related work and evaluation results. | Avoid all five adjectives as unqualified project claims. “Innovative” requires novelty analysis; “robust” requires broader testing; “low-overhead” requires completed benchmarks; “production-ready” conflicts with the documented scope. |

## Conflicts requiring editorial correction

1. **Current Chapter 1 is stale about policies and anomaly detection.**
   `Thesis/content/chapters/chapter1.tex`, Scope of the Work, says policy-based
   filtering, anomaly detection, and event correlation are future extensions.
   YAML policies, point detectors, short-window sequence detectors, and alert
   output are already implemented. The section should describe them as current
   prototype capabilities while retaining their limitations.

2. **“Complete pipeline” is an overclaim.**
   `chapter1.tex` uses this phrase for the kernel-to-userspace path. The data
   path exists, but “implemented pipeline” is safer because coverage,
   losslessness, operational hardening, and performance are not complete.

3. **The current Chapter 1 calls eBPF “new” and states low overhead as a general
   property.** These are external claims requiring authoritative citations and
   qualification. They are not established by the project repository.

4. **Transport comments disagree.** The early `Project` struct comment in
   `pkg/ebpf/project.go` describes ring-buffer event hooks, while `Init` and the
   current documentation identify perf buffer as the operational path for
   attached event hooks. Chapter 1 should use the latter wording.

5. **Policy capability is sometimes described too broadly.** The code has
   monitor, detect, and suppress modes, but the runtime data path currently uses
   policy matching primarily as event admission. It does not yet use matched
   policy identity to select detectors or populate alert provenance.

6. **Process-tree language is broader than the implementation.** The collective
   detector key can join the same process and an immediate parent-child pair. It
   is not a persistent process graph and should not be presented as full process
   tree tracking.

7. **MITRE mappings are declarations, not validated detection coverage.** The
   implementation validates the syntax of tactic and technique IDs and carries
   them to output. It does not verify mappings against the ATT&CK catalog or use
   them to drive runtime behavior.

8. **Kubernetes is an intended deployment, not a tested implementation.** The
   repository contains guidance and an illustrative manifest fragment, but no
   production manifest or recorded cluster validation.

9. **The performance record is configuration-dependent.** An early combined
   estimate was below 5%, while later detector-heavy userspace samples were
   above 5%. Chapter 1 should present the threshold as an evaluation objective,
   not an achieved property.

10. **Recent evidence is not yet in the Git baseline.** The UID filter, enhanced
    registry tests, dispatch metrics, and benchmark profiles are visible in the
    audited working tree but uncommitted. They should be committed and rerun
    before being treated as release-level thesis evidence.

## Recommended Chapter 1 evidence boundary

Chapter 1 can safely claim that Vesuvius is a Rocky Linux-oriented research
prototype with an implemented kernel-to-userspace monitoring pipeline, a custom
event contract, selected process/security coverage, userspace policy selection,
YAML point and short-window collective detectors, qualified process-aware
correlation, temporal alert deduplication, MITRE metadata propagation,
structured diagnostic logging, and Docker packaging.

Chapter 1 should present the kernel UID filter as an experimental optimization,
Kubernetes as a future deployment direction, and the 5% CPU figure as an open
evaluation target. Detailed hook inventories, binary layouts, correlation state,
filter map offsets, detector rules, container privileges, and performance
numbers belong in Chapters 3-5 or the appendix.
