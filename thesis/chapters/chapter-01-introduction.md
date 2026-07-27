# Dossier Capitolo 1 - Introduction

## Stato

Preparazione completata per avviare ricerca, fact-checking e revisione
editoriale tramite agenti separati. Il file LaTeX e l'abstract non sono stati
modificati in questa fase.

L'ondata 1 degli agenti e' stata completata. Research questions, contributi,
scope e claims policy sono ora consolidati nella
[sintesi editoriale vincolante](chapter-01-editorial-synthesis.md), che prevale
su questo dossier quando una proposta iniziale e' stata raffinata.

## Obiettivo del capitolo

Spiegare perche' il problema e' rilevante, quale problema affronta il sistema,
quali obiettivi e domande guidano il lavoro, quali contributi sono stati
implementati e quali limiti definiscono il perimetro della tesi.

Il capitolo non deve ancora insegnare nel dettaglio eBPF, descrivere tutte le
funzioni del tool o presentare risultati numerici completi.

## Struttura da seguire

La struttura canonica e' definita in
[definitive-outline.md](../definitive-outline.md):

1. Context and Motivation;
2. Problem Statement;
3. Objectives and Thesis Structure.

## Audit del testo LaTeX corrente

Il file `Thesis/content/chapters/chapter1.tex` offre una buona prima base ma
richiede queste correzioni:

- separare contesto, problema e obiettivi;
- aggiungere citazioni alle affermazioni generali su eBPF, verifier, JIT e
  runtime security;
- evitare di descrivere eBPF come tecnologia completamente "new";
- aggiornare gli obiettivi includendo policy, detector, correlazione collective
  e MITRE ATT&CK;
- sostituire i TODO ancora presenti;
- aggiornare lo scope, che considera ancora anomaly detection e policy come
  estensioni future;
- esplicitare metodologia e contributi;
- chiarire il ruolo di Tracee come riferimento, non come implementazione
  replicata integralmente;
- chiarire il perimetro della componente networking sviluppata in
  collaborazione;
- mantenere la struttura finale a sei capitoli.

## Fatti del progetto gia' verificabili

### Target e vincoli

- Target principale: Rocky Linux 8.10.
- Kernel: `4.18.0-553.109.1.el8_10.x86_64` su `x86_64`.
- Il kernel include backport RHEL, quindi il numero di versione da solo non
  descrive tutte le feature eBPF disponibili.
- Il runtime usa Go, CGO e `libbpfgo`; i programmi eBPF sono scritti in C.
- Il prototipo opera come monitor host-level e limita la correlazione al
  contesto locale disponibile sul sistema osservato.
- La containerizzazione e' implementata, ma un eventuale deployment Kubernetes
  appartiene agli sviluppi futuri e non costituisce un contributo della tesi.

Fonti interne principali:

- `documentation/implementation/overview.md`;
- `documentation/implementation/userspace-lifecycle.md`;
- `documentation/implementation/docker.md`;
- `demo_project/README.md`.

### Pipeline realizzata

La pipeline corrente comprende:

```text
kernel hook eBPF
  -> filtro UID kernel-side opzionale
  -> costruzione evento
  -> perf buffer
  -> runtime Go
  -> decoder
  -> selezione e policy
  -> detector engine
  -> evento o alert
```

Il ring buffer rimane supportato ma il perf buffer e' il trasporto operativo
principale.

Fonti interne principali:

- `documentation/implementation/overview.md`;
- `documentation/implementation/event-buffer.md`;
- `documentation/decisions/decision-log.md`;
- `demo_project/pkg/ebpf/project.go`.

### Detection realizzata

- Policy YAML per selezionare eventi e scenari.
- Detector YAML point e collective.
- Correlazione locale process-aware con finestre temporali brevi.
- Dedup temporale degli alert.
- Output eventi e alert separato, inclusa modalita' `--alerts-only`.
- Propagazione negli alert di tactic e technique MITRE ATT&CK dichiarate dal
  detector.

Fonti interne principali:

- `documentation/next-steps/detectors-and-correlations.md`;
- `documentation/implementation/tracee-detectors-policies-runtime.md`;
- `demo_project/pkg/detectors/`;
- `demo_project/rules/detectors/`;
- `demo_project/rules/policies/`.

### Copertura security rilevante

Il tool osserva famiglie di eventi relative a:

- process lifecycle ed execution;
- credenziali e privilege changes;
- filesystem e file permissions;
- memory mappings e cross-process memory access;
- namespace e mount;
- kernel modules e kernel tampering signals;
- eBPF objects and operations;
- cgroup e selected networking activity.

Il Capitolo 1 deve citare queste famiglie, non elencare ogni hook. Il catalogo
completo appartiene al Capitolo 4 o all'Appendice A.

Fonte interna principale:

- `documentation/implementation/hooks.md`.

## Obiettivo proposto

Progettare, implementare e valutare un prototipo di runtime security monitoring
basato su eBPF, orientato a processi e segnali di sicurezza, compatibile con il
kernel Rocky Linux target e capace di trasformare eventi kernel in telemetria e
alert configurabili.

La formulazione inglese finale spetta all'agente editoriale dopo il controllo
del fact-checker.

## Contributi implementativi candidati

1. Una pipeline kernel-to-userspace modulare per eventi process e security su
   Rocky Linux 8.10.
2. Un contratto eventi custom con registry coerente tra eBPF, decoder e probe.
3. Un sistema dichiarativo userspace di policy e detector YAML.
4. Supporto a point anomalies e collective anomalies locali con correlazione
   process-aware e finestre brevi.
5. Propagazione di tactic e technique MITRE ATT&CK negli alert.
6. Separazione tra eventi, alert e logging diagnostico strutturato.
7. Un primo filtro UID kernel-side per ridurre il traffico verso userspace.
8. Packaging containerizzato del runtime e dell'oggetto eBPF.

## Candidate novelty da verificare

Le seguenti non devono ancora essere presentate come novelty dimostrate:

- combinazione di detector YAML point e collective locali con correlazione
  process-aware in un agent node-level;
- uso esplicito di MITRE ATT&CK come contratto descrittivo degli alert e futura
  base di selezione/copertura;
- adattamento target-aware di pattern Tracee a un kernel Rocky Linux 4.18 con
  una architettura piu' piccola e controllabile;
- separazione tra detection locale e analisi distribuita demandata a un
  eventuale sistema esterno;
- compromesso tra genericita' del detector engine e pre-filtering minimale nel
  kernel.

Per rivendicare originalita' servono related work, confronto funzionale e
risultati sperimentali.

## Scope proposto

### Incluso

- monitoraggio runtime host/node-level;
- eventi di processi e sicurezza;
- telemetria kernel decodificata in userspace;
- policy e detector configurabili;
- point e collective anomalies locali;
- classificazione MITRE ATT&CK;
- valutazione funzionale e prestazionale sul target.

### Escluso o non ancora maturo

- enforcement o blocco delle operazioni;
- contextual anomalies basate su stato globale del cluster;
- correlazione distribuita tra nodi;
- compatibilita' universale con kernel e architetture differenti;
- equivalenza funzionale con Tracee;
- garanzia definitiva del target CPU sotto il 5% prima della chiusura dei
  benchmark;
- supporto production-grade completo.

## Fonti esterne necessarie

Gli agenti dovranno trovare fonti primarie per:

- definizione ed evoluzione di eBPF;
- verifier, JIT, mappe, helper e attachment model;
- vantaggi e limiti del kernel-level runtime monitoring;
- BTF e CO-RE;
- Tracee e altri strumenti eBPF di runtime security;
- point, contextual e collective anomalies;
- MITRE ATT&CK;
- vincoli di sicurezza dei workload eBPF in ambienti containerizzati.

Le fonti generali non devono essere ricavate esclusivamente da blog o dalla
documentazione interna del progetto.

## Decisioni ancora necessarie

1. Formalizzare con relatore e azienda l'attribuzione della componente
   networking, che nel frattempo resta fuori dai contributi principali.
2. Definire il profilo benchmark rappresentativo usato per verificare il target
   del 5% di un core.
3. Completare nel Capitolo 2 il confronto necessario a sostenere la candidate
   novelty target-aware e bounded.
4. Confermare eventuali limiti di pagine e requisiti formali del corso di
   laurea.

## Materiale che gli agenti non devono modificare

- `Thesis/content/abstract.tex`;
- risultati benchmark non ancora conclusivi;
- scope networking senza una decisione di attribuzione;
- codice del tool.

## Criteri di completamento del Capitolo 1

- struttura conforme all'indice canonico;
- nessun TODO o placeholder;
- obiettivo generale e descrizione dei Capitoli 2-6 coerenti;
- ogni affermazione generale supportata da una citazione;
- nessuna novelty presentata come dimostrata senza confronto;
- nessun dettaglio implementativo che appartenga ai Capitoli 3 o 4;
- terminologia conforme a `terminology-and-style.md`;
- compilazione LaTeX senza errori e riferimenti mancanti;
- revisione finale approvata dall'autore.

## Audit bibliografico e acronimi del 2026-07-27

- Tutte le chiavi citate nel capitolo sono definite in `bibliography.bib`.
- Non sono presenti chiavi BibTeX duplicate.
- Sono state aggiunte fonti ufficiali per il backporting RHEL e per
  `libbpfgo`.
- La fonte Kubernetes DaemonSet e' stata rimossa perche' il deployment
  Kubernetes non fa piu' parte dell'argomentazione del Capitolo 1.
- Le fonti Tracee non ancora citate sono mantenute per il related work del
  Capitolo 2.
- Il glossario definisce e usa `eBPF`, `CPU` e `MITRE ATT&CK`.
- `references.bib` e' un residuo del template, contiene soltanto una voce su
  Turing e non viene caricato da `main.tex`; il file bibliografico effettivo e'
  `bibliography.bib`.
- La compilazione completa e' attualmente bloccata dall'installazione TeX
  locale, nella quale manca `biblatex.sty`. I controlli statici su citazioni,
  acronimi e sintassi delle modifiche sono invece completati.
