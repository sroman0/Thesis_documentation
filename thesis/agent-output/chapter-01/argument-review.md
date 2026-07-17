# Academic argument review - Chapter 1

## Overall assessment

The proposed Chapter 1 has a coherent macro-structure:

```text
context -> problem -> objectives -> research questions -> contributions
        -> scope -> methodology -> thesis structure
```

This sequence is appropriate for an MSc thesis and can be retained. The
current LaTeX draft, however, does not yet follow it: it starts from eBPF as a
technology, omits an explicit problem statement and methodology, and still
describes policies, anomaly detection and correlation as future extensions.

The main revision should therefore be argumentative rather than incremental.
Chapter 1 should begin with the runtime security problem, derive the need for
kernel-level visibility, and introduce eBPF only as the selected enabling
technology. It should not read as a short tutorial on eBPF.

## Blocking issues before drafting

### 1. The meaning of anomaly detection is currently ambiguous

The current detectors are declarative and rule-based. They match individual
events or short event sequences configured in YAML. Calling this system simply
"anomaly detection" may imply statistical, learned or baseline-driven anomaly
detection, which is not currently implemented.

The chapter should either:

- use the more precise expression `point and collective security patterns`; or
- explicitly define `point anomaly` and `collective anomaly` as project-local
  terms for rule-defined suspicious events and sequences.

The broader taxonomy belongs to Chapter 2. Chapter 1 only needs a short and
careful definition.

### 2. MITRE ATT&CK classification must not be confused with detection coverage

The implementation propagates MITRE ATT&CK tactics and techniques declared in
detector YAML files into alert output. This is an implemented contribution.

It does not yet demonstrate that the system detects every behavior associated
with those techniques, nor that it provides comprehensive ATT&CK coverage.
Chapter 1 should describe MITRE as alert classification and traceability. Any
coverage claim must be evaluated in Chapter 5.

### 3. The current scope statement is obsolete

The final paragraph of `Scope of the Work` presents policy-based filtering,
anomaly detection and event correlation as future work. All three now have an
MVP implementation. The sentence must be replaced with a distinction between:

- implemented local policy and detector capabilities;
- advanced distributed or contextual analysis left to a central platform;
- enforcement, which is outside the current scope.

### 4. Networking authorship and scope are unresolved

Networking hooks are present in the project, but the dossier states that this
area was developed in collaboration. The thesis must not imply sole authorship
without an agreed attribution.

Before drafting the final scope, the author, company and supervisor should
decide whether networking is:

- part of the evaluated thesis contribution;
- integrated background work credited to a collaborator; or
- mentioned only as an adjacent project capability.

Until then, Chapter 1 should retain its primary focus on process and security
monitoring.

### 5. Kubernetes is a deployment direction, not yet a demonstrated result

The architecture is intended for node-level deployment and the container model
is compatible with a privileged DaemonSet. Unless an actual Kubernetes
deployment is implemented and evaluated, Chapter 1 should say `deployment
target`, `deployment perspective` or `intended integration`, not claim a
validated Kubernetes integration.

## Missing premises

The following premises are needed to make the argument complete:

1. Runtime behavior can differ from static configuration and therefore needs
   observation during execution.
2. Process execution, credentials, memory, filesystem, namespaces and kernel
   activity form a relevant local security surface.
3. User-space-only visibility may miss or weaken the semantic context available
   at kernel hooks. This requires an external citation and cautious wording.
4. Kernel instrumentation introduces a tension between visibility,
   compatibility, safety and runtime overhead.
5. The target is deliberately constrained to Rocky Linux 8.10 and its
   RHEL-compatible kernel instead of claiming universal portability.
6. A node-level agent has detailed local visibility but lacks the cluster-wide
   context required for contextual anomalies.
7. Monitoring, detection and enforcement are different responsibilities;
   Vesuvius currently implements the first two in a limited local form, not
   enforcement.
8. The evaluation criteria must follow directly from the research questions:
   compatibility and functionality, detection case studies, and overhead.

## Review of the research questions

The three-question structure is sound and maps naturally to Chapters 3-5. The
wording needs minor refinement to make the questions more measurable and less
promotional.

### RQ1 - Recommended formulation

Current weakness: `meaningful process and security visibility` is subjective
and difficult to evaluate.

Recommended formulation:

> Which architectural and implementation adaptations are required to build an
> eBPF-based process and security monitoring pipeline on the target Rocky Linux
> 8.10 kernel?

This version can be answered through design choices, compatibility constraints,
verifier-driven changes and functional validation. It does not require proving
that the visibility is universally meaningful.

### RQ2 - Recommended formulation

Current weakness: `identify anomalies` can be interpreted as statistical
anomaly detection, while the implementation is rule-based.

Recommended formulation:

> How can heterogeneous kernel events be normalized and evaluated through
> declarative policies and detectors to identify security-relevant point events
> and short collective event patterns?

If the thesis retains the word `anomaly`, its project-specific meaning must be
defined before this question.

### RQ3 - Recommended formulation

The current question is already suitable, but `representative workloads` must
be operationally defined in Chapter 5.

Recommended formulation:

> What CPU and memory overhead does the proposed pipeline introduce under the
> selected benchmark profiles, and how does early kernel-side UID filtering
> affect that overhead?

This wording reflects the measurements currently available. If the final
evaluation includes latency, event loss or throughput, those metrics can be
added later.

## Contributions: recommended consolidation

The dossier currently lists eight implementation contributions. Presenting all
eight at the same level would make the introduction resemble a feature list.
They should be consolidated into four principal contributions.

### Contribution 1 - Target-aware monitoring architecture

A modular kernel-to-userspace monitoring pipeline adapted to the constraints of
Rocky Linux 8.10, covering selected process and security event families.

This includes implementation contributions such as hook selection, loader and
probe lifecycle, but their details belong to Chapters 3 and 4.

### Contribution 2 - Coherent event contract and userspace processing

A controlled binary event contract connecting eBPF programs, transport,
decoder, event registry and structured output, with consistency checks across
the kernel and Go components.

### Contribution 3 - Declarative local detection pipeline

A userspace policy and detector architecture supporting rule-defined point
events and short collective patterns, process-aware local grouping, alert
deduplication and MITRE ATT&CK metadata propagation.

The exact limits of `process-aware` must be stated: the system does not maintain
a complete persistent process graph.

### Contribution 4 - Operational and performance-oriented controls

Separation of diagnostic logging, events and alerts; container-oriented
packaging; reproducible benchmark profiles; and an initial kernel-side UID
filter intended to reduce kernel-to-userspace traffic.

This contribution must not claim that the final CPU target has already been
met.

## Engineering contributions versus candidate novelty

### Clearly supported engineering contributions

- implementation of the target-aware eBPF and Go pipeline;
- selected process and security event coverage;
- custom event contract, decoder and registry consistency checks;
- YAML policy and detector runtime;
- point and short-sequence rule evaluation;
- MITRE metadata in alerts;
- deduplication, logging separation and optional UID pre-filtering;
- container packaging for the intended deployment model.

### Candidate research novelty requiring comparison

- the exact combination of target-aware eBPF monitoring, declarative point and
  collective detectors and local process-aware grouping;
- the architectural boundary between local point/collective detection and
  centralized contextual analysis;
- the trade-off between a generic userspace detector engine and intentionally
  minimal kernel-side filtering on an older enterprise kernel.

### Claims that should not be presented as novelty by themselves

- using eBPF for runtime security;
- using YAML for policies or detections;
- attaching MITRE ATT&CK metadata to alerts;
- using Tracee-inspired architectural patterns;
- packaging an eBPF agent in Docker;
- supporting perf buffers or CO-RE.

These may contribute to the system design, but their originality cannot be
assumed.

## Unsupported or promotional claims in the current draft

The following expressions should be removed or rewritten unless supported by
specific evidence:

- `eBPF is a new programmability mechanism`;
- `sophisticated verifier framework`;
- `security and performance constraints` as an exhaustive description of what
  the verifier guarantees;
- `achieving high performance and low overhead`;
- `maintaining or even enhancing the security of the kernel`;
- `reliable and structured way`;
- `complete pipeline`;
- `the impact of this new technology is significant`;
- `next-generation security tools`.

The phrase `allows developers to run custom code in kernel space` is also too
imprecise. eBPF executes restricted programs under a kernel-controlled model;
it should not be equated with unrestricted kernel code execution.

The current text also repeats `leverage` and makes organization-level claims
without evidence. The motivation should stay technical and problem-driven.

## Content placement across Chapters 2-5

### Defer to Chapter 2

- history from BPF to eBPF;
- verifier internals and guarantees;
- JIT compilation;
- program types, maps and helpers;
- CO-RE and BTF;
- formal anomaly taxonomy;
- detailed explanation of MITRE ATT&CK;
- detailed description and comparison of Tracee and other tools.

### Defer to Chapter 3

- complete functional and non-functional requirements;
- exact target assumptions;
- detailed architecture diagrams;
- rationale for kernel/userspace responsibilities;
- policy/detector architecture;
- deployment and trust model;
- design alternatives and decisions.

### Defer to Chapter 4

- individual hooks and syscall handling;
- libbpfgo lifecycle and probe registry;
- event structure sizes and binary layout;
- perf buffer and ring buffer implementation details;
- YAML schema and detector operators;
- process grouping implementation;
- UID map offsets and filtering code;
- Dockerfile, Zap and package-level implementation details.

### Defer to Chapter 5

- number and coverage of implemented events;
- detector demonstrations and false-positive discussion;
- ATT&CK coverage matrix;
- benchmark values and the 5% target;
- effects of warm-up, workloads and kernel filtering;
- comparison results, limitations and threats to validity.

## Recommended paragraph-level narrative

### 1.1 Context and Motivation

1. Introduce runtime security as observation of behavior while workloads are
   executing.
2. Explain why process and security activity needs visibility close to the
   kernel boundary.
3. Introduce eBPF as a controlled instrumentation mechanism selected for this
   purpose, without explaining its internals.
4. Move from the general setting to the industrial target: Rocky Linux,
   containerized deployment and node-level monitoring.
5. End with the central tension: useful visibility must coexist with kernel
   compatibility, safety and bounded overhead.

### 1.2 Problem Statement

1. State the concrete input-output problem: heterogeneous kernel activity must
   become normalized events and actionable local alerts.
2. Explain why collection alone is insufficient: event volume and isolated
   records create noise.
3. Introduce policy selection and short event correlation as the proposed local
   response.
4. State technical constraints: enterprise kernel, verifier, transport cost and
   node-local context.
5. Delimit the unresolved cluster-level problem without attempting to solve it.

### 1.3 Objectives and Research Questions

1. State one main objective in a single paragraph.
2. Present four or five specific objectives aligned with the pipeline:
   collection, normalization, configurable detection, classification and
   evaluation.
3. Present the three research questions in the same order as the later
   evaluation.
4. Add one sentence explaining how each question is answered by subsequent
   chapters.

### 1.4 Contributions of the Work

1. Introduce the section as implemented contributions, not claims of universal
   novelty.
2. Present the four consolidated contributions.
3. Explain the distinctive combination of target-aware monitoring and local
   collective detection in cautious terms.
4. State that originality is assessed against related work in Chapter 2 and
   supported experimentally in Chapter 5.

The title `Candidate Research Novelty` is unusual in final thesis prose. After
the literature review, prefer `Distinctive Aspects of the Proposed Approach`
or integrate defensible originality directly into the contribution narrative.

### 1.5 Scope and Limitations

1. Define the host/node-level scope and selected security domains.
2. State that the system performs monitoring and local rule-based detection.
3. Exclude enforcement, distributed correlation and cluster-wide contextual
   anomalies.
4. Clarify target-specific portability and prototype maturity.
5. Add the agreed networking attribution.

### 1.6 Methodology

1. Study Tracee and relevant literature to derive reusable patterns.
2. Establish requirements for the Rocky Linux target.
3. Develop incrementally across kernel collection, userspace processing and
   detection.
4. Validate event behavior with controlled commands and consistency tests.
5. Evaluate detector scenarios and runtime overhead using defined profiles.

This section should describe the research and engineering method, not repeat
the system architecture.

### 1.7 Thesis Structure

Use one concise paragraph or list covering Chapters 2-6. Write it only after
their final boundaries are stable. The current six-chapter summary is broadly
correct and needs only terminology updates.

## Decisions requiring author, company or supervisor approval

1. Approve the final wording of the three research questions.
2. Decide whether the thesis may explicitly name the company and describe its
   operational requirements.
3. Define attribution and evaluation scope for networking contributions.
4. Confirm whether `anomaly` is acceptable for rule-based detections or should
   be replaced by `security pattern`.
5. Confirm the benchmark profile used for the 5% CPU objective and whether the
   target is a requirement, aspiration or acceptance criterion.
6. Decide whether Kubernetes deployment is implemented work or future
   integration.
7. Decide which candidate novelty is defensible after the literature review.
8. Confirm formal requirements such as page limits, citation style and expected
   number of research questions.

## Final recommendation

The canonical outline can be retained, but the Chapter 1 LaTeX draft should be
rewritten around the problem rather than patched paragraph by paragraph. The
three research questions should use the recommended formulations above. The
contributions should be consolidated from eight features into four coherent
results, and the final chapter should avoid a standalone novelty claim until
the literature evidence is available.

No final prose should be drafted before resolving the terminology for
rule-based anomalies, networking attribution and benchmark scope.

