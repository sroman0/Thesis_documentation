# Workflow agenti - Capitolo 3

> **Nota sulla numerazione corrente:** questo workflow precede la separazione
> dell'implementazione in Capitolo 4 kernel-space e Capitolo 5 user-space.
> I riferimenti storici alla valutazione nel Capitolo 5 indicano ora il
> Capitolo 6; i dettagli implementativi vanno assegnati a 4 o 5 secondo il
> confine corrente. L'indice canonico resta `definitive-outline.md`.

## Strategia

Il lavoro segue tre ondate:

1. quattro agenti analizzano in parallelo requisiti e componenti, senza
   modificare la tesi;
2. dopo un audit umano e una sintesi editoriale unica, un agente scrive il
   capitolo;
3. un revisore indipendente controlla coerenza tecnica, fonti e separazione tra
   design e implementazione.

Gli agenti della prima ondata devono lavorare su file diversi. Il codice e'
fonte primaria per descrivere il comportamento implementato; la documentazione
fornisce storia e motivazioni, ma ogni divergenza deve essere risolta a favore
del codice e segnalata.

## Ondata 1 - Agenti Da Avviare In Parallelo

### Agente 1 - Requirements and Constraints Auditor

Output:

`documentation/thesis/agent-output/chapter-03/requirements-and-constraints-audit.md`

Prompt:

```text
You are the requirements and constraints auditor for Chapter 3 of an MSc
thesis about an eBPF-based runtime security monitoring system.

Read completely:
- /home/simone/project/documentation/thesis/definitive-outline.md
- /home/simone/project/documentation/thesis/terminology-and-style.md
- /home/simone/project/documentation/thesis/chapters/chapter-03-requirements-system-design.md
- /home/simone/project/Thesis/content/chapters/chapter1.tex
- /home/simone/project/Thesis/content/chapters/chapter2.tex
- /home/simone/project/documentation/implementation/overview.md
- /home/simone/project/documentation/report.md
- /home/simone/project/demo_project/README.md

Inspect the latest code under /home/simone/project/demo_project, especially
configuration, CLI, probe selection, policy loading, detector loading, output,
logging, and benchmark scripts. Treat code as authoritative when documentation
is stale. Report uncommitted changes separately from the committed baseline.

Produce:
1. a traceable list of functional requirements;
2. a traceable list of non-functional and operational requirements;
3. target-environment assumptions and compatibility constraints;
4. a requirement-to-evidence matrix with exact files and symbols;
5. acceptance evidence already available and evidence deferred to Chapter 5;
6. conflicts, obsolete claims, or requirements that are only aspirational;
7. safe academic wording for Sections 3.1 and 3.7.

Preserve these boundaries:
- target Rocky Linux 8.10 and kernel 4.18.0-553.137.1.el8_10.x86_64;
- deterministic rule-based detection, not machine learning;
- observation and detection, not enforcement;
- local point and collective detection, not cluster-wide contextual analysis;
- the 5% CPU figure is a target evaluated per profile, not a universal fact;
- Kubernetes is future operational context, not a thesis contribution;
- never use the internal project name in proposed thesis prose.

Do not modify any Thesis or implementation file. Write only:
/home/simone/project/documentation/thesis/agent-output/chapter-03/requirements-and-constraints-audit.md
```

### Agente 2 - Kernel and Event Architecture Analyst

Output:

`documentation/thesis/agent-output/chapter-03/kernel-event-architecture.md`

Prompt:

```text
You are the kernel-space and event-contract architecture analyst for Chapter 3
of an MSc thesis about an eBPF-based runtime security monitoring system.

Read completely:
- /home/simone/project/documentation/thesis/definitive-outline.md
- /home/simone/project/documentation/thesis/terminology-and-style.md
- /home/simone/project/documentation/thesis/chapters/chapter-03-requirements-system-design.md
- /home/simone/project/Thesis/content/chapters/chapter2.tex
- /home/simone/project/documentation/implementation/overview.md
- /home/simone/project/documentation/implementation/hooks.md
- /home/simone/project/documentation/report.md

Inspect the latest relevant code under:
- /home/simone/project/demo_project/pkg/ebpf/c
- /home/simone/project/demo_project/pkg/ebpf
- /home/simone/project/demo_project/pkg/ebpf/probes
- /home/simone/project/demo_project/pkg/bufferdecoder
- /home/simone/project/demo_project/pkg/events

Reconstruct the design, not a source-code walkthrough. Produce:
1. kernel-space responsibilities and their boundary with Go user space;
2. hook-selection criteria and the distinction between hooks, support probes,
   dependencies, and logical events;
3. attach-time event selection and optional kernel-side UID filtering;
4. the conceptual binary event contract and typed argument model;
5. the current perf-buffer transport and its loss/error boundaries;
6. compatibility decisions required by the target enterprise kernel;
7. two proposed Chapter 3 diagrams described precisely in text;
8. a claim-to-code matrix for Sections 3.2, 3.3 and 3.4;
9. implementation details that must be deferred to Chapter 4.

Verify explicitly that the current runtime does not implement BPF ring-buffer
transport. Do not enumerate every hook in the chapter proposal. Do not describe
CO-RE as complete portability, and do not infer support from the upstream
kernel version alone.

Do not modify any Thesis or implementation file. Write only:
/home/simone/project/documentation/thesis/agent-output/chapter-03/kernel-event-architecture.md
```

### Agente 3 - Go Runtime Architecture Analyst

Output:

`documentation/thesis/agent-output/chapter-03/userspace-runtime-architecture.md`

Prompt:

```text
You are the Go user-space runtime architecture analyst for Chapter 3 of an MSc
thesis about an eBPF-based runtime security monitoring system.

Read completely:
- /home/simone/project/documentation/thesis/definitive-outline.md
- /home/simone/project/documentation/thesis/terminology-and-style.md
- /home/simone/project/documentation/thesis/chapters/chapter-03-requirements-system-design.md
- /home/simone/project/Thesis/content/chapters/chapter1.tex
- /home/simone/project/Thesis/content/chapters/chapter2.tex
- /home/simone/project/documentation/implementation/overview.md
- /home/simone/project/documentation/implementation/output.md
- /home/simone/project/documentation/report.md

Inspect the latest code under:
- /home/simone/project/demo_project/cmd
- /home/simone/project/demo_project/pkg/cmd
- /home/simone/project/demo_project/pkg/config
- /home/simone/project/demo_project/pkg/ebpf
- /home/simone/project/demo_project/pkg/bufferdecoder
- /home/simone/project/demo_project/pkg/events
- /home/simone/project/demo_project/pkg/output
- /home/simone/project/demo_project/pkg/logging

Reconstruct the end-to-end user-space design. Produce:
1. startup, configuration, object loading, attachment, event consumption and
   shutdown lifecycle;
2. decoding, normalisation and event-registry responsibilities;
3. boundaries between raw event output, detector alert output and zap
   operational logging;
4. error propagation and malformed-event handling;
5. how event selection influences probe attachment and later dispatch;
6. a proposed end-to-end sequence diagram described in text;
7. a claim-to-code matrix for Sections 3.2 and 3.5;
8. implementation details that belong only in Chapter 4.

Explain why Go and libbpfgo are suitable without presenting language choice as
novelty. Do not confuse --alerts-only with kernel filtering. Do not claim that
zap formats eBPF event or alert output.

Do not modify any Thesis or implementation file. Write only:
/home/simone/project/documentation/thesis/agent-output/chapter-03/userspace-runtime-architecture.md
```

### Agente 4 - Policy and Detection Architecture Auditor

Output:

`documentation/thesis/agent-output/chapter-03/policy-detection-architecture.md`

Prompt:

```text
You are the policy and detection architecture auditor for Chapter 3 of an MSc
thesis about an eBPF-based runtime security monitoring system.

Read completely:
- /home/simone/project/documentation/thesis/definitive-outline.md
- /home/simone/project/documentation/thesis/terminology-and-style.md
- /home/simone/project/documentation/thesis/chapters/chapter-03-requirements-system-design.md
- /home/simone/project/Thesis/content/chapters/chapter2.tex
- /home/simone/project/documentation/implementation/correlation.md
- /home/simone/project/documentation/next-steps/detectors-and-correlations.md
- /home/simone/project/documentation/implementation/output.md

Inspect the latest code and rule definitions under:
- /home/simone/project/demo_project/pkg/policy
- /home/simone/project/demo_project/pkg/detectors
- /home/simone/project/demo_project/pkg/events
- /home/simone/project/demo_project/rules/policies
- /home/simone/project/demo_project/rules/detectors

Verify every behaviour against code and tests. Produce:
1. the exact architectural distinction between policy, detector and alert;
2. how policies constrain event scope and when this can affect probe
   attachment;
3. how detector paths enable detector loading independently of policy files;
4. the point-detector lifecycle;
5. the collective-detector lifecycle and evidence retention;
6. the implemented process, process_tree, resource, cgroup and composite
   correlation semantics;
7. time-window, pruning and retained-state boundaries;
8. the role and validation limits of MITRE ATT&CK metadata;
9. a claim-to-code-and-test matrix for Section 3.6;
10. unsupported behaviours and Chapter 5 validation needs.

Verify and state these subtleties:
- process identity includes start time;
- process_tree is same-process or immediate parent-to-child, not arbitrary
  ancestry;
- resource correlation requires dev and inode and has no pathname fallback;
- cgroup identity is local and is not Kubernetes workload identity;
- user_session is rejected rather than approximated with UID;
- composite strategies require all components;
- only one partial sequence is retained for a detector correlation key;
- state is bounded by short windows and a per-detector cap;
- policies do not currently select detector IDs or MITRE metadata.

Do not present the policy-detector separation itself as novel. Do not modify any
Thesis or implementation file. Write only:
/home/simone/project/documentation/thesis/agent-output/chapter-03/policy-detection-architecture.md
```

## Passaggio Intermedio - Audit E Sintesi Editoriale

Dopo i quattro output:

1. controllare che ogni agente abbia scritto esclusivamente il proprio file;
2. confrontare ogni claim con il branch e il commit analizzati;
3. separare modifiche committed da esperimenti non committed;
4. risolvere conflitti a favore del comportamento verificato nel codice;
5. eliminare ripetizioni con il Capitolo 2;
6. decidere quali diagrammi includere;
7. costruire una matrice requisito-design-evidenza;
8. scrivere `agent-output/chapter-03/editorial-synthesis.md`.

La sintesi deve diventare vincolante prima della stesura LaTeX.

## Ondata 2 - Chapter Writer and LaTeX Integrator

Avviare un solo agente dopo la creazione e l'approvazione di
`editorial-synthesis.md`.

Prompt:

```text
You are the sole Chapter 3 writer and LaTeX integrator for an MSc thesis about
an eBPF-based runtime security monitoring system.

Precondition:
- /home/simone/project/documentation/thesis/agent-output/chapter-03/editorial-synthesis.md
  must exist and must have been approved.

If the editorial synthesis is missing, internally inconsistent, or still marks
blocking questions as unresolved, stop without modifying the Thesis repository
and report the blocker.

Read completely:
- /home/simone/project/documentation/thesis/definitive-outline.md
- /home/simone/project/documentation/thesis/terminology-and-style.md
- /home/simone/project/documentation/thesis/chapters/chapter-03-requirements-system-design.md
- /home/simone/project/documentation/thesis/agent-output/chapter-03/editorial-synthesis.md
- /home/simone/project/documentation/thesis/agent-output/chapter-03/requirements-and-constraints-audit.md
- /home/simone/project/documentation/thesis/agent-output/chapter-03/kernel-event-architecture.md
- /home/simone/project/documentation/thesis/agent-output/chapter-03/userspace-runtime-architecture.md
- /home/simone/project/documentation/thesis/agent-output/chapter-03/policy-detection-architecture.md
- /home/simone/project/Thesis/content/chapters/chapter1.tex
- /home/simone/project/Thesis/content/chapters/chapter2.tex
- /home/simone/project/Thesis/content/chapters.tex
- /home/simone/project/Thesis/glossaries.tex
- /home/simone/project/Thesis/bibliography.bib
- /home/simone/project/Thesis/common/packages.tex

Inspect the latest implementation under /home/simone/project/demo_project only
when needed to resolve an ambiguity. Treat the baseline and conflict decisions
recorded in editorial-synthesis.md as binding. Do not silently promote
uncommitted or experimental work into the thesis.

Write Chapter 3 using exactly this structure:

3.1 Design Goals and Constraints
  3.1.1 Target Environment and Compatibility Assumptions
  3.1.2 Functional Requirements
  3.1.3 Non-Functional and Operational Requirements
3.2 Overall System Architecture
  3.2.1 Kernel-Space and User-Space Responsibilities
  3.2.2 End-to-End Event Lifecycle
3.3 Kernel-Space Collection Design
  3.3.1 Hook Selection and Logical Events
  3.3.2 Early Event Selection and Filtering
3.4 Event Contract and Transport Design
  3.4.1 Stable Binary Event Contract
  3.4.2 Perf-Buffer Transport and Failure Boundaries
3.5 User-Space Runtime Design
  3.5.1 Loading, Attachment and Runtime Lifecycle
  3.5.2 Decoding, Normalisation and Event Registration
  3.5.3 Event Output, Alert Output and Operational Logging
3.6 Policy and Detection Architecture
  3.6.1 Policy-Based Event Scope
  3.6.2 Point and Collective Detectors
  3.6.3 Bounded Local Correlation
  3.6.4 Alert Evidence and MITRE ATT&CK Metadata
3.7 Design Decisions and Scope Boundaries

The chapter must fulfil the promise made in Chapter 1: define the target
environment, functional and operational requirements, architectural
components, event flow, and the boundary between kernel-space collection and
user-space processing.

Mandatory technical boundaries:
- the target is Rocky Linux 8.10 with kernel
  4.18.0-553.137.1.el8_10.x86_64;
- vendor backports require capabilities to be verified on the actual target;
- kernel space performs observation, bounded extraction, optional early UID
  filtering, event construction and serialisation;
- Go user space owns lifecycle, decoding, normalisation, policy evaluation,
  detector evaluation, alert generation, presentation and operational logging;
- logical events, physical hooks and support-probe dependencies are distinct;
- the runtime transport is a perf buffer; BPF ring-buffer transport is not
  implemented;
- policies constrain event scope and exact allowlists may narrow probe
  selection before attachment;
- policy files do not activate detector IDs or select detectors by ATT&CK
  metadata;
- detector event requirements are not automatically converted into probe
  dependencies;
- point detectors evaluate one normalised event;
- collective detectors evaluate short ordered sequences with bounded local
  state;
- process correlation uses stable process identity;
- process-tree correlation is limited to the same process or an immediate
  parent-to-child relationship;
- resource, cgroup and composite correlation are local identities, not
  Kubernetes workload attribution;
- user-session correlation, arbitrary ancestry, cluster-wide correlation,
  general complex-event processing and enforcement are not implemented;
- alerts preserve matching evidence and may carry syntactically validated
  MITRE ATT&CK metadata;
- matched-policy provenance is not currently propagated end to end into
  detector alerts;
- the below-five-per-cent CPU objective is profile-specific and belongs to
  evaluation, not an unconditional design guarantee.

Editorial requirements:
- write in clear, natural British English;
- use an academic but readable narrative rather than a source-code walkthrough;
- never use the internal project name;
- do not modify the abstract;
- do not rewrite Chapters 1 or 2;
- avoid repeating the eBPF background and related-work comparison covered by
  Chapter 2; define point and collective detector semantics and the role of
  ATT&CK here because they belong to the system design;
- distinguish requirements from implementation evidence and evaluation
  results;
- explain what and why in Chapter 3; reserve function names, Go packages,
  struct layouts, event IDs, byte offsets, map definitions, YAML fields, CLI
  flags, exact state-key encodings, verifier fixes and test commands for
  Chapter 4 or the appendices;
- the exact 136-byte context layout and the long-record offset correction are
  Chapter 4 evidence, not Chapter 3 exposition;
- describe structured operational logging conceptually; reserve Zap wiring for
  Chapter 4;
- describe stable file-resource identity conceptually; reserve concrete device
  and inode fields for Chapter 4;
- present policy/detector separation as an adopted architectural pattern, not
  as research novelty;
- treat Kubernetes deployment as future context, not an implemented
  contribution;
- do not claim complete event coverage, production readiness, universal
  portability, complete ATT&CK coverage or universal performance compliance.

Requirements presentation:
- use a compact requirements table or a concise structured presentation;
- assign stable identifiers such as FR-01 and NFR-01 only when they improve
  traceability;
- do not reproduce the full evidence matrix from the agent dossier;
- connect requirements explicitly to the architecture that satisfies them;
- defer validation results and acceptance evidence to Chapter 5.

Figures:
- include only diagrams approved by editorial-synthesis.md;
- prefer one overall architecture/data-flow figure and, only if justified, one
  figure explaining physical probes, dependencies and logical events;
- operational logging must appear as a separate diagnostic path;
- do not use Mermaid in the LaTeX source;
- do not invent external image assets or add a new LaTeX graphics dependency
  unless the approved synthesis explicitly requires it and compilation is
  verified.

Citations and terminology:
- reuse verified bibliography entries where they directly support general
  claims;
- do not cite internal documentation as authority for general eBPF or security
  concepts;
- do not invent citations or BibTeX;
- add a bibliography entry only after verifying its metadata;
- add glossary or acronym entries only when the term is used and not already
  defined;
- keep policy, detector, event, alert, hook and logical event distinct.

Allowed modifications:
- create /home/simone/project/Thesis/content/chapters/chapter3.tex;
- update /home/simone/project/Thesis/content/chapters.tex to include it;
- update /home/simone/project/Thesis/glossaries.tex only when required;
- update /home/simone/project/Thesis/bibliography.bib only with verified
  references required by Chapter 3;
- create approved Chapter 3 figure assets under
  /home/simone/project/Thesis/figures/chapter3/ when necessary.

Do not modify implementation code, documentation agent outputs, the abstract,
Chapter 1, Chapter 2, or unrelated template files.

After writing:
1. run the repository's documented Thesis compilation workflow;
2. fix compilation errors introduced by Chapter 3;
3. inspect warnings for undefined references, citations and glossary entries;
4. verify that Chapter 3 appears after Chapter 2;
5. report modified files, compilation result, remaining warnings and any claim
   that could not be included safely.
```

## Ondata 3 - Independent Chapter Reviewer

Avviare un agente diverso dallo scrittore soltanto dopo che il Capitolo 3 e'
stato integrato e compilato.

Prompt:

```text
You are the independent technical and editorial reviewer for Chapter 3 of an
MSc thesis about an eBPF-based runtime security monitoring system.

You did not write the chapter. Review it critically and do not assume that the
writer followed the synthesis correctly.

Read completely:
- /home/simone/project/documentation/thesis/definitive-outline.md
- /home/simone/project/documentation/thesis/terminology-and-style.md
- /home/simone/project/documentation/thesis/chapters/chapter-03-requirements-system-design.md
- /home/simone/project/documentation/thesis/agent-output/chapter-03/editorial-synthesis.md
- all four first-wave outputs under
  /home/simone/project/documentation/thesis/agent-output/chapter-03/
- /home/simone/project/Thesis/content/chapters/chapter1.tex
- /home/simone/project/Thesis/content/chapters/chapter2.tex
- /home/simone/project/Thesis/content/chapters/chapter3.tex
- /home/simone/project/Thesis/content/chapters.tex
- /home/simone/project/Thesis/glossaries.tex
- /home/simone/project/Thesis/bibliography.bib

Inspect the relevant implementation and tests under
/home/simone/project/demo_project to verify every implementation-specific
statement. Check the repository state and distinguish the committed baseline
selected by editorial-synthesis.md from uncommitted experiments.

Review objectives:
1. verify that Chapter 3 fulfils the exact promise in Chapter 1:
   target environment, functional and operational requirements, architecture,
   event flow, and kernel/user-space boundaries;
2. verify consistency with the terminology and technical boundaries established
   in Chapter 2;
3. verify that every requirement is represented by an architectural component
   or is explicitly marked as evaluation pending;
4. identify implementation details that should move to Chapter 4;
5. identify experimental results or performance conclusions that should move
   to Chapter 5;
6. verify every technical claim against code and tests;
7. verify diagrams against the prose and actual data flow;
8. check citations, BibTeX use, glossary entries, cross-references, labels and
   British English;
9. identify repetition, mechanical lists, abrupt transitions and prose that
   reads like generated documentation rather than a thesis chapter;
10. detect overclaims about novelty, portability, coverage, performance,
    production readiness, enforcement or Kubernetes deployment.

Mandatory technical checks:
- perf-buffer transport is described as implemented and ring-buffer transport
  is not;
- support probes remain dependencies rather than public events in the selected
  stable baseline;
- unknown event IDs and malformed records are treated as contract failures,
  not silently accepted;
- policies and detectors retain separate roles;
- exact policy allowlists may narrow probe attachment;
- detector files are loaded independently from policies;
- detector requirements do not automatically enable their required probes;
- policy modes do not imply distinct detector-routing semantics beyond the
  implemented allow/suppress behaviour;
- policy provenance is not claimed as present in emitted detector alerts;
- point and collective detectors are deterministic and rule-based;
- collective order means user-space arrival order;
- process-tree correlation is not described as arbitrary ancestry;
- resource and cgroup identities are local and are not described as path,
  session or Kubernetes workload identities;
- composite correlation requires all configured dimensions;
- only one partial sequence exists for a detector correlation key;
- partial state is bounded, but runtime Flush is not periodically scheduled;
- ATT&CK validation is syntactic and metadata mapping is classification;
- observation and detection are not described as enforcement;
- the five-per-cent CPU objective is not presented as universally achieved.

Chapter-boundary checks:
- Chapter 3 may state that a stable shared event contract exists, but exact
  byte sizes, offsets and the long-record fix belong in Chapter 4;
- Chapter 3 may state that operational logs are separate, but Zap configuration
  and CLI flags belong in Chapter 4;
- Chapter 3 may explain correlation identities, but concrete key encodings,
  device/inode fields, maps, limits and pruning loops belong in Chapter 4;
- benchmark values, pass/fail conclusions and test commands belong in
  Chapter 5 or appendices;
- Kubernetes container orchestration must not be presented as implemented or
  evaluated thesis work.

Compilation review:
- run the documented Thesis compilation workflow without editing source files;
- report compilation failures, undefined references, missing citations,
  glossary warnings and layout problems;
- if PDF text extraction or visual inspection is unavailable, state that
  limitation explicitly rather than assuming the rendered result is correct.

Output format:
- findings first, ordered as Blocking, Major, Moderate and Minor;
- every finding must include the file and exact line or section;
- explain why it is a problem and propose the smallest safe correction;
- include a requirement-to-section coverage table;
- include a technical-claim audit table with Verified, Qualified or Unsupported;
- include a short list of Chapter 4 material found prematurely in Chapter 3;
- include a short list of missing evidence deferred to Chapter 5;
- finish with one verdict: Ready, Ready after minor corrections, or Not ready.

Do not modify the Thesis repository, implementation code, agent dossiers or
editorial synthesis. Write only:
/home/simone/project/documentation/thesis/agent-output/chapter-03/final-review.md
```
