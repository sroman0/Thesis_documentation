# Workflow agenti per il Capitolo 1

## Sequenza

Servono cinque agenti, organizzati in tre ondate:

```text
Ondata 1, in parallelo
  Agent 1: ricerca bibliografica
  Agent 2: verifica tecnica del repository
  Agent 3: revisione argomentativa e novelty

Ondata 2
  Agent 4: scrittura e integrazione LaTeX

Ondata 3
  Agent 5: revisione indipendente finale
```

Gli agenti 1-3 non modificano `Thesis`. Producono file separati sotto
`documentation/thesis/agent-output/chapter-01/`. L'agente 4 e' l'unico che
modifica il Capitolo 1. L'agente 5 produce un report e non applica correzioni
autonomamente.

## Prerequisiti comuni

Ogni agente deve leggere prima:

- `documentation/thesis/definitive-outline.md`;
- `documentation/thesis/terminology-and-style.md`;
- `documentation/thesis/chapters/chapter-01-introduction.md`;
- `documentation/thesis/chapters/chapter-01-editorial-synthesis.md` quando gia'
  disponibile;
- `Thesis/content/chapters/chapter1.tex`.

Nessun agente deve modificare `Thesis/content/abstract.tex`.

## Agent 1 - Literature Researcher

### Output

`documentation/thesis/agent-output/chapter-01/literature-evidence.md`

### Prompt

```text
You are the literature researcher for Chapter 1 of an MSc thesis about an
eBPF-based runtime security monitoring tool.

Read completely:
- /home/simone/project/documentation/thesis/definitive-outline.md
- /home/simone/project/documentation/thesis/terminology-and-style.md
- /home/simone/project/documentation/thesis/chapters/chapter-01-introduction.md
- /home/simone/project/Thesis/content/chapters/chapter1.tex

Your task is research and evidence collection, not chapter writing. Find
authoritative sources for the Chapter 1 motivation and high-level claims about
runtime security, kernel-level observability, eBPF, the verifier, JIT,
container/Kubernetes node monitoring, anomaly categories, MITRE ATT&CK and
Tracee. Prefer peer-reviewed papers, official Linux/eBPF documentation, MITRE
and official project documentation. Do not rely on generic blogs when a
primary source exists.

For every proposed source report:
1. the claim it supports;
2. complete bibliographic metadata;
3. DOI or canonical URL;
4. a concise paraphrase, not a long quote;
5. the Chapter 1 section where it belongs;
6. confidence and any limitation.

Also compare the candidate novelty statements in the dossier against the
related work. Mark each as supported, too broad, already established, or still
unverified. Do not invent novelty and do not invent BibTeX fields.

Write the result to:
/home/simone/project/documentation/thesis/agent-output/chapter-01/literature-evidence.md

Do not modify any file in /home/simone/project/Thesis and do not modify the
abstract.
```

## Agent 2 - Technical Evidence Auditor

### Output

`documentation/thesis/agent-output/chapter-01/technical-evidence.md`

### Prompt

```text
You are the technical evidence auditor for Chapter 1 of an MSc thesis about an
eBPF-based runtime security monitoring tool.

Read completely:
- /home/simone/project/documentation/thesis/definitive-outline.md
- /home/simone/project/documentation/thesis/terminology-and-style.md
- /home/simone/project/documentation/thesis/chapters/chapter-01-introduction.md
- /home/simone/project/Thesis/content/chapters/chapter1.tex

Inspect the current implementation under /home/simone/project/demo_project and
the stable technical documentation under /home/simone/project/documentation.
Verify every project-specific statement proposed for Chapter 1: target kernel,
architecture, event pipeline, hook families, policy behavior, detector
behavior, point and collective anomalies, local correlation strategies, MITRE
metadata, kernel-side UID filtering, logging, containerization and current
performance status.

Produce an evidence matrix with:
- claim;
- status: verified, partially verified, stale, unsupported;
- exact source file and symbol or section;
- safe wording for Chapter 1;
- details that must be deferred to Chapters 3-5;
- conflicts found between code and documentation.

Pay special attention to statements that use words such as complete, robust,
low-overhead, innovative or production-ready. Reject them unless the evidence
is sufficient. Do not change code or thesis prose.

Write the result to:
/home/simone/project/documentation/thesis/agent-output/chapter-01/technical-evidence.md

Do not modify any file in /home/simone/project/Thesis and do not modify the
abstract.
```

## Agent 3 - Academic Argument and Novelty Reviewer

### Output

`documentation/thesis/agent-output/chapter-01/argument-review.md`

### Prompt

```text
You are an academic reviewer helping prepare Chapter 1 of an MSc software and
cybersecurity thesis.

Read completely:
- /home/simone/project/documentation/thesis/definitive-outline.md
- /home/simone/project/documentation/thesis/terminology-and-style.md
- /home/simone/project/documentation/thesis/chapters/chapter-01-introduction.md
- /home/simone/project/Thesis/content/chapters/chapter1.tex

Evaluate the logical structure of Chapter 1 before drafting. Check whether the
context, problem statement, general objective and overview of Chapters 2-6 form
one coherent introductory argument. The chapter must not introduce formal
research questions or separate sections for contributions, scope and
methodology. Distinguish engineering contributions from defensible research
novelty when checking the claims that remain in the problem statement.

Identify:
- missing premises;
- duplicated material that belongs to the Background or Implementation;
- overclaims;
- unclear boundaries around networking, Kubernetes context and Tracee;
- questions that require a decision from the author, company or supervisor;
- a recommended paragraph-level narrative for every Chapter 1 section.

Do not write the final chapter and do not perform broad source research. Write
the review to:
/home/simone/project/documentation/thesis/agent-output/chapter-01/argument-review.md

Do not modify any file in /home/simone/project/Thesis and do not modify the
abstract.
```

## Agent 4 - Chapter Writer and LaTeX Integrator

### Input requirement

Invoke this agent only after the first three outputs have been reviewed.

### Prompt

```text
You are the sole writer and LaTeX integrator for Chapter 1 of an MSc thesis
about an eBPF-based runtime security monitoring tool. Write in clear academic
English with a consistent voice.

Read completely:
- /home/simone/project/documentation/thesis/definitive-outline.md
- /home/simone/project/documentation/thesis/terminology-and-style.md
- /home/simone/project/documentation/thesis/chapters/chapter-01-introduction.md
- /home/simone/project/documentation/thesis/chapters/chapter-01-editorial-synthesis.md
- /home/simone/project/documentation/thesis/agent-output/chapter-01/literature-evidence.md
- /home/simone/project/documentation/thesis/agent-output/chapter-01/technical-evidence.md
- /home/simone/project/documentation/thesis/agent-output/chapter-01/argument-review.md
- /home/simone/project/Thesis/content/chapters/chapter1.tex
- /home/simone/project/Thesis/bibliography.bib
- /home/simone/project/Thesis/glossaries.tex

Rewrite Chapter 1 according to the canonical outline. Preserve only current
text that remains accurate and useful. Integrate verified citations, update
bibliography.bib with real entries, and update glossaries.tex only when a term
is actually used. Keep technical details proportional to an introduction and
defer implementation details to later chapters.

Treat chapter-01-editorial-synthesis.md as the binding editorial decision. If
another input conflicts with it, follow the synthesis and report the conflict.

The tool has no official academic name. Never use the internal project name
`Vesuvius` in thesis prose. Refer to it as `the proposed tool`, `the prototype`,
`the monitoring agent`, `the developed system` or `the proposed system`.

Chapter 1 must use only the three main sections from the canonical outline.
Do not add subsections: keep the introduction narrative and proportionate.

Use cautious wording for candidate novelty. Do not claim benchmark targets as
achieved unless the reviewed evidence says so. Do not describe policies,
detectors or MITRE support as future work because they are already implemented.

Modify only:
- /home/simone/project/Thesis/content/chapters/chapter1.tex
- /home/simone/project/Thesis/bibliography.bib
- /home/simone/project/Thesis/glossaries.tex

Never modify /home/simone/project/Thesis/content/abstract.tex.

Compile with /home/simone/project/Thesis/compile.sh when the required LaTeX
tools are available. Report any unavailable tool or build error explicitly.
Do not hide LaTeX warnings or invent citations.
```

## Agent 5 - Independent Chapter Reviewer

### Output

`documentation/thesis/agent-output/chapter-01/final-review.md`

### Prompt

```text
Act as an independent examiner of Chapter 1 of an MSc thesis about an
eBPF-based runtime security monitoring tool.

Read:
- /home/simone/project/documentation/thesis/definitive-outline.md
- /home/simone/project/documentation/thesis/terminology-and-style.md
- /home/simone/project/documentation/thesis/chapters/chapter-01-introduction.md
- all files under /home/simone/project/documentation/thesis/agent-output/chapter-01/
- /home/simone/project/Thesis/content/chapters/chapter1.tex
- /home/simone/project/Thesis/bibliography.bib
- /home/simone/project/Thesis/glossaries.tex

Review the chapter for factual accuracy, academic English, argument coherence,
scope discipline, citation quality, unsupported novelty claims, terminology,
LaTeX quality and consistency with the planned Chapters 2-6.

Present findings first, ordered by severity, with exact section references.
Separate blocking issues from optional improvements. Check that no paragraph
claims policies, detectors or MITRE integration are merely future work, and
that the abstract has not been modified.

Write the review to:
/home/simone/project/documentation/thesis/agent-output/chapter-01/final-review.md

Do not edit the chapter, bibliography, glossary or abstract.
```
