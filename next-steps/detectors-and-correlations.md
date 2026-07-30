# Detector YAML e alert correlati

## Problema

Le policy classiche catturano bene anomalie puntuali:

```text
evento singolo + filtro
```

Esempi:

- `open` su `/etc/shadow`;
- `chmod` con bit SUID;
- `security_task_kill` verso un processo sensibile;
- `security_task_fix_setuid` con `new_euid=0`.

Molti comportamenti interessanti, pero', non sono evidenti da un singolo evento.
Diventano rilevanti quando piu' eventi avvengono nello stesso contesto.

## Obiettivo

Introdurre un motore di detector dichiarativi:

```text
decoded event
  -> detector engine
  -> sequence state
  -> correlated alert
```

Il detector engine deve lavorare in userspace, almeno nella prima versione.

La pipeline applica anche una protezione anti-rumore: il detector engine
mantiene una piccola cache temporale degli alert appena emessi e sopprime gli
alert identici prodotti entro pochi secondi. Questo e' utile soprattutto per
point anomalies come `security_file_open`, dove un processo puo' aprire lo
stesso file molte volte in rapida sequenza.

Il dispatcher del detector engine usa un indice per evento:

```text
event_name -> detector interessati
```

Questo evita il modello costoso in cui ogni evento viene passato a tutti i
detector. Le metriche `DetectorMatched`, `DetectorInvoked` e
`DetectorSkipped` servono a verificare il comportamento durante benchmark e
debug. In particolare, `DetectorSkipped` misura quante valutazioni sono state
evitate grazie all'indice.

## Selezione dei detector tramite policy

Quando `--detectors` indica una directory, una policy può restringere il
catalogo prima della costruzione del detector engine:

```yaml
detectors:
  include:
    severities: [high, critical]
    tactics: [TA0004]
    techniques: [T1548]
    tags: [collective]
    stateful: true
  exclude:
    ids: [known-noisy-detector]
```

Le liste della stessa proprietà usano OR. Proprietà differenti usano AND:
nell'esempio un detector deve avere severity high o critical, essere mappato a
`TA0004` e `T1548`, avere il tag `collective` ed essere stateful. `exclude`
prevale all'interno della policy.

Il filtro viene applicato una volta all'avvio:

```text
catalogo YAML
  -> definizioni validate
  -> selezione metadata della policy
  -> costruzione dei soli detector scelti
  -> registry e dispatcher
```

Non è quindi un controllo ripetuto per ogni evento. Se nessuna policy attiva
contiene `detectors`, resta il comportamento storico e viene caricato l'intero
catalogo indicato dalla CLI. Quando almeno una policy usa il selettore, le
policy senza tale sezione continuano a filtrare eventi ma non aggiungono
detector alla selezione. Ogni detector scelto deve avere tutti i propri eventi
inclusi nelle regole della stessa policy; un conflitto interrompe il bootstrap
con un errore.

Se la policy contiene `detectors` ma la CLI non fornisce `--detectors`, il tool
parte comunque. Le regole evento della policy restano attive, mentre la
selezione detector viene saltata e il runtime emette un warning:

```text
detector policy selection skipped
reason="no detector catalog was configured with --detectors"
```

Un ID detector viene considerato esistente solo se compare nelle definizioni
YAML caricate dai path passati a `--detectors`. Il bootstrap costruisce il
catalogo dagli `id` dei file analizzati e valida contro quel catalogo gli ID
usati in `include` ed `exclude`. Non viene consultato un registry globale:
un ID può quindi essere valido in una directory e mancare in un catalogo più
ristretto.

Questa feature usa metadata MITRE dichiarati localmente nei detector. Non è
ancora un report di coverage e non verifica la semantica degli ID contro un
dataset ATT&CK scaricato.

Il dedup e' intenzionalmente locale e corto:

```text
detector_id + titolo + severita' + evento sorgente + pid + uid + comm + args
```

Se cambia processo, path, detector o contenuto degli argomenti, l'alert resta
visibile. Se invece lo stesso processo ripete lo stesso comportamento nella
finestra di default, il tool stampa un solo alert.

## Tipi di anomalie supportabili nel tool

### Point anomaly

Evento singolo rilevante da solo.

Esempio:

```yaml
type: detector
name: direct-shadow-access
rules:
  - event: open
    filters:
      - data.pathname=/etc/shadow
```

### Collective anomaly locale

Gruppo breve di eventi che insieme descrive un comportamento.

Primo caso implementato nel tool:

```yaml
id: privilege-exec-chain
stateful: true
window: 5s
group_by:
  - process_tree
steps:
  - event: security_task_fix_setuid
    conditions:
      - field: args.new_euid
        operator: eq
        value: "0"
  - event: sched_process_exec
    conditions:
      - field: uid
        operator: eq
        value: "0"
```

Questo detector produce un alert quando una transizione a effective UID root e'
seguita da una esecuzione come UID 0 entro una finestra breve. La chiave
`process_tree` correla eventi dello stesso processo o di una relazione
parent-child immediata usando solo il contesto locale dell'evento.

Il parser dei detector valida ora i campi `group_by` durante il caricamento.
Questo evita configurazioni ambigue: se un detector usa una chiave non ancora
supportata, il tool fallisce in modo esplicito
invece di caricare una regola che non produrra' mai match.

Esempio target futuro:

```yaml
type: detector
name: root-sensitive-file-after-setuid
window: 2s
group_by:
  - process_tree
steps:
  - event: security_task_fix_setuid
    filters:
      - data.new_euid=0
  - event: open
    filters:
      - data.pathname=/etc/*
```

Altro esempio:

```yaml
type: detector
name: drop-and-execute
window: 5s
group_by:
  - process_tree
steps:
  - event: open
    filters:
      - data.pathname=/tmp/*
  - event: chmod
    filters:
      - data.mode_contains=executable
  - event: execve
    filters:
      - data.pathname=/tmp/*
```

### Contextual anomaly fuori scope locale

Le anomalie contestuali richiedono contesto piu' ampio: baseline per host,
utente, pod, namespace Kubernetes, servizio o cluster. Il tool girera' come
agent in un pod e non avra' una vista globale sufficiente per decidere da solo
se un evento e' anomalo rispetto all'intero ambiente.

Per questo motivo la prima implementazione locale non deve provare a produrre
contextual anomalies complete. Il suo compito e':

- emettere point anomalies quando un evento singolo e' chiaramente rilevante;
- emettere collective anomalies quando una breve sequenza locale e' sospetta;
- lasciare le contextual anomalies al sistema centralizzato che ricevera' gli
  alert e avra' visibilita' di cluster.

## Stato runtime

Per ogni detector serve uno stato temporaneo.

Struttura concettuale:

```text
state[detector][group_key]
  current_step
  first_seen
  last_seen
  matched_events[]
```

`group_key` viene costruita da una o piu' strategie:

- `process`: identita' stabile `host_pid:start_time`;
- `process_tree`: processo corrente o parent immediato;
- `resource`: oggetto file `dev:inode`;
- `cgroup`: identificatore cgroup locale.

La lista e' componibile. Con `group_by: [cgroup, resource]` devono coincidere
sia cgroup sia file. `comm`, pathname e UID non sono usati come identita'
stabili. La vecchia sintassi `pid`/`host_pid` viene convertita esplicitamente a
`process`.

La prima versione usa solo group key locali. Non dipende da informazioni
globali di cluster.

`process_tree` non ricostruisce l'intero albero dei processi in memoria. Per
mantenere basso l'overhead, calcola chiavi brevi a partire dal contesto gia'
presente nell'evento:

```text
self   = host_pid:task_start_time
parent = host_ppid:parent_start_time
```

Quando un detector stateful riceve il primo evento della sequenza, salva lo
stato sulla chiave `self`. Quando riceve gli eventi successivi, prova sia
`self` sia `parent`. In questo modo la sequenza puo' completarsi nello stesso
processo oppure tra parent e child, senza usare una chiave `global`.

### Pulizia periodica dello stato

La prima implementazione attraversava l'intera mappa delle sequenze a ogni
evento stateful. Il comportamento era corretto, ma il costo cresceva con il
numero di correlazioni parziali aperte.

Il runtime ora calcola al caricamento una frequenza di pruning:

```text
min(window / 2, 1s)
```

Tra due scadenze controlla direttamente solo la sequenza individuata dalla
chiave dell'evento. Alla scadenza esegue la pulizia globale; `Flush()` continua
a forzarla anche in assenza di nuovi eventi. In questo modo le finestre brevi
restano rispettate senza pagare una scansione completa su ogni record.

I componenti `group_by` vengono compilati durante `NewDetector()`, quindi la
hot path non reinterpreta i nomi YAML. Lo stato e' limitato temporalmente e a
4096 sequenze incomplete per detector.

Detector collective correnti:

| Detector | Sequenza | Scopo |
| --- | --- | --- |
| `privilege-exec-chain` | `security_task_fix_setuid` -> `sched_process_exec` | Cambio privilegi seguito da exec root. |
| `privilege-sensitive-file-chain` | `security_task_fix_setuid` -> `security_file_open` | Cambio privilegi seguito da accesso root a file sensibili, con esclusione degli helper legittimi di `sudo`. |
| `memfd-exec-chain` | `memfd_create` -> `sched_process_exec` | Creazione file anonimo in memoria seguita da exec da path memfd. |
| `kernel-module-kprobe-chain` | `do_init_module` -> `register_kprobe` | Modulo kernel inizializzato seguito da registrazione kprobe dinamica. |
| `temp-script-write-exec` | `security_file_permission` -> `security_bprm_check` | Script `/tmp/*.sh` scritto e poi preparato per esecuzione sullo stesso `dev:inode`. |

Nota sui detector demo: alcune regole escludono helper legittimi e molto
rumorosi, in particolare `sudo` e `unix_chkpwd`. Questa scelta riduce falsi
positivi durante demo e test locali; non impedisce di creare in futuro detector
piu' conservativi che includano anche questi processi.

## Output degli alert correlati

Un alert correlato deve contenere:

- nome detector;
- severita' o score, se presente;
- chiave di correlazione;
- durata della sequenza;
- lista degli eventi che hanno completato il match;
- motivazione leggibile.
- mapping MITRE ATT&CK quando dichiarato dal detector.

Esempio:

```json
{
  "type": "correlated_alert",
  "name": "root-sensitive-file-after-setuid",
  "group_by": "process_tree",
  "duration_ms": 1260,
  "events": [
    {"event": "security_task_fix_setuid", "new_euid": 0},
    {"event": "open", "pathname": "/etc/shadow"}
  ]
}
```

Stato attuale: i metadata MITRE dichiarati nel detector YAML vengono propagati
fino all'alert runtime. L'output table espone una sintesi compatta:

```text
mitre=TA0004|T1548|TA0007|T1083
```

L'output JSON espone invece un blocco strutturato `threat` con framework,
versione, tattiche, tecniche, data source e data component. Questo mantiene il
table leggibile durante demo e rende il JSON adatto a integrazioni SOC/SIEM.

## Regole operative

- Gli eventi singoli rilevanti devono continuare a essere emessi subito.
- Gli alert correlati devono avere un output separato.
- Le sequenze devono restare corte.
- La finestra temporale deve rispettare il guardrail attuale del contratto
  detector: default `2s`, massimo `5s`.
- I detector devono essere caricabili da file.
- La prima versione deve essere userspace-only.
- La prima versione locale deve evitare contextual anomalies basate su baseline
  globali o contesto Kubernetes completo.
- I detector dovrebbero dichiarare metadati MITRE ATT&CK quando la detection ha
  una tecnica o tattica chiaramente associabile.

La strategia `user_session` resta differita: UID non e' una sessione e il
context normalizzato non contiene ancora un audit/session ID. Dettagli, sintassi
ed esempi sono nella
[documentazione della correlazione locale](../implementation/correlation.md).
