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

## Tipi di anomalie supportabili

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

### Contextual anomaly

Evento normale che diventa sospetto nel contesto corretto.

Esempio:

```text
open /etc/passwd
```

puo' essere normale. Diventa piu' interessante se:

- arriva da un processo insolito;
- arriva da un utente insolito;
- arriva dopo un cambio di credenziali;
- arriva da un namespace/container inatteso.

Prima versione implementabile:

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

### Collective anomaly

Gruppo di eventi che insieme descrive un comportamento.

Esempio:

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
- `process_tree`;
- `container_id`, se disponibile in futuro.

## Output degli alert correlati

Un alert correlato deve contenere:

- nome detector;
- severita' o score, se presente;
- chiave di correlazione;
- durata della sequenza;
- lista degli eventi che hanno completato il match;
- motivazione leggibile.

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

## Regole operative

- Gli eventi singoli rilevanti devono continuare a essere emessi subito.
- Gli alert correlati devono avere un output separato.
- Le sequenze devono restare corte.
- La finestra temporale deve rispettare il guardrail attuale del contratto
  detector: default `2s`, massimo `5s`.
- I detector devono essere caricabili da file.
- La prima versione deve essere userspace-only.
- I detector dovrebbero dichiarare metadati MITRE ATT&CK quando la detection ha
  una tecnica o tattica chiaramente associabile.
