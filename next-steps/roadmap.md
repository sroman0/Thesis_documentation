# Roadmap tecnica

## Obiettivo

Il prossimo salto del tool dovrebbe essere il passaggio da:

```text
event monitor
```

a:

```text
policy and behavior monitor
```

Questo significa continuare a stampare eventi singoli quando sono rilevanti, ma
aggiungere anche un livello capace di generare alert correlati.

## Priorita' 1: policy YAML userspace-only

Prima di portare filtri complessi nel kernel, conviene implementare una prima
versione userspace-only.

Funzionalita':

- flag `--policy`;
- parsing di un formato YAML semplice;
- validazione degli eventi rispetto al registry esistente;
- filtro per `scope`;
- filtro per argomenti evento gia' decodificati;
- incompatibilita' esplicita tra `--policy` e flag manuali ambigui come
  `--events`, `--drop-events`, `--comms`.

Esempio:

```yaml
type: policy
name: sensitive-file-access
description: monitor selected file accesses
scope:
  - comm=cat
rules:
  - event: open
    filters:
      - data.pathname=/etc/*
```

Motivo: questa fase consente di testare il modello policy senza introdurre
subito complessita' eBPF o problemi di verifier.

## Priorita' 2: detector YAML

Le policy selezionano e filtrano eventi. I detector devono invece generare alert.

Un detector dovrebbe poter essere aggiunto come file YAML, senza modificare codice
Go e senza ricostruire l'immagine del tool.

Esempio:

```yaml
type: detector
name: privilege-escalation-chain
description: setuid followed by sensitive file access
window: 2s
group_by:
  - process_tree
threat:
  framework: MITRE ATT&CK Enterprise
  tactics:
    - TA0004
  techniques:
    - T1548
steps:
  - event: security_task_fix_setuid
    filters:
      - data.new_euid=0
  - event: open
    filters:
      - data.pathname=/etc/shadow
```

Motivo: questo rende il tool estendibile da un analista senza rebuild.

## Priorita' 3: output separato per eventi e alert

Il tool deve distinguere:

- eventi singoli;
- alert correlati.

Formato concettuale:

```json
{"type":"event","event":"open","comm":"cat"}
```

```json
{
  "type": "correlated_alert",
  "name": "privilege-escalation-chain",
  "events": ["security_task_fix_setuid", "open"]
}
```

Motivo: gli alert correlati non devono sporcare l'output operativo degli eventi.

## Priorita' 4: correlazioni corte e leggibili

Le sequenze devono essere brevi. Una catena troppo lunga e' fragile, difficile da
spiegare e rischia di generare alert tardivi.

Regola pratica:

```text
2-4 step per detector
finestra temporale breve: default 2s, massimo 5s nella prima versione
group_by esplicito
output con eventi che hanno causato il match
```

Quando possibile, ogni detector dovrebbe indicare anche i metadati MITRE
ATT&CK: tattica, tecnica, eventuale sub-tecnica e data source/data component.

## Priorita' 5: kernel-side filtering minimo

Solo dopo aver stabilizzato policy e detector in userspace conviene riportare
parte del filtering nel kernel.

Prima fase kernel-side:

- evento abilitato;
- `comm`;
- `uid`;
- `pid`.

Seconda fase:

- `data.pathname` per eventi file-related;
- eventuali filtri su `target_comm` per signal/process security.

Motivo: ridurre CPU e pressione su perf buffer senza anticipare complessita'.
