# Related Runtime Security Tools: Source-Backed Analysis

## Scope and Version Note

This document supports Sections 2.4 and 2.5 of the thesis. It compares Tracee,
Falco, and Tetragon because they represent three materially different design
points:

- Tracee combines broad runtime event collection, policy-based event selection,
  enrichment, and detector execution.
- Falco evaluates a rule language over streams of kernel and plugin events and
  emits alerts when a rule condition matches.
- Tetragon places selectors and optional actions close to the monitored kernel
  hook through programmable tracing policies.

The comparison does not imply feature equivalence with the proposed system.
Tracee's local source tree was inspected at commit
`df687a838e542d3c1e99c6f93d71e28e9e0d7d27`. Tracee's detector API is under
active development after the v0.24 release, so statements about detectors below
refer to the inspected development version, while stable v0.24 documentation is
used where appropriate for established output and installation behaviour.
Falco and Tetragon statements refer to their official documentation as accessed
on 27 July 2026.

## 2.4 Related Runtime Security Tools

Runtime security tools built around eBPF share a broad objective: they observe
kernel-mediated activity and convert low-level execution data into information
that can be inspected or acted upon. Their internal abstractions differ,
however. Some systems primarily expose a rich event stream, some centre their
design on alert rules, and others couple event collection with in-kernel policy
actions. Tracee, Falco, and Tetragon illustrate these alternatives and therefore
provide a useful basis for positioning the architecture developed in this
thesis. A recent peer-reviewed comparison also identifies these projects among
the principal open-source eBPF security tools, while noting meaningful
differences in their goals and architectures \cite{her2025ebpfsecuritytools}.

### 2.4.1 Tracee

Tracee is a Linux runtime security and observability tool that collects system
calls, file operations, networking activity, Linux Security Module events, and
other kernel-derived events. Its documentation deliberately presents these
observations through a common event abstraction rather than exposing each
instrumentation mechanism as an independent user-facing interface
\cite{aquasecurity_tracee,aquasecurity_tracee_events}. This event-oriented
design makes the collection layer useful both for direct tracing and for
higher-level security detection.

Tracee's policies define the event domain to be traced. A policy combines a
scope, which identifies the workloads or execution contexts of interest, with
rules that select events and optionally constrain their data. The current
implementation supports up to 64 simultaneous policies. In the eBPF event
context, a 64-bit `matched_policies` bitmap records which policies matched an
event. Kernel-side filtering reduces unnecessary submissions where possible,
while filters that cannot be evaluated in the kernel are completed in user
space \cite{aquasecurity_tracee_policies}. The inspected source confirms this
division in `pkg/ebpf/c/common/filtering.h`, `pkg/ebpf/c/types.h`, and
`pkg/policy/policy.go`.

Detection is a separate concern. Tracee's current detector API lets a detector
declare its required input events, process those events, and emit one or more
derived event types. Detectors can be implemented in Go or described in YAML,
and the engine dispatches an input event only to detectors registered for that
event. Detector outputs are registered in the same event system as kernel
events, allowing policies to select them and other detectors to consume them
\cite{aquasecurity_tracee_detectors,aquasecurity_tracee_detector_api}. This
supports detector chaining and preserves provenance between a derived detection
and its parent events. A Go detector may also maintain internal state and
consume several event types, although this flexibility should not be described
as a universal temporal-correlation language.

Tracee primarily provides observation, detection, and forensic collection. Its
documented outputs include tables, JSON, templates, files, forwarding
destinations, and webhooks, with optional enrichment of process and workload
metadata \cite{aquasecurity_tracee_output}. The reviewed documentation does not
define a general inline enforcement model comparable to Tetragon's tracing
policy actions. Tracee's own security model also states an important trust
assumption: the kernel is expected to be trustworthy when Tracee starts, while
monitoring kernel-level compromise is necessarily best effort
\cite{aquasecurity_tracee_security_model}.

Tracee is relevant to this thesis principally as an architectural reference.
The separation among event collection, policy selection, detector execution,
and output informed the proposed system. The thesis implementation remains an
independent and narrower prototype, particularly in its focus on process and
security activity, its compatibility target, and its bounded local detection
model.

### 2.4.2 Falco

Falco is a runtime security engine whose central abstraction is a rule evaluated
over an event stream. Its default event source is Linux system-call activity
captured by a kernel driver, while plugins can introduce additional event
sources. The current default Linux driver is the modern eBPF probe, although a
kernel-module driver remains available. The `libscap` layer provides a common
capture interface and negotiates driver API and event-schema versions
\cite{falcosecurity_falco,falcosecurity_falco_kernel_events,
falcosecurity_falco_kernel_architecture}.

Falco rules are written in YAML and combine a condition, output text, priority,
and optional exceptions, tags, and source information. Lists and macros provide
reuse. For each incoming event, Falco evaluates the relevant condition and
emits an alert when the expression is true
\cite{falcosecurity_falco_rules,falcosecurity_falco_rule_conditions}. This
model is effective for declarative single-event detection over a rich set of
event and process fields. Falco maintains process and container context for
enrichment, but its core rule syntax is not a general temporal sequence
language. Moreover, official documentation states that event sources run in
separate threads and that rules cannot correlate events across different event
sources \cite{falcosecurity_falco_event_sources}.

Filtering occurs through the selected event set, rule conditions, and
exceptions. Falco can enrich events with container and orchestration metadata
and can send alerts to standard output, files, syslog, spawned programs, and
HTTP endpoints. Broader integrations are commonly supplied by external output
consumers \cite{falcosecurity_falco_outputs}. Falco's core role is therefore
observation and alert generation; automated response is normally delegated to
another component rather than executed as a general inline kernel policy.

Falco contributes a different prior-art lesson from Tracee. It demonstrates the
value of a compact declarative rule language, reusable macros and lists,
explicit exception handling, and a stable alert-oriented interface. It also
shows the boundary of a per-event rule engine when the desired anomaly is
defined by a short ordered sequence rather than one event.

### 2.4.3 Tetragon

Tetragon is an eBPF-based security observability and runtime enforcement system.
It records process lifecycle activity by default and can collect additional
events through tracing policies attached to kernel functions, tracepoints,
user-space probes, LSM hooks, and other supported instrumentation points
\cite{cilium_tetragon,cilium_tetragon_tracing_policy}. Its event representation
contains process identity and ancestry, arguments, credentials, capabilities,
timestamps, and node information. Events can be consumed through gRPC, JSON
export, or the `tetra` command-line client
\cite{cilium_tetragon_events,cilium_tetragon_process_lifecycle}.

A Tetragon `TracingPolicy` specifies a hook, the arguments and return values to
capture, and selectors that filter matching activity in the kernel. Selectors
can match process identifiers, binaries, namespaces, capabilities, workload
metadata, arguments, return values, and other event data. They can also request
actions such as signal delivery or return-value override
\cite{cilium_tetragon_selectors,cilium_tetragon_enforcement}. Tetragon therefore
moves a larger part of filtering and response into the kernel than the
user-space detector architecture considered in this thesis.

Tetragon enriches individual observations with stateful process identity and
lifecycle context. However, the reviewed tracing-policy documentation defines
matching around a selected hook and its event data; it does not document a
general-purpose temporal language for correlating an arbitrary ordered sequence
of heterogeneous events. Its principal distinction is instead enforcement:
policies may run in monitoring, enforcement, or monitor-only modes, and eligible
hooks can reject an operation or terminate a process
\cite{cilium_tetragon_policy_modes,cilium_tetragon_enforcement}.

This stronger kernel integration introduces explicit platform assumptions.
Official installation guidance requires Linux 4.19 or later and BTF support,
with individual capabilities depending on kernel features and configuration
\cite{cilium_tetragon_requirements}. Its threat model also recognises that an
attacker with root-equivalent control of the host may disable the agent,
manipulate eBPF programs, or interfere with the event stream
\cite{cilium_tetragon_threat_model}.

### Comparative Synthesis

The three projects share the use of kernel-derived runtime evidence but optimise
for different control points. Tracee presents a broad event pipeline in which
policies select observations and detectors derive additional events. Falco
centres the design on declarative conditions that transform individual incoming
events into alerts. Tetragon centres the design on programmable hook-specific
policies with in-kernel selectors and optional actions. These are complementary
architectural precedents rather than interchangeable feature sets.

The proposed system adopts an event pipeline and a separation between policy
selection and detector logic, but it does not attempt to reproduce Tracee's
event catalogue, Falco's rule ecosystem, or Tetragon's enforcement surface. Its
distinctive engineering focus is a compact process- and security-oriented
prototype for a constrained Rocky Linux 8 kernel, with Go-based user-space
normalisation, automatically activated YAML detectors, short process-aware
collective detection, MITRE ATT&CK-labelled alerts, and explicit performance
measurement. These differences establish the system's position without
requiring a claim that each mechanism is unprecedented.

## Comparison Table

| Criterion | Tracee | Falco | Tetragon |
|---|---|---|---|
| Purpose and threat model | Runtime security, observability, and forensic event collection. It assumes a trustworthy kernel at startup and treats kernel compromise detection as best effort \cite{aquasecurity_tracee_security_model}. | Runtime detection over host, container, and plugin event streams. Its core operation is matching rules and emitting alerts \cite{falcosecurity_falco}. | Security observability plus optional inline enforcement. Its threat model covers workload attackers and recognises that root-equivalent host attackers can interfere with the agent and event stream \cite{cilium_tetragon_threat_model}. |
| Kernel event sources and eBPF architecture | System calls, kprobes, tracepoints, LSM-related hooks, networking, and other sources are normalised as events. CO-RE and BTF are used where available; supported installation includes RHEL 8's 4.18 kernel \cite{aquasecurity_tracee_events,aquasecurity_tracee_requirements}. | System-call events are captured through a common `libscap` interface using the modern eBPF driver by default or a kernel-module driver; plugins add non-kernel sources \cite{falcosecurity_falco_kernel_events,falcosecurity_falco_kernel_architecture}. | eBPF programs attach through tracing policies to kprobes, tracepoints, uprobes, LSM hooks, and related sources; process exec and exit visibility is built in \cite{cilium_tetragon_tracing_policy,cilium_tetragon_process_lifecycle}. |
| Event model | A common event schema represents kernel observations and detector-derived events. Detector output can re-enter the event pipeline \cite{aquasecurity_tracee_events,aquasecurity_tracee_detector_api}. | Events belong to a named source and expose source-specific fields used by rule conditions. Kernel events follow a versioned driver schema \cite{falcosecurity_falco_event_sources,falcosecurity_falco_kernel_architecture}. | Structured events include process identity, ancestry, arguments, credentials, capabilities, time, and node metadata; hook-specific events add captured fields \cite{cilium_tetragon_events}. |
| Policy, rule, signature, or detector model | Policies select scope and events. Modern Go or YAML detectors consume selected events and emit derived events; legacy signatures are being superseded by detectors \cite{aquasecurity_tracee_policies,aquasecurity_tracee_detectors}. | YAML rules define conditions, output, priority, source, tags, and exceptions; macros and lists provide reuse \cite{falcosecurity_falco_rules}. | Tracing policies define hooks, captured arguments, selectors, and optional actions. Policies may be loaded dynamically \cite{cilium_tetragon_tracing_policy,cilium_tetragon_selectors}. |
| Single-event and multi-event detection | YAML and Go detectors support event-driven detection. Go detectors may consume several event types, retain state, and consume prior detector outputs; detector chaining has explicit provenance support \cite{aquasecurity_tracee_detector_api}. | Core conditions are evaluated against one incoming event. Process context can enrich that event, but cross-source correlation is explicitly unsupported and no general temporal sequence syntax is documented \cite{falcosecurity_falco_rule_conditions,falcosecurity_falco_event_sources}. | Selectors match data associated with a policy hook and process context. Process lifecycle state is preserved, but the reviewed documentation does not define a general arbitrary-event temporal sequence language \cite{cilium_tetragon_selectors,cilium_tetragon_process_lifecycle}. |
| Filtering and enrichment | Kernel and user-space filters apply policy scope and event data constraints; enrichment adds process and workload context \cite{aquasecurity_tracee_policies,aquasecurity_tracee_output}. | Event selection, conditions, and exceptions suppress irrelevant matches; event fields can include process, container, and orchestration context \cite{falcosecurity_falco_rule_conditions,falcosecurity_falco_exceptions}. | Selectors filter in kernel by process, namespace, capability, workload, argument, and return-value data; export filters can further reduce user-space output \cite{cilium_tetragon_selectors,cilium_tetragon_events}. |
| Enforcement versus observation | Observation, detection, and forensic collection are the documented core. The reviewed documentation does not expose a general inline enforcement model equivalent to Tetragon actions. | The core engine observes events and emits alerts. Response is delegated to configured outputs and external consumers rather than expressed as general inline kernel enforcement \cite{falcosecurity_falco_outputs}. | Supports observation and inline actions, including signal delivery and return-value override where the selected hook supports them \cite{cilium_tetragon_enforcement,cilium_tetragon_policy_modes}. |
| Output and integration model | Table, JSON, templates, files, forwarding, and webhooks, with optional enrichment and separate logging streams \cite{aquasecurity_tracee_output}. | Standard output, file, syslog, spawned program, and HTTP(S), with additional integrations supplied through output consumers \cite{falcosecurity_falco_outputs}. | gRPC event stream, JSON export, and `tetra` CLI, with allow-list and deny-list field filters \cite{cilium_tetragon_events}. |
| Portability and kernel assumptions | Linux only; current guidance supports kernel 5.4+ and RHEL 8's 4.18 kernel, with BTF and `/proc/kallsyms` requirements and feature-dependent LSM support \cite{aquasecurity_tracee_requirements}. | Linux kernel support depends on the chosen driver. The kernel module supports older kernels, while modern eBPF capabilities depend on available kernel features \cite{falcosecurity_falco_kernel_events}. | Requires Linux 4.19+ and BTF; individual features depend on kernel configuration and architecture \cite{cilium_tetragon_requirements}. |

## Established Prior Art

The following design ideas must be presented as established prior art rather
than as original contributions:

1. Collecting Linux runtime activity through eBPF programs attached to system
   calls, tracepoints, kprobes, LSM-related hooks, and process lifecycle hooks.
2. Normalising heterogeneous kernel observations into a common user-space event
   representation.
3. Selecting event classes and workload scopes through declarative policies.
4. Applying part of an event filter in the kernel to reduce user-space traffic.
5. Expressing security detections in YAML or another declarative rule format.
6. Separating low-level collection from higher-level detection logic.
7. Enriching events with process identity, ancestry, credentials, capabilities,
   container, or workload metadata.
8. Producing structured alerts and forwarding them through files, streams,
   webhooks, or external integration components.
9. Associating detections with MITRE ATT&CK tactics and techniques.
10. Maintaining detector state, consuming multiple event types, or chaining
    derived detections in a user-space engine.
11. Using process-aware correlation keys and bounded time windows for local
    sequence detection.
12. Supporting observation-only and enforcement-oriented operating modes in
    eBPF security tooling.

## Differences That Can Support Section 2.5

The following statements are supported by the implementation and are suitably
bounded for the positioning section:

1. The proposed system targets process and security monitoring on Rocky Linux 8
   with kernel `4.18.0-553.109.1.el8_10.x86_64`, rather than pursuing a broad
   cross-platform event catalogue.
2. Its eBPF-side design is adapted to an enterprise kernel whose functionality
   includes backported features; compatibility is therefore evaluated against
   the target kernel rather than inferred only from the upstream version
   number.
3. The kernel-to-user-space event protocol, normalisation layer, policy manager,
   detector engine, and output path are implemented as a compact independent
   prototype in C and Go.
4. Policies and detectors have intentionally distinct responsibilities:
   policies select the event domain, while detectors evaluate normalised events
   and generate alerts.
5. Detectors are automatically activated from the configured policy and their
   declared event requirements, reducing manual coordination between event
   selection and detection configuration.
6. The detector engine supports both single-event anomalies and bounded,
   process-aware collective sequences while deliberately excluding
   cluster-wide contextual anomaly detection from the local agent.
7. Collective alerts preserve the ordered source-event sequence, making the
   evidence behind a match visible in table and structured output.
8. MITRE ATT&CK metadata is validated and propagated into alerts as a stable
   classification and integration field, connecting local detections to a
   recognised adversary-behaviour vocabulary.
9. Optional kernel-side UID filtering demonstrates an explicit mechanism for
   reducing event volume before transport, while remaining disabled by default
   to preserve general observability.
10. Performance is treated as a measurable design constraint, with repeatable
    benchmark profiles and a stated target of remaining below five per cent of
    one CPU core under the defined workload. This is an evaluation objective,
    not a completed general performance claim.

These points should be described as differences in scope, integration, and
engineering emphasis. They do not establish superiority over the compared
systems and should not be labelled as novel without a broader literature
review and experimental validation.

## Claims to Avoid

- Do not state that the proposed system has feature parity with Tracee, Falco,
  or Tetragon.
- Do not call Tracee policies detectors: they select scope and events, while
  detectors consume events and derive detections.
- Do not describe Falco's process enrichment as a general multi-event sequence
  engine.
- Do not claim that Tetragon lacks state: it maintains process lifecycle
  context, even though its documented tracing-policy language is hook-centred.
- Do not claim that any of the tools prevents every observed attack.
- Do not infer kernel support solely from an upstream version number when
  enterprise distributions backport functionality.
- Do not present Kubernetes deployment as a central differentiator.

## Verified BibTeX Proposals

The existing `Thesis/bibliography.bib` already contains the general Tracee
entry, the Tracee policy entry, and the peer-reviewed comparison by Her et al.
Those entries should be retained. The following non-duplicate entries provide
more precise support for the claims above.

```bibtex
@online{aquasecurity_tracee_events,
  author  = {{Aqua Security}},
  title   = {Tracee Events},
  year    = {2026},
  url     = {https://aquasecurity.github.io/tracee/dev/docs/events/},
  urldate = {2026-07-27},
  note    = {Official Tracee documentation}
}

@online{aquasecurity_tracee_detectors,
  author  = {{Aqua Security}},
  title   = {Tracee Detectors},
  year    = {2026},
  url     = {https://aquasecurity.github.io/tracee/dev/docs/detectors/},
  urldate = {2026-07-27},
  note    = {Official Tracee development documentation}
}

@online{aquasecurity_tracee_detector_api,
  author  = {{Aqua Security}},
  title   = {Tracee Detector API Reference},
  year    = {2026},
  url     = {https://aquasecurity.github.io/tracee/dev/docs/detectors/api-reference/},
  urldate = {2026-07-27},
  note    = {Official Tracee development documentation}
}

@online{aquasecurity_tracee_output,
  author  = {{Aqua Security}},
  title   = {Tracee Output Configuration},
  year    = {2026},
  url     = {https://aquasecurity.github.io/tracee/dev/docs/outputs/},
  urldate = {2026-07-27},
  note    = {Official Tracee documentation}
}

@online{aquasecurity_tracee_requirements,
  author  = {{Aqua Security}},
  title   = {Installing Tracee},
  year    = {2026},
  url     = {https://aquasecurity.github.io/tracee/dev/docs/install/},
  urldate = {2026-07-27},
  note    = {Official Tracee documentation}
}

@online{aquasecurity_tracee_security_model,
  author  = {{Aqua Security}},
  title   = {Tracee Security Model},
  year    = {2026},
  url     = {https://aquasecurity.github.io/tracee/dev/docs/security-model/},
  urldate = {2026-07-27},
  note    = {Official Tracee documentation}
}

@online{falcosecurity_falco,
  author  = {{The Falco Project Authors}},
  title   = {Falco Documentation},
  year    = {2026},
  url     = {https://falco.org/docs/},
  urldate = {2026-07-27},
  note    = {Official Falco documentation}
}

@online{falcosecurity_falco_event_sources,
  author  = {{The Falco Project Authors}},
  title   = {Falco Event Sources},
  year    = {2026},
  url     = {https://falco.org/docs/concepts/event-sources/},
  urldate = {2026-07-27},
  note    = {Official Falco documentation}
}

@online{falcosecurity_falco_kernel_events,
  author  = {{The Falco Project Authors}},
  title   = {Falco Kernel Events},
  year    = {2026},
  url     = {https://falco.org/docs/concepts/event-sources/kernel/},
  urldate = {2026-07-27},
  note    = {Official Falco documentation}
}

@online{falcosecurity_falco_kernel_architecture,
  author  = {{The Falco Project Authors}},
  title   = {Falco Kernel Event Architecture},
  year    = {2026},
  url     = {https://falco.org/docs/concepts/event-sources/kernel/architecture/},
  urldate = {2026-07-27},
  note    = {Official Falco documentation}
}

@online{falcosecurity_falco_rules,
  author  = {{The Falco Project Authors}},
  title   = {Falco Rules},
  year    = {2026},
  url     = {https://falco.org/docs/concepts/rules/},
  urldate = {2026-07-27},
  note    = {Official Falco documentation}
}

@online{falcosecurity_falco_rule_conditions,
  author  = {{The Falco Project Authors}},
  title   = {Falco Rule Conditions},
  year    = {2026},
  url     = {https://falco.org/docs/concepts/rules/conditions/},
  urldate = {2026-07-27},
  note    = {Official Falco documentation}
}

@online{falcosecurity_falco_exceptions,
  author  = {{The Falco Project Authors}},
  title   = {Falco Rule Exceptions},
  year    = {2026},
  url     = {https://falco.org/docs/concepts/rules/exceptions/},
  urldate = {2026-07-27},
  note    = {Official Falco documentation}
}

@online{falcosecurity_falco_outputs,
  author  = {{The Falco Project Authors}},
  title   = {Falco Outputs},
  year    = {2026},
  url     = {https://falco.org/docs/concepts/outputs/},
  urldate = {2026-07-27},
  note    = {Official Falco documentation}
}

@online{cilium_tetragon,
  author  = {{The Tetragon Authors}},
  title   = {Tetragon Source Repository},
  year    = {2026},
  url     = {https://github.com/cilium/tetragon},
  urldate = {2026-07-27},
  note    = {Official source repository}
}

@online{cilium_tetragon_tracing_policy,
  author  = {{The Tetragon Authors}},
  title   = {Tetragon Tracing Policies},
  year    = {2026},
  url     = {https://tetragon.io/docs/concepts/tracing-policy/},
  urldate = {2026-07-27},
  note    = {Official Tetragon documentation}
}

@online{cilium_tetragon_selectors,
  author  = {{The Tetragon Authors}},
  title   = {Tetragon Tracing Policy Selectors},
  year    = {2026},
  url     = {https://tetragon.io/docs/concepts/tracing-policy/selectors/},
  urldate = {2026-07-27},
  note    = {Official Tetragon documentation}
}

@online{cilium_tetragon_events,
  author  = {{The Tetragon Authors}},
  title   = {Tetragon Events},
  year    = {2026},
  url     = {https://tetragon.io/docs/concepts/events/},
  urldate = {2026-07-27},
  note    = {Official Tetragon documentation}
}

@online{cilium_tetragon_process_lifecycle,
  author  = {{The Tetragon Authors}},
  title   = {Process Lifecycle Monitoring with Tetragon},
  year    = {2026},
  url     = {https://tetragon.io/docs/use-cases/process-lifecycle/},
  urldate = {2026-07-27},
  note    = {Official Tetragon documentation}
}

@online{cilium_tetragon_enforcement,
  author  = {{The Tetragon Authors}},
  title   = {Tetragon Enforcement},
  year    = {2026},
  url     = {https://tetragon.io/docs/concepts/enforcement/},
  urldate = {2026-07-27},
  note    = {Official Tetragon documentation}
}

@online{cilium_tetragon_policy_modes,
  author  = {{The Tetragon Authors}},
  title   = {Tetragon Tracing Policy Modes},
  year    = {2026},
  url     = {https://tetragon.io/docs/concepts/tracing-policy/mode/},
  urldate = {2026-07-27},
  note    = {Official Tetragon documentation}
}

@online{cilium_tetragon_requirements,
  author  = {{The Tetragon Authors}},
  title   = {Tetragon Installation FAQ},
  year    = {2026},
  url     = {https://tetragon.io/docs/installation/faq/},
  urldate = {2026-07-27},
  note    = {Official Tetragon documentation}
}

@online{cilium_tetragon_threat_model,
  author  = {{The Tetragon Authors}},
  title   = {Tetragon Threat Model},
  year    = {2026},
  url     = {https://tetragon.io/docs/threat-model/},
  urldate = {2026-07-27},
  note    = {Official Tetragon documentation}
}
```

## Tracee Source Permalinks

The following source links pin the inspected Tracee behaviour to a concrete
revision rather than to a moving development branch:

- [`EventDetector` API](https://github.com/aquasecurity/tracee/blob/df687a838e542d3c1e99c6f93d71e28e9e0d7d27/api/v1beta1/detection/detector.go)
- [Detector event registration](https://github.com/aquasecurity/tracee/blob/df687a838e542d3c1e99c6f93d71e28e9e0d7d27/pkg/detectors/events.go)
- [Detector dispatch engine](https://github.com/aquasecurity/tracee/blob/df687a838e542d3c1e99c6f93d71e28e9e0d7d27/pkg/detectors/engine.go)
- [Policy representation](https://github.com/aquasecurity/tracee/blob/df687a838e542d3c1e99c6f93d71e28e9e0d7d27/pkg/policy/policy.go)
- [Kernel-side filtering](https://github.com/aquasecurity/tracee/blob/df687a838e542d3c1e99c6f93d71e28e9e0d7d27/pkg/ebpf/c/common/filtering.h)
- [`matched_policies` event field](https://github.com/aquasecurity/tracee/blob/df687a838e542d3c1e99c6f93d71e28e9e0d7d27/pkg/ebpf/c/types.h)

