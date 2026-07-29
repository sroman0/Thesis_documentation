# Sintesi editoriale vincolante - Capitolo 1

## Scopo

Questo documento consolida i risultati dell'ondata 1 degli agenti e costituisce
la fonte decisionale principale per la scrittura del Capitolo 1.

Input analizzati:

- `literature-evidence.md`;
- `technical-evidence.md`;
- `argument-review.md`;
- dossier preparatorio del Capitolo 1;
- indice canonico e guida terminologica.

In caso di conflitto, lo scrittore deve seguire questa sintesi e segnalare il
problema, senza scegliere autonomamente una formulazione piu' forte.

## Tesi argomentativa del capitolo

Il Capitolo 1 deve partire dal problema della runtime security, non dalla
tecnologia eBPF.

La linea argomentativa approvata e':

```text
il comportamento runtime richiede osservabilita'
  -> gli eventi kernel forniscono semantica locale rilevante
  -> eBPF offre il meccanismo di instrumentazione scelto
  -> il target Rocky Linux introduce vincoli concreti
  -> gli eventi isolati devono essere normalizzati e valutati
  -> il sistema proposto combina monitoraggio, policy e detection locale
  -> il sistema deve essere valutato per correttezza e overhead
```

Il capitolo presenta il sistema proposto come research prototype. Non deve descriverlo
come prodotto completo, production-ready o nuova categoria di runtime security
tool.

## Terminologia approvata per la detection

La formulazione canonica e':

> rule-based detection of security-relevant point events and short collective
> event patterns

I termini `point anomaly` e `collective anomaly` possono essere usati dopo una
definizione esplicita basata sulla tassonomia di Chandola et al. Deve essere
sempre chiarito che il tool usa regole dichiarative e correlazione temporale,
non machine learning, statistical baselines o unsupervised anomaly detection.

Le contextual anomalies non sono impossibili in assoluto per un node agent.
Sono escluse dal perimetro locale del tool quando richiedono stato globale
del cluster e vengono demandate a un eventuale livello centralizzato.

## Obiettivo principale

Design, implement and evaluate an eBPF-based runtime security monitoring
prototype that collects process and security-relevant kernel events on the
target Rocky Linux environment, normalizes them in user space, and evaluates
them through configurable policies and local detectors.

Lo scrittore puo' migliorare la forma, ma non ampliarne il significato.

## Obiettivi specifici

1. Raccogliere eventi selezionati di processo e sicurezza mediante programmi
   eBPF compatibili con il kernel target.
2. Definire un contratto coerente per trasporto, decoding, registry e output
   degli eventi.
3. Permettere selezione tramite policy e detection locale tramite regole YAML.
4. Supportare point events e short collective event patterns con stato bounded
   e strategie di correlazione locale configurabili.
5. Propagare metadata MITRE ATT&CK negli alert come classificazione e
   tracciabilita'.
6. Valutare correttezza funzionale, overhead e impatto del primo filtro UID
   kernel-side.

## Quattro contributi principali

### C1 - Target-aware monitoring architecture

Una pipeline modulare kernel-to-userspace adattata a Rocky Linux 8.10 e al suo
kernel RHEL-compatible 4.18, con copertura selezionata di famiglie process e
security.

### C2 - Coherent event contract and userspace processing

Un contratto controllato tra programmi eBPF, trasporto, decoder, registry e
output, rafforzato da controlli di coerenza tra componenti kernel e Go.

### C3 - Declarative local detection pipeline

Una architettura userspace per policy e detector YAML che supporta point events,
short collective patterns, grouping locale per processo, risorsa, cgroup o
chiavi composite, dedup temporale e propagazione dei metadata MITRE ATT&CK.

La strategia `process_tree` non implica un grafo processi persistente completo:
copre lo stesso processo e relazioni parent-child immediate. Le strategie
`resource` e `cgroup` permettono invece correlazioni cross-process locali,
senza introdurre stato distribuito o contesto Kubernetes.

### C4 - Operational and performance-oriented controls

Separazione tra log diagnostici, eventi e alert; packaging containerizzato;
profili di benchmark ripetibili; primo filtro UID kernel-side volto a ridurre il
traffico verso userspace.

Il filtro UID e i benchmark recenti restano sperimentali finche' le modifiche
non sono consolidate e le misure definitive non sono completate.

## Posizionamento della candidate novelty

### Formulazione ammessa

Il contributo distintivo piu' difendibile e' lo studio progettuale ed empirico
di una pipeline eBPF target-aware per un kernel enterprise meno recente,
combinata con una detection locale dichiarativa, bounded e correlabile tramite
identita' semantiche configurabili.

Una formulazione utilizzabile e':

> The distinctive aspect of the work is the design and evaluation of a
> lightweight, bounded and locally correlated detection pipeline integrated
> with an eBPF monitoring agent adapted to the constraints of Rocky Linux 8's
> 4.18-based enterprise kernel.

La frase deve restare prudente finche' Capitolo 2 e Capitolo 5 non forniscono
confronto e risultati sufficienti.

### Elementi che non sono novelty autonome

- uso di eBPF per runtime security;
- policy o detector YAML;
- detection multi-evento in generale;
- metadata MITRE ATT&CK negli alert;
- uso di perf buffer, CO-RE o libbpfgo;
- packaging Docker;
- separazione node agent / backend centralizzato;
- kernel-side filtering in generale.

Questi elementi restano contributi implementativi o scelte architetturali.

## Scope approvato

### In scope

- monitoraggio runtime host/node-level;
- process lifecycle, execution e security-relevant kernel activity;
- normalizzazione degli eventi in userspace;
- policy dichiarative;
- detector rule-based point e collective locali;
- classificazione degli alert tramite MITRE ATT&CK;
- valutazione funzionale e prestazionale sul target Rocky Linux.

### Fuori scope

- enforcement e blocco delle operazioni;
- correlazione distribuita tra nodi;
- cluster-wide contextual anomaly detection;
- compatibilita' universale tra kernel e architetture;
- equivalenza funzionale con Tracee;
- garanzie production-grade;
- affermazione preventiva del target CPU come risultato raggiunto.

### Networking

Il networking non viene presentato tra i contributi principali dell'autore. Il
Capitolo 1 puo' indicarlo come capacita' adiacente integrata nel progetto e
sviluppata in collaborazione. La sua inclusione nella valutazione richiede una
decisione esplicita di attribuzione con azienda e relatore.

### Deployment

La containerizzazione e' implementata e puo' essere descritta nel capitolo
tecnico come scelta di packaging. Kubernetes e' soltanto una possibile
destinazione futura gestita dall'azienda: non va presentato nel Capitolo 1 come
parte dell'architettura, risultato sperimentale o contributo dell'autore.
L'eventuale riferimento deve essere confinato agli sviluppi futuri.

## MITRE ATT&CK

Il tool valida la forma dei metadata tactic/technique dichiarati nei detector
e li propaga negli alert. Questo offre classificazione e tracciabilita'.

Non affermare che:

- il mapping e' stato validato semanticamente rispetto all'intero catalogo;
- il tool offre copertura MITRE completa;
- MITRE guida gia' automaticamente il runtime;
- l'integrazione MITRE e' una novelty autonoma.

La coverage analysis appartiene al Capitolo 5.

## Performance

Il Capitolo 1 puo' dichiarare:

> The project adopts a steady-state userspace CPU target below 5% of one core
> under a benchmark profile that must be defined explicitly in the evaluation.

Non puo' dichiarare che l'obiettivo e' gia' raggiunto. Le misure esistenti sono
dipendenti da workload, eventi, detector, warm-up e filtro UID. I risultati
numerici restano nel Capitolo 5.

## Claims policy

### Formulazioni ammesse

- `research prototype`;
- `implemented kernel-to-userspace pipeline`;
- `selected process and security event coverage`;
- `rule-based point and short collective event patterns`;
- `MITRE ATT&CK metadata propagation`;
- `experimental kernel-side UID pre-filter`;
- `container packaging for the implemented runtime`;
- `Tracee-inspired architectural patterns adapted to the target environment`.

### Formulazioni vietate o da qualificare

- complete visibility;
- complete pipeline, se inteso come prodotto completo;
- production-ready;
- universally portable;
- negligible or proven low overhead;
- guaranteed security through the verifier;
- novel MITRE integration;
- novel collective anomaly detection in generale;
- persistent process-tree tracking;
- validated Kubernetes deployment;
- new general-purpose eBPF runtime-security architecture.

## Piano citazioni per il Capitolo 1

### Context and Motivation

- Gbadamosi et al. e Linux eBPF Userspace API per eBPF ad alto livello;
- Her et al. per il contesto degli eBPF security tools cloud-native;
- He et al. per i rischi del kernel condiviso nei container.

### Problem Statement

- Chandola et al. per point, contextual e collective anomalies;
- MITRE ATT&CK Design and Philosophy per tactic e technique;
- evidenze del repository per target, scope locale e target prestazionale.

## Fonti validate ma da trattare con cautela

- Il paper `The eBPF Runtime in the Linux Kernel` e' un preprint recente.
- La documentazione dei tool e' fonte primaria per le loro feature, non prova
  indipendente di efficacia.
- Le pagine `dev` di Tracee devono essere sostituite, quando possibile, da una
  release o commit fissato per il confronto finale.
- Il paper IEEE Access 2025 e' utile per il confronto generale, ma non deve
  sostituire le fonti ufficiali per feature correnti.

## Decisioni non bloccanti ancora aperte

1. Confermare con relatore e azienda se nominare esplicitamente l'azienda nel
   corpo della tesi.
2. Formalizzare l'attribuzione della componente networking.
3. Definire il profilo benchmark rappresentativo prima del Capitolo 5.
4. Fissare versioni o commit di Tracee, Falco e Tetragon nel Capitolo 2.
5. Confermare limiti di pagine e requisiti formali.

Questi punti non bloccano una prima stesura prudente del Capitolo 1, purche' lo
scrittore usi le formulazioni conservative indicate qui.

## Istruzioni finali per lo scrittore

- Riscrivere il capitolo attorno al problema, non correggere soltanto frasi
  isolate del draft corrente.
- Presentare l'obiettivo generale in forma narrativa e chiudere con una lista
  dei Capitoli 2-6 e del loro focus.
- Mantenere la spiegazione tecnica di eBPF ad alto livello.
- Non anticipare dettagli dei Capitoli 3-5.
- Non modificare l'abstract.
- Non usare il nome interno `Vesuvius`: il tool non ha un nome ufficiale nella
  tesi.
- Usare soltanto le tre `section` principali del Capitolo 1, senza
  `subsection`.
- Segnalare ogni citazione o decisione non risolta.
