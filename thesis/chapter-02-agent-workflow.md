# Workflow agenti - Capitolo 2

## Strategia

Il lavoro e' diviso in tre ondate:

1. quattro agenti raccolgono in parallelo fonti ed evidenze senza modificare la
   tesi;
2. dopo una sintesi editoriale unica, un solo agente scrive il capitolo;
3. un revisore indipendente controlla testo, fonti e posizionamento.

Gli agenti della prima ondata devono lavorare su file diversi. Questo evita
conflitti e mantiene separate ricerca bibliografica, teoria della detection,
confronto tra strumenti ed evidenza implementativa.

## Ondata 1 - Agenti da avviare in parallelo

### Agente 1 - Linux ed eBPF Foundations Researcher

Output:

`documentation/thesis/agent-output/chapter-02/ebpf-foundations-evidence.md`

Prompt:

```text
You are the Linux and eBPF foundations researcher for Chapter 2 of an MSc
thesis about an eBPF-based runtime security monitoring system.

Read completely:
- /home/simone/project/documentation/thesis/definitive-outline.md
- /home/simone/project/documentation/thesis/terminology-and-style.md
- /home/simone/project/documentation/thesis/chapters/chapter-02-background-related-work.md
- /home/simone/project/Thesis/content/chapters/chapter1.tex
- /home/simone/project/Thesis/bibliography.bib

Research the material required for:
- 2.1 Linux Kernel Observability;
- 2.2.1 Execution Model, Verifier and JIT Compilation;
- 2.2.2 Program Types, Hooks and Attachment Mechanisms;
- 2.2.3 Maps, Helpers and Kernel-to-Userspace Transport;
- 2.2.4 BTF and CO-RE Portability.

Use primary and authoritative sources: Linux kernel documentation, eBPF
specifications, peer-reviewed papers and official libbpf documentation.
Technical blogs may only help locate primary sources and must not be the main
evidence.

For every planned claim provide:
1. the claim in cautious academic English;
2. the source and canonical URL or DOI;
3. the exact Chapter 2 subsection where it belongs;
4. a concise explanation of what the source supports;
5. limitations or wording that would overstate the source;
6. a verified BibTeX proposal.

Explicitly clarify:
- the modern naming of BPF/eBPF;
- what the verifier checks and what it cannot guarantee;
- interpreter versus JIT execution;
- program type versus hook versus attachment mechanism;
- maps, helpers, perf buffers and ring buffers;
- BTF and CO-RE, including their portability limits;
- why enterprise-kernel backports make version-only assumptions unreliable.

Do not describe the proposed tool implementation and do not modify any Thesis
file. Write only:
/home/simone/project/documentation/thesis/agent-output/chapter-02/ebpf-foundations-evidence.md
```

### Agente 2 - Runtime Security and Detection Theory Researcher

Output:

`documentation/thesis/agent-output/chapter-02/runtime-security-evidence.md`

Prompt:

```text
You are the runtime security and detection theory researcher for Chapter 2 of
an MSc thesis about an eBPF-based runtime security monitoring system.

Read completely:
- /home/simone/project/documentation/thesis/definitive-outline.md
- /home/simone/project/documentation/thesis/terminology-and-style.md
- /home/simone/project/documentation/thesis/chapters/chapter-02-background-related-work.md
- /home/simone/project/Thesis/content/chapters/chapter1.tex
- /home/simone/project/Thesis/bibliography.bib
- /home/simone/project/documentation/next-steps/detectors-and-correlations.md

Research the material required for:
- 2.3.1 Point, Contextual and Collective Anomalies;
- 2.3.2 Policies, Detectors and Alert Generation;
- 2.3.3 MITRE ATT&CK as a Classification Framework.

Prefer peer-reviewed anomaly-detection literature, official MITRE ATT&CK
publications, and authoritative runtime-security sources. Keep statistical or
machine-learning anomaly detection clearly separate from deterministic
rule-based detection.

For every planned claim provide:
1. a cautious academic formulation;
2. source, DOI or canonical URL;
3. destination subsection;
4. supported meaning and limitations;
5. verified BibTeX.

Build a terminology table for event, policy, detector, alert, point anomaly,
contextual anomaly and collective anomaly. Explain how a bounded ordered event
sequence can be described without claiming general anomaly discovery.

Explain the value of MITRE ATT&CK as a common classification and communication
framework. Do not portray metadata mapping alone as proof of detection
coverage, but avoid promotional or unnecessarily dismissive wording.

Do not modify any Thesis file. Write only:
/home/simone/project/documentation/thesis/agent-output/chapter-02/runtime-security-evidence.md
```

### Agente 3 - Related Runtime Security Tools Analyst

Output:

`documentation/thesis/agent-output/chapter-02/related-tools-comparison.md`

Prompt:

```text
You are the related-work analyst for Chapter 2 of an MSc thesis about an
eBPF-based runtime security monitoring system.

Read completely:
- /home/simone/project/documentation/thesis/definitive-outline.md
- /home/simone/project/documentation/thesis/terminology-and-style.md
- /home/simone/project/documentation/thesis/chapters/chapter-02-background-related-work.md
- /home/simone/project/documentation/implementation/tracee-detectors-policies-runtime.md
- /home/simone/project/documentation/implementation/tracee-policies.md
- /home/simone/project/Thesis/bibliography.bib

Inspect the local Tracee repository when useful:
- /home/simone/project/tracee

Research Tracee, Falco and Tetragon using official documentation, source
repositories and peer-reviewed papers where available. Add another tool only
when it contributes a materially different design point.

Compare the tools using the same criteria:
- purpose and threat model;
- kernel event sources and eBPF architecture;
- event model;
- policy, rule, signature or detector model;
- support for single-event and multi-event detection;
- filtering and enrichment;
- enforcement versus observation;
- output and integration model;
- portability and kernel assumptions.

Produce:
1. a source-backed narrative for Section 2.4;
2. a comparison table with no empty or speculative cells;
3. a list of design ideas that are established prior art;
4. a list of differences that can safely support Section 2.5;
5. verified BibTeX proposals.

Do not use marketing statements as facts. Do not claim feature equivalence
between the proposed system and these tools. Kubernetes deployment is not a
central comparison criterion.

Do not modify any Thesis file. Write only:
/home/simone/project/documentation/thesis/agent-output/chapter-02/related-tools-comparison.md
```

### Agente 4 - Technical Positioning Auditor

Output:

`documentation/thesis/agent-output/chapter-02/technical-positioning-audit.md`

Prompt:

```text
You are the technical evidence auditor for Chapter 2 of an MSc thesis about an
eBPF-based runtime security monitoring system.

Read completely:
- /home/simone/project/documentation/thesis/definitive-outline.md
- /home/simone/project/documentation/thesis/terminology-and-style.md
- /home/simone/project/documentation/thesis/chapters/chapter-02-background-related-work.md
- /home/simone/project/documentation/implementation/overview.md
- /home/simone/project/documentation/implementation/hooks.md
- /home/simone/project/documentation/report.md
- /home/simone/project/Thesis/content/chapters/chapter1.tex

Inspect relevant code under:
- /home/simone/project/demo_project

Audit the candidate positioning claims for Section 2.5:
- Rocky Linux 8.10 and the 4.18 enterprise-kernel target;
- kernel-to-userspace event contract;
- Go and libbpfgo runtime;
- policies and YAML detectors;
- point and bounded collective detections;
- short-window local correlation by process, resource, cgroup or composite
  identity;
- MITRE ATT&CK metadata;
- kernel-side UID filtering;
- performance target and current benchmark limitations.

For each claim report:
1. verified, partially verified or unsupported;
2. exact source files and symbols;
3. safe academic wording;
4. implementation details that must be deferred to Chapters 3 or 4;
5. evidence still required for Chapter 5.

Identify statements in the Chapter 2 outline that risk overclaiming novelty,
portability, completeness, production readiness or performance. The internal
project name must never appear in proposed thesis prose. Treat Kubernetes as
future context, not as an implemented contribution.

Do not modify code or Thesis files. Write only:
/home/simone/project/documentation/thesis/agent-output/chapter-02/technical-positioning-audit.md
```

## Passaggio intermedio - Sintesi editoriale

Dopo il completamento dei quattro output, un editor deve:

1. eliminare duplicazioni e fonti deboli;
2. verificare ogni BibTeX e URL;
3. costruire una claim-to-source matrix;
4. definire il piano per paragrafi delle Sezioni 2.1-2.5;
5. approvare il confronto tra strumenti;
6. decidere quali acronimi aggiungere;
7. creare `agent-output/chapter-02/editorial-synthesis.md`.

La sintesi diventa vincolante per lo scrittore.

## Ondata 2 - Chapter Writer and LaTeX Integrator

Avviare questo agente soltanto dopo l'approvazione della sintesi.

Prompt:

```text
You are the sole writer and LaTeX integrator for Chapter 2 of an MSc thesis
about an eBPF-based runtime security monitoring system.

Read completely:
- /home/simone/project/documentation/thesis/definitive-outline.md
- /home/simone/project/documentation/thesis/terminology-and-style.md
- /home/simone/project/documentation/thesis/chapters/chapter-02-background-related-work.md
- every file under
  /home/simone/project/documentation/thesis/agent-output/chapter-02/
- /home/simone/project/Thesis/content/chapters/chapter1.tex
- /home/simone/project/Thesis/bibliography.bib
- /home/simone/project/Thesis/glossaries.tex

Treat editorial-synthesis.md as binding. If another agent output conflicts
with it, follow the synthesis and report the conflict.

Write Chapter 2 in clear academic English using exactly the canonical Sections
2.1-2.5 and their approved subsections. Build a coherent narrative rather than
concatenating research notes. Distinguish background knowledge from related
work and reserve implementation details for Chapters 3 and 4.

Requirements:
- never use the internal project name;
- do not modify the abstract;
- cite every general technical or comparative claim;
- add only verified entries to bibliography.bib;
- add glossary entries only for acronyms actually used;
- avoid unsupported claims of novelty, completeness or low overhead;
- present Kubernetes, if mentioned at all, only as future context;
- preserve British English and existing LaTeX style.

Modify only:
- /home/simone/project/Thesis/content/chapters/chapter2.tex
- /home/simone/project/Thesis/content/chapters.tex
- /home/simone/project/Thesis/bibliography.bib
- /home/simone/project/Thesis/glossaries.tex

Run static checks and the available LaTeX build. Report missing local LaTeX
packages separately from content errors.
```

## Ondata 3 - Independent Chapter Reviewer

Prompt:

```text
Act as an independent examiner of Chapter 2 of an MSc thesis about an
eBPF-based runtime security monitoring system.

Read:
- /home/simone/project/documentation/thesis/definitive-outline.md
- /home/simone/project/documentation/thesis/terminology-and-style.md
- /home/simone/project/documentation/thesis/chapters/chapter-02-background-related-work.md
- all Chapter 2 agent outputs;
- /home/simone/project/Thesis/content/chapters/chapter1.tex
- /home/simone/project/Thesis/content/chapters/chapter2.tex
- /home/simone/project/Thesis/bibliography.bib
- /home/simone/project/Thesis/glossaries.tex

Review factual accuracy, source quality, citation placement, academic English,
terminology, logical progression, related-work fairness and separation from
the implementation chapters. Check every comparison claim against its cited
source and identify missing or unused glossary entries.

Present findings first, ordered by severity and with exact section references.
Separate blocking issues from optional improvements. Do not edit the thesis.
Write only:
/home/simone/project/documentation/thesis/agent-output/chapter-02/final-review.md
```
