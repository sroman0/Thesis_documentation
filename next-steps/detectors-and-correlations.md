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
`process_tree` e' piu' stretta della vecchia chiave demo `global`: correla
eventi dello stesso processo o di una relazione parent-child usando solo il
contesto locale dell'evento.

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

`group_key` puo' essere:

- `pid`;
- `tid`;
- `uid`;
- `comm`;
- `host_pid`;
- `host_tid`;
- `global`, utile solo per detector demo o segnali ravvicinati dove la stessa
  catena puo' attraversare PID diversi;
- `process_tree`, implementato come correlazione locale tra processo corrente e
  parent tramite `host_pid`, `host_ppid`, `start_time` e `parent_start_time`;
- `container_id`, se disponibile in futuro.

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

Detector collective correnti:

| Detector | Sequenza | Scopo |
| --- | --- | --- |
| `privilege-exec-chain` | `security_task_fix_setuid` -> `sched_process_exec` | Cambio privilegi seguito da exec root. |
| `privilege-sensitive-file-chain` | `security_task_fix_setuid` -> `security_file_open` | Cambio privilegi seguito da accesso root a file sensibili, con esclusione degli helper legittimi di `sudo`. |
| `memfd-exec-chain` | `memfd_create` -> `sched_process_exec` | Creazione file anonimo in memoria seguita da exec da path memfd. |
| `kernel-module-kprobe-chain` | `do_init_module` -> `register_kprobe` | Modulo kernel inizializzato seguito da registrazione kprobe dinamica. |

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
