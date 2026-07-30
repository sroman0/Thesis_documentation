# Dossier Capitolo 3 - Requirements and System Design

## Obiettivo

Tradurre il problema e i vincoli introdotti nei primi due capitoli in requisiti
e decisioni architetturali verificabili. Il capitolo deve spiegare:

1. quali responsabilita' ha il sistema;
2. quali vincoli derivano dal kernel target e dall'uso operativo previsto;
3. come sono separati raccolta kernel, normalizzazione, selezione, detection e
   output;
4. perche' sono state adottate queste separazioni.

Il Capitolo 3 descrive il design, non il codice. Nomi di funzioni, struct, mappe,
file Go, sezioni ELF, flag CLI e procedure di test appartengono al Capitolo 4 o
alle appendici.

## Struttura canonica

### 3.1 Design Goals and Constraints

#### 3.1.1 Target Environment and Compatibility Assumptions

Definisce Rocky Linux 8.10, il kernel enterprise
`4.18.0-553.109.1.el8_10.x86_64`, la presenza di backport e la necessita' di
verificare sul target hook, helper, BTF, attachment e comportamento del
verifier. CO-RE riduce la dipendenza dai layout, ma non sostituisce funzionalita'
assenti.

#### 3.1.2 Functional Requirements

Formalizza requisiti tracciabili, tra cui:

- osservare eventi rilevanti per processi e sicurezza host;
- selezionare gli eventi richiesti senza collegare probe non necessarie;
- trasferire e normalizzare record tipizzati;
- applicare policy configurabili;
- valutare detector point e collective;
- conservare negli alert le evidenze del match;
- associare metadati MITRE ATT&CK;
- supportare output eventi, output alert e logging operativo distinti.

#### 3.1.3 Non-Functional and Operational Requirements

Copre compatibilita', verificabilita', stato limitato, controllo del rumore,
chiarezza dell'output, gestione degli errori e obiettivo prestazionale. Il 5% di
un core resta un obiettivo da valutare per profili definiti, non una proprieta'
universale gia' dimostrata.

### 3.2 Overall System Architecture

#### 3.2.1 Kernel-Space and User-Space Responsibilities

Spiega il confine fondamentale:

- kernel space: osservazione, estrazione minima, selezione anticipata
  disponibile e serializzazione;
- user space in Go: lifecycle eBPF, decoding, normalizzazione, registry eventi,
  policy, detector, correlazione, alert, output e logging.

La scelta mantiene i programmi eBPF compatibili con il verifier e consente di
evolvere le regole senza ricompilare ogni probe.

#### 3.2.2 End-to-End Event Lifecycle

Presenta il flusso logico:

```text
kernel hook
  -> eBPF event collection
  -> binary record
  -> perf-buffer transport
  -> Go decoder and event registry
  -> event and policy selection
  -> detector dispatch
  -> raw event output and/or alert output
```

Il diagramma finale deve mostrare anche il percorso separato dei log operativi.

### 3.3 Kernel-Space Collection Design

#### 3.3.1 Hook Selection and Logical Events

Descrive i criteri di scelta degli hook: semantica osservata, necessita' del
valore di ritorno, compatibilita' con il target e costo previsto. Distingue hook
fisico ed evento logico e raggruppa la copertura per dominio, senza elencare
tutti gli hook nel corpo del capitolo.

#### 3.3.2 Early Event Selection and Filtering

Spiega la selezione delle probe prima dell'attach e il filtro UID opzionale
applicato nel kernel. Non deve suggerire un motore completo di policy nel
kernel: le policy e i detector restano principalmente responsabilita' del
runtime Go.

### 3.4 Event Contract and Transport Design

#### 3.4.1 Stable Binary Event Contract

Motiva un header comune e argomenti tipizzati, condivisi tra C e Go. Introduce
la necessita' di mantenere coerenti identificatori, dimensioni e semantica tra
producer e consumer. Il layout da 136 byte e le correzioni specifiche vengono
documentati nel Capitolo 4.

#### 3.4.2 Perf-Buffer Transport and Failure Boundaries

Motiva il perf buffer come trasporto operativo compatibile con il kernel
target. Discute perdita, backpressure, record malformati e osservabilita' degli
errori come confini progettuali. Il BPF ring buffer non e' implementato nel
runtime corrente.

### 3.5 User-Space Runtime Design

#### 3.5.1 Loading, Attachment and Runtime Lifecycle

Descrive configurazione, caricamento dell'oggetto, risoluzione delle probe,
attach selettivo, consumo degli eventi e chiusura ordinata. Evitare walkthrough
di package e funzioni.

#### 3.5.2 Decoding, Normalisation and Event Registration

Spiega come il decoder trasformi record binari in eventi con identita',
contesto e argomenti tipizzati. Il registry rappresenta il contratto userspace
per nomi, ID, schemi e dipendenze degli eventi.

#### 3.5.3 Event Output, Alert Output and Operational Logging

Mantiene distinti:

- telemetria raw degli eventi;
- alert prodotti dai detector;
- log diagnostici strutturati del runtime.

Le modalita' che mostrano soltanto gli alert appartengono al livello di
presentazione e non modificano la logica di raccolta o detection. La tecnologia
di logging e le flag concrete vengono descritte nel Capitolo 4.

### 3.6 Policy and Detection Architecture

#### 3.6.1 Policy-Based Event Scope

Le policy selezionano il dominio degli eventi e possono restringere la
selezione effettiva prima dell'attach quando gli allow rule sono espliciti.
Non selezionano attualmente i detector per ID o metadata MITRE: i detector sono
caricati da percorsi configurati separatamente.

#### 3.6.2 Point and Collective Detectors

I detector point valutano un evento. I detector collective valutano sequenze
ordinate e trattengono evidenze locali fino al completamento o alla scadenza.
Entrambi sono deterministici e rule-based.

#### 3.6.3 Bounded Local Correlation

Spiega a livello architetturale le identita' di processo stabile, relazione
parent-child immediata, identita' stabile di una risorsa file, cgroup locale e
combinazioni composite. Lo stato e' limitato da finestre brevi e da un massimo
per detector. Non esiste un grafo persistente dei processi o una correlazione
cluster-wide. I campi concreti usati per costruire queste identita' appartengono
al Capitolo 4.

#### 3.6.4 Alert Evidence and MITRE ATT&CK Metadata

Gli alert mantengono l'evento o la sequenza che ha prodotto il match. Tattiche e
tecniche ATT&CK forniscono un vocabolario condiviso per classificazione e
integrazione, senza sostituire la validazione funzionale dei detector.

### 3.7 Design Decisions and Scope Boundaries

Raccoglie le decisioni principali e le alternative escluse:

- eBPF per raccolta, Go per elaborazione e detection;
- perf buffer invece del ring buffer nel target corrente;
- registry espliciti invece di attach implicito di ogni programma;
- policy separate dai detector;
- correlazione locale e limitata invece di CEP generale;
- osservazione e detection senza enforcement;
- deployment Kubernetes e correlazione globale demandati a sviluppi esterni o
  futuri.

## Confini Con I Capitoli Adiacenti

Il Capitolo 2 contiene teoria e related work. Il Capitolo 3 presenta requisiti,
componenti e ragioni progettuali. Il Capitolo 4 dimostra come tali decisioni
sono realizzate nel codice. Il Capitolo 5 valuta comportamento, copertura e
prestazioni.

## Evidenze Interne Da Verificare

- `demo_project/pkg/config`;
- `demo_project/pkg/ebpf`;
- `demo_project/pkg/ebpf/probes`;
- `demo_project/pkg/bufferdecoder`;
- `demo_project/pkg/events`;
- `demo_project/pkg/policy`;
- `demo_project/pkg/detectors`;
- `demo_project/pkg/output`;
- `demo_project/pkg/logging`;
- `demo_project/rules`;
- `documentation/implementation/overview.md`;
- `documentation/implementation/hooks.md`;
- `documentation/implementation/output.md`;
- `documentation/implementation/correlation.md`;
- `documentation/report.md`;
- `documentation/debugging/commands.md`.

## Output Attesi Prima Della Scrittura

1. matrice requisiti-evidenze e audit dei vincoli;
2. modello dell'architettura kernel, contratto eventi e trasporto;
3. modello del runtime Go e del lifecycle end-to-end;
4. modello verificato di policy, detector, correlazione e alert;
5. sintesi editoriale unica con diagrammi proposti e confini Chapter 3/4;
6. stesura LaTeX da parte di un solo agente;
7. revisione indipendente tecnica ed editoriale.
