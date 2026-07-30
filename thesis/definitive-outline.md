# Indice canonico della tesi

## Stato del documento

Questo documento sostituisce la precedente struttura a nove capitoli proposta
in `thesis-outline.md` e allinea la documentazione alla struttura a sei capitoli
gia' anticipata nell'introduzione LaTeX.

L'indice e' considerato stabile a livello di capitoli. Section e subsection
possono ricevere piccoli aggiustamenti durante la scrittura, ma ogni modifica
deve preservare i confini descritti qui per evitare duplicazioni.

L'abstract non rientra nell'attuale fase di lavoro e verra' riscritto soltanto
dopo il completamento della valutazione e delle conclusioni.

## Chapter 1 - Introduction

### 1.1 Context and Motivation

Introduce il bisogno di osservare il comportamento reale dei workload durante
l'esecuzione e i limiti della sola telemetria applicativa o userspace.

Presenta eBPF soltanto ad alto livello: programmabilita' controllata del kernel,
hook e raccolta di telemetria. Verifier, mappe, CO-RE e BTF appartengono al
Capitolo 2.

Inquadra la collaborazione aziendale, il target Rocky Linux e il funzionamento
del prototipo come monitor host-level.

### 1.2 Problem Statement

Definisce il problema: raccogliere eventi di processo e sicurezza dal kernel e
trasformarli in informazioni utilizzabili per detection e analisi.

Riassume i vincoli principali: kernel Rocky Linux 4.18 con backport RHEL,
compatibilita' eBPF, overhead contenuto e assenza di contesto distribuito nel
monitor locale.

### 1.3 Objectives and Thesis Structure

Definisce l'obiettivo complessivo di progettare, implementare e valutare un
runtime security monitor eBPF per processi e segnali security-related.

Presenta quindi i Capitoli 2-6 con una lista sintetica, indicando per ciascuno
contenuto e focus. Il Capitolo 1 non contiene research questions, una sezione
autonoma sui contributi, una sezione di scope o una metodologia separata:
questi elementi vengono sviluppati nei capitoli tecnici e sperimentali.

## Chapter 2 - Background and eBPF architecture

### 2.1 Linux Kernel Observability

### 2.2 eBPF Fundamentals

#### 2.2.1 Execution Model, Verifier and JIT Compilation

#### 2.2.2 Program Types, Hooks and Attachment Mechanisms

#### 2.2.3 Maps, Helpers and Kernel-to-Userspace Transport

#### 2.2.4 BTF and CO-RE Portability

### 2.3 Runtime Security and Anomaly Detection

#### 2.3.1 Point, Contextual and Collective Anomalies

#### 2.3.2 Policies, Detectors and Alert Generation

#### 2.3.3 MITRE ATT&CK as a Classification Framework

### 2.4 Related Runtime Security Tools

#### 2.4.1 Tracee

#### 2.4.2 Other Relevant eBPF Security Tools

### 2.5 Positioning of the Proposed Work

Il Capitolo 2 spiega tecnologie e stato dell'arte. Non descrive nel dettaglio
come il sistema proposto le implementa.

## Chapter 3 - Requirements and System Design

### 3.1 Design Goals and Constraints

#### 3.1.1 Target Environment and Compatibility Assumptions

#### 3.1.2 Functional Requirements

#### 3.1.3 Non-Functional and Operational Requirements

### 3.2 Overall System Architecture

#### 3.2.1 Kernel-Space and User-Space Responsibilities

#### 3.2.2 End-to-End Event Lifecycle

### 3.3 Kernel-Space Collection Design

#### 3.3.1 Hook Selection and Logical Events

#### 3.3.2 Early Event Selection and Filtering

### 3.4 Event Contract and Transport Design

#### 3.4.1 Stable Binary Event Contract

#### 3.4.2 Perf-Buffer Transport and Failure Boundaries

### 3.5 User-Space Runtime Design

#### 3.5.1 Loading, Attachment and Runtime Lifecycle

#### 3.5.2 Decoding, Normalisation and Event Registration

#### 3.5.3 Event Output, Alert Output and Operational Logging

### 3.6 Policy and Detection Architecture

#### 3.6.1 Policy-Based Event Scope

#### 3.6.2 Point and Collective Detectors

#### 3.6.3 Bounded Local Correlation

#### 3.6.4 Alert Evidence and MITRE ATT&CK Metadata

### 3.7 Design Decisions and Scope Boundaries

Il Capitolo 3 spiega cosa deve fare il sistema e perche' e' stato progettato in
quel modo. Nomi di funzioni e dettagli riga per riga appartengono al Capitolo 4.

## Chapter 4 - System Implementation

### 4.1 Build Toolchain and Project Organization

### 4.2 eBPF Program Implementation

#### 4.2.1 Process Lifecycle and Execution Hooks

#### 4.2.2 Credentials and Privilege Hooks

#### 4.2.3 Filesystem, Memory and Namespace Hooks

#### 4.2.4 Kernel Integrity and eBPF Activity Hooks

### 4.3 Loader, Probe Registry and Attachment Lifecycle

### 4.4 Binary Event Contract and Kernel Transport

### 4.5 Userspace Decoder and Event Registry

### 4.6 Event Selection and Kernel-Side Filtering

### 4.7 Policy and Detector Engine

#### 4.7.1 YAML Policy and Detector Definitions

#### 4.7.2 Point and Collective Detection

#### 4.7.3 Local Correlation Strategies and Deduplication

#### 4.7.4 MITRE ATT&CK Metadata Propagation

### 4.8 Output and Structured Logging

### 4.9 Containerization and Deployment Considerations

### 4.10 Compatibility and Verifier Challenges

Il Capitolo 4 documenta il comportamento realmente implementato. Gli elenchi
esaustivi di eventi e comandi possono essere spostati in appendice.

## Chapter 5 - Experimental Evaluation

### 5.1 Evaluation Goals and Methodology

### 5.2 Experimental Environment

### 5.3 Functional Validation and Event Coverage

### 5.4 Detector Case Studies

#### 5.4.1 Point Anomaly Detection

#### 5.4.2 Collective Anomaly Detection

### 5.5 MITRE ATT&CK Coverage Analysis

### 5.6 Performance Evaluation

#### 5.6.1 Benchmark Profiles and Workloads

#### 5.6.2 CPU, Memory and Thread Measurements

#### 5.6.3 Impact of Kernel-Side Filtering

### 5.7 Discussion of Results

### 5.8 Threats to Validity and Current Limitations

Il Capitolo 5 contiene risultati riproducibili e distingue misure concluse da
benchmark esplorativi o interrotti.

## Chapter 6 - Conclusions and Future Work

### 6.1 Summary of the Work

### 6.2 Main Results

### 6.3 Limitations

### 6.4 Future Work

## Appendices

### Appendix A - Event and Hook Catalogue

### Appendix B - Policy and Detector Examples

### Appendix C - Reproducibility Commands

Le appendici evitano che il corpo della tesi diventi un manuale operativo.
