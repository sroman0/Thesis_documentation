# Dossier Capitolo 2 - Background and eBPF architecture

## Obiettivo

Fornire al lettore le conoscenze necessarie per comprendere la raccolta
kernel-space e il percorso di un programma eBPF, dalla compilazione al
trasporto degli eventi verso user space, e posizionare il lavoro rispetto allo
stato dell'arte. Il capitolo deve rispondere a due domande:

1. quali concetti di Linux ed eBPF sono necessari per capire il sistema;
2. rispetto a quali strumenti e modelli esistenti si colloca il lavoro.

Il capitolo non descrive nel dettaglio il codice del prototipo. Nomi di
funzioni, registri probe, layout binari, flag CLI e workflow di build
appartengono ai Capitoli 3 e 4.

## Struttura canonica

### 2.1 Linux Kernel Observability

Introduce il kernel come punto di osservazione di processi, credenziali,
filesystem, memoria e operazioni security-sensitive. Confronta brevemente
telemetria applicativa, audit/tracing tradizionale ed eBPF, senza trasformare
la sezione in una storia completa del tracing Linux.

### 2.2 eBPF Fundamentals

#### 2.2.1 Execution Model, Verifier and JIT Compilation

Spiega caricamento, verifica statica, esecuzione event-driven e compilazione
JIT. Evitare formulazioni assolute come "the verifier guarantees safety".

#### 2.2.2 Object Files, Loading and Attachment Lifecycle

Spiega il passaggio dal sorgente all'oggetto ELF, il ruolo del loader e di
libbpf, la creazione degli oggetti kernel, la separazione tra loading e
attachment, e la durata di programmi, mappe e link. Pinning e file descriptor
vanno introdotti come concetti di ownership, senza anticipare il loader concreto
del Capitolo 5.

#### 2.2.3 Hook Selection and Event Semantics

Distingue program type, hook e meccanismo di attachment. Introduce soltanto le
famiglie rilevanti per tracing e security monitoring.

#### 2.2.4 Maps, Helpers and Kernel-to-Userspace Transport

Spiega mappe array, hash, per-CPU e condivise, helper, perf buffer e ring buffer
come primitive. Distingue stato persistente, workspace temporaneo,
configurazione da user space e trasporto. Il confronto prestazionale specifico
del prototipo appartiene al Capitolo 6.

#### 2.2.5 BTF and CO-RE Portability

Introduce BTF, relocation CO-RE e ruolo di libbpf. Deve chiarire che CO-RE
riduce, ma non elimina, i problemi di compatibilita' tra kernel.

### 2.3 Related eBPF Runtime Security Tools

#### 2.3.1 Tracee

Descrive Tracee come principale riferimento architetturale: eventi eBPF,
policy, detector/signature, dipendenze e filtering. Evitare un walkthrough del
codice sorgente, che e' materiale di studio interno.

#### 2.3.2 Other Relevant eBPF Security Tools

Confronta almeno Falco e Tetragon, aggiungendo altri strumenti soltanto quando
portano una differenza rilevante. Il confronto deve usare criteri omogenei:
fonti evento, modello di regole/policy, correlazione, enforcement, output,
portabilita' e obiettivo operativo.

### 2.4 Positioning of the Proposed Work

Riassume il gap affrontato senza dichiarare novelty non dimostrate. Il
posizionamento approvato e':

- focus su process and security monitoring;
- target Rocky Linux 8.10 con kernel enterprise 4.18 e backport;
- contratto eventi coerente tra kernel e Go userspace;
- separazione tra raccolta eBPF e analisi nel runtime Go;
- policy e detector come elementi di posizionamento architetturale, rinviando
  definizione e funzionamento del sistema proposto al Capitolo 3.

## Confini editoriali

- Non usare il nome interno del tool.
- Non presentare Kubernetes come contributo.
- Non attribuire all'autore la parte networking sviluppata in collaborazione.
- Non dichiarare overhead inferiore al 5% come risultato acquisito.
- Non descrivere i detector rule-based come machine learning.
- Non introdurre una sezione autonoma sulla tassonomia delle anomalie: point e
  collective detector sono definiti nel Capitolo 3 in base all'evidenza che
  consumano.
- Non spiegare qui il policy layer del prototipo o la propagazione ATT&CK.
- Non affermare equivalenza funzionale con Tracee.
- Non usare documentazione interna come fonte per nozioni generali.
- Preferire paper peer-reviewed, documentazione Linux, specifiche e
  documentazione ufficiale dei progetti.

## Glossario

La nuova trattazione del ciclo di vita eBPF usa l'acronimo `ELF`, definito come
`Executable and Linkable Format` nel glossario della tesi.

## Materiale interno di partenza

- `Thesis/content/chapters/chapter1.tex`;
- `Thesis/bibliography.bib`;
- `documentation/thesis/definitive-outline.md`;
- `documentation/thesis/terminology-and-style.md`;
- `documentation/implementation/overview.md`;
- `documentation/implementation/hooks.md`;
- `documentation/implementation/tracee-detectors-policies-runtime.md`;
- `documentation/implementation/tracee-policies.md`;
- `documentation/report.md`;
- repository `demo_project/`;
- repository `tracee/`.

## Output attesi prima della scrittura

1. dossier di fonti su Linux observability ed eBPF;
2. dossier sul ciclo di vita di oggetti, programmi, mappe e attachment eBPF;
3. confronto strutturato tra Tracee, Falco, Tetragon e strumenti pertinenti;
4. audit tecnico del posizionamento rispetto a quanto implementato;
5. sintesi editoriale con claim-to-source matrix e piano per paragrafi;
6. solo dopo questi passaggi, stesura LaTeX da parte di un unico agente.
