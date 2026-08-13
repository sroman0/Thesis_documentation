# Indice canonico della tesi

## Stato del documento

Questo documento sostituisce la precedente struttura a nove capitoli proposta
in `thesis-outline.md` e allinea la documentazione alla struttura a sette capitoli
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

Presenta quindi i Capitoli 2-7 con una lista sintetica, indicando per ciascuno
contenuto e focus. Il Capitolo 1 non contiene research questions, una sezione
autonoma sui contributi, una sezione di scope o una metodologia separata:
questi elementi vengono sviluppati nei capitoli tecnici e sperimentali.

## Chapter 2 - Background and eBPF architecture

### 2.1 Linux Kernel Observability

### 2.2 eBPF Fundamentals

#### 2.2.1 Execution Model, Verifier and JIT Compilation

#### 2.2.2 Object Files, Loading and Attachment Lifecycle

#### 2.2.3 Hook Selection and Event Semantics

#### 2.2.4 Maps, Helpers and Kernel-to-Userspace Transport

#### 2.2.5 BTF and CO-RE Portability

### 2.3 Related eBPF Runtime Security Tools

#### 2.3.1 Tracee

#### 2.3.2 Other Relevant eBPF Security Tools

### 2.4 Positioning of the Proposed Work

Il Capitolo 2 e' centrato sul percorso eBPF, dalla compilazione all'attachment e
al trasporto verso user space, e sullo stato dell'arte. Policy, detector,
correlazione e classificazione ATT&CK del sistema proposto appartengono al
Capitolo 3; il Capitolo 2 li menziona solo quando sono necessari per confrontare
strumenti esistenti.

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
quel modo. I dettagli kernel-space appartengono al Capitolo 4; runtime Go,
policy, detector e presentazione appartengono al Capitolo 5.

## Chapter 4 - Kernel-Space Event Collection Implementation

### 4.1 Build Toolchain and Project Organization

### 4.2 eBPF Program Implementation

#### 4.2.1 Shared Event-Construction Framework

#### 4.2.2 Process Lifecycle and Executable Transitions

#### 4.2.3 Identity, Credentials, Process Control and Limits

#### 4.2.4 File and Filesystem Security

#### 4.2.5 Memory, Namespace and Cgroup Transitions

#### 4.2.6 Kernel Integrity and Kernel-Facing Activity

### 4.3 Binary Event Contract and Kernel Transport

### 4.4 Kernel-Side UID Filtering

### 4.5 Compatibility and Verifier Challenges

### 4.6 Kernel-Space Implementation Summary

Il Capitolo 4 termina al confine kernel-to-userspace. Documenta i produttori,
il contratto binario, il transport e i vincoli del kernel target, senza
anticipare decoder, policy, detector o risultati sperimentali.

## Chapter 5 - User-Space Runtime and Detection Implementation

### 5.1 Loader, Probe Registry and Attachment Lifecycle

### 5.2 User-Space Decoder and Event Registry

### 5.3 User-Space Event Selection and Admission

### 5.4 Policy and Detector Engine

#### 5.4.1 YAML Policy and Detector Definitions

#### 5.4.2 Point and Collective Detection

#### 5.4.3 Local Correlation Strategies and Deduplication

#### 5.4.4 MITRE ATT&CK Metadata Propagation

### 5.5 Output and Structured Logging

### 5.6 Containerization and Deployment Considerations

### 5.7 User-Space Implementation Summary

Il Capitolo 5 documenta il runtime Go e il livello di detection realmente
implementati. Gli elenchi esaustivi di eventi, flag e regole restano materiale
da appendice.

La prosa del capitolo e' stata azzerata prima del nuovo audit agentico. La
struttura sopra e' una baseline provvisoria: il workflow HTML in
`chapter-05-agent-workflow.html` puo' proporre accorpamenti motivati, purche'
preservi i confini con i Capitoli 2, 3, 4 e 6 e mantenga stabili o migri
esplicitamente le label usate dai riferimenti LaTeX.

## Chapter 6 - Experimental Evaluation

### 6.1 Evaluation Objectives and Experimental Protocol

### 6.2 Minikube Environment and Three-Container Testbed

### 6.3 AES-beta Workload and Controlled Attack Scenario

### 6.4 Functional Evaluation

#### 6.4.1 Cross-Container Event Visibility

#### 6.4.2 Collective Detector Validation

#### 6.4.3 Benign Control and Repeated Runs

### 6.5 Performance Evaluation

#### 6.5.1 Metrics, Sampling and Benchmark Profiles

#### 6.5.2 Resource Consumption and Workload Duration

#### 6.5.3 Effect of Kernel-Side UID Filtering

### 6.6 Discussion of Results

### 6.7 Threats to Validity and Current Limitations

### 6.8 Evaluation Summary

Il Capitolo 6 usa la challenge AES-beta in un Pod Minikube con target, trigger e
monitor per valutare la visibilita' eBPF tra container e il detector collective
costruito sulla sequenza shell, comando e accesso al flag. Il controllo benigno
e i run ripetuti restano separati dal caso offensivo. ATT&CK viene riportato
come classificazione dell'alert, non come analisi autonoma di copertura.

Il capitolo distingue rigorosamente risultati funzionali conclusi, calibrazione
prestazionale esplorativa e benchmark replicati. Le cinque repliche per profilo
supportano il risultato inferiore al 5% per la CPU user-space del monitor
filtered. Questo risultato non deve essere trasformato in una dichiarazione di
overhead totale, poiche' il lavoro eBPF non e' interamente attribuito al cgroup
del monitor e il nodo rimane quasi saturo dal workload.

## Chapter 7 - Conclusions and Future Work

### 7.1 Summary of the Work

### 7.2 Main Results

### 7.3 Limitations

### 7.4 Future Work

## Appendices

### Appendix A - Event and Hook Catalogue

### Appendix B - Policy and Detector Examples

### Appendix C - Reproducibility Commands

Le appendici evitano che il corpo della tesi diventi un manuale operativo.
