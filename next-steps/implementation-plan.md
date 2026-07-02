# Piano di implementazione

## Fase 1: struttura config

Estendere la configurazione del tool con:

```text
PolicyPath
DetectorPath
EnableCorrelations
```

Flag possibili:

```bash
--policy policy.yaml
--detectors detectors/
--alerts-output json
```

Nota: nella prima versione `--policy` e `--detectors` possono essere separati.
La policy filtra il flusso, i detector generano alert.

## Fase 2: formato detector

Definire un package Go dedicato:

```text
pkg/detectors/
```

Strutture iniziali:

```go
type DetectorFile struct {
    Type        string         `yaml:"type"`
    Name        string         `yaml:"name"`
    Description string         `yaml:"description"`
    Within      string         `yaml:"within"`
    GroupBy     []string       `yaml:"group_by"`
    Steps       []DetectorStep `yaml:"steps"`
}

type DetectorStep struct {
    Event   string   `yaml:"event"`
    Filters []string `yaml:"filters"`
}
```

Validazioni:

- `type` deve essere `detector`;
- `name` obbligatorio;
- `steps` non vuoto;
- ogni evento deve esistere nel registry;
- `within` deve essere parsabile come durata;
- `group_by` deve usare campi supportati.

## Fase 3: engine userspace

Inserire il detector engine dopo il decoder e dopo i filtri base.

Pipeline:

```text
raw event
  -> decoder
  -> event filter / comm filter
  -> policy filter
  -> detector engine
  -> event output
  -> alert output
```

Il detector engine deve:

1. ricevere eventi decodificati;
2. calcolare la `group_key`;
3. controllare se l'evento avanza una sequenza esistente;
4. controllare se l'evento apre una nuova sequenza;
5. eliminare stati scaduti;
6. emettere alert quando una sequenza viene completata.

## Fase 4: output alert

Aggiungere un tipo output separato.

Possibile struttura:

```go
type Alert struct {
    Type        string
    Name        string
    Description string
    GroupKey    string
    StartedAt   uint64
    CompletedAt uint64
    Events      []output.Event
}
```

Il formato `table` puo' stampare una riga compatta:

```text
alert=root-sensitive-file-after-setuid group=pid:1234 events=setuid->open duration=1.2s
```

Il formato `json` deve mantenere tutti gli eventi che hanno composto l'alert.

## Fase 5: detector iniziali

Creare alcuni detector demo:

### Privilege escalation and sensitive file access

```yaml
type: detector
name: root-sensitive-file-after-setuid
within: 10s
group_by:
  - process_tree
steps:
  - event: security_task_fix_setuid
    filters:
      - data.new_euid=0
  - event: open
    filters:
      - data.pathname=/etc/shadow
```

### Drop and execute

```yaml
type: detector
name: drop-and-execute
within: 30s
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

### Suspicious signal

```yaml
type: detector
name: suspicious-signal-to-service
within: 5s
group_by:
  - uid
steps:
  - event: security_task_kill
    filters:
      - data.target_comm=sshd
```

## Fase 6: kernel-side filtering

Solo dopo la stabilizzazione del detector engine:

- portare `event enabled`, `comm`, `uid`, `pid` nel kernel;
- aggiungere filtri su `pathname` per eventi file-related;
- misurare riduzione eventi e CPU.

Metriche da raccogliere:

```text
events_seen
events_dropped_userspace
events_dropped_kernel
alerts_emitted
cpu_percent
rss
```

