# Piano di implementazione

## Fase 1: struttura config

Stato: completata a livello di modello di configurazione.

La configurazione centrale del tool e' stata estesa in
`demo_project/pkg/config/config.go` con:

```text
Policies.Paths
Detectors.Paths
Detectors.Enabled
Alerts.Enabled
Alerts.Format
```

Questi campi preparano il terreno per policy dichiarative, detector YAML e
output separato degli alert. Le flag CLI sono state collegate in
`demo_project/pkg/cmd/cobra/cobra.go`.

Il runner applicativo in `demo_project/pkg/cmd/project.go` prepara ora un
blocco `RuntimeExtensions` con path policy, path detector, stato detector e
configurazione alert prima di avviare il runtime eBPF. Questo e' il punto di
aggancio per i futuri package `pkg/policy` e `pkg/detectors`.

Flag disponibili:

```bash
--policy policy.yaml
--detectors detectors/
--alerts
--alerts-output json
```

Nota: `--detectors` abilita automaticamente il motore dei detector quando
riceve almeno un file o una directory. La policy filtra il flusso, i detector
generano alert.

## Fase 1.1: modello policy interno

Stato: completata.

E' stato creato `demo_project/pkg/policy/policy.go` con il modello normalizzato
che il runtime usera' dopo il caricamento delle policy dichiarative:

```text
Policy
  -> Mode
  -> Intent
  -> Scope
  -> []Rule

Rule
  -> EventSelector
  -> comm filter
  -> uid filter
```

Il modello non dipende ancora da YAML o JSON. Questa separazione e' voluta: il
loader potra' leggere file in un formato esterno, ma il runtime lavorera' sempre
su strutture Go gia' validate e normalizzate.

Semantiche fissate:

- una policy senza nome non e' valida;
- una policy senza regole non e' valida;
- `Mode` distingue monitoraggio, detection e soppressione del rumore;
- `Intent` descrive lo scopo operativo della policy;
- se `Mode` non e' specificato, il default e' `monitor`;
- se `Intent` non e' specificato, il default e' `threat_hunting`;
- lo scope della policy viene controllato prima delle singole regole;
- `EventSelector.Include` vuoto significa "tutti gli eventi";
- `EventSelector.Exclude` ha priorita' su `Include`.

Elemento di novelty: la policy non e' solo un filtro tecnico. Il modello
separa la selezione degli eventi dalla motivazione operativa, cosi' il tool puo'
distinguere policy di osservazione, policy per detection, policy di hardening,
policy di compliance e policy per riduzione del rumore.

## Fase 1.2: loader policy YAML

Stato: completata.

E' stato creato `demo_project/pkg/policy/loader.go`. Il loader trasforma i path
passati con `--policy` in una lista di `policy.Policy` normalizzate.

Comportamento attuale:

- accetta file singoli `.yaml` e `.yml`;
- accetta directory e carica i file `.yaml`/`.yml` non ricorsivamente;
- ignora file con estensioni diverse quando scansiona una directory;
- rifiuta un file singolo con estensione non supportata;
- ordina i file di directory per avere un caricamento deterministico;
- valida ogni policy dopo il parsing;
- supporta gia' `mode` e `intent`, quindi la novelty e' disponibile anche nel
  formato dichiarativo.

Esempio iniziale:

```yaml
name: suspicious-root-exec
description: Root execution policy
mode: detect
intent: hardening
scope:
  uids: [0]
rules:
  - name: exec-events
    events:
      include: [execve, execveat]
```

Il runner applicativo carica gia' le policy in `RuntimeExtensions.Policies`
prima di avviare eBPF. Lo step successivo usera' queste policy caricate per
costruire un manager runtime.

## Fase 1.3: manager policy runtime

Stato: completata.

E' stato creato `demo_project/pkg/policy/manager.go`. Il manager prende le
policy gia' caricate e validate e le rende operative per il runtime.

Componenti principali:

```text
Manager
  -> []Policy

MatchResult
  -> []Policy matchate
  -> Suppressed
  -> Detect
  -> Monitor
```

Funzioni disponibili:

- `NewManager(policies)` valida e conserva le policy attive;
- `Match(input)` restituisce quali policy matchano un evento;
- `IsEventSelected(input)` indica se l'evento deve proseguire nella pipeline;
- `SelectedEvents()` restituisce gli eventi esplicitamente richiesti dalle
  policy non-suppress.

Regola operativa importante:

```text
suppress vince su monitor/detect
```

Questa scelta rende utilizzabile la novelty `PolicyMode`: una policy puo'
osservare, alimentare detection oppure ridurre rumore. Lo step successivo
iniziera' a definire il contratto dei detector, che riceveranno solo eventi gia'
decodificati e filtrabili dal manager.

## Fase 2: contratto detector

Stato: completata a livello di contratto runtime iniziale.

E' stato creato `demo_project/pkg/detectors/detector.go`. Questo file definisce
il contratto che dovranno rispettare sia detector Go nativi sia detector YAML
futuri.

Interfaccia base:

```go
type Detector interface {
    Definition() Definition
    Init(context.Context) error
    OnEvent(context.Context, Event) ([]Alert, error)
    Flush(context.Context, time.Time) ([]Alert, error)
}
```

Scelte principali:

- `Event` e' un alias di `bufferdecoder.Event`, quindi i detector ricevono gia'
  eventi decodificati;
- `Definition` contiene metadata runtime (`ID`, `Name`, `Description`) e
  dichiara anche gli eventi consumati tramite `Consumes`;
- `Stateful` indica che il detector mantiene contesto tra eventi diversi;
- `Window` limita la finestra temporale di correlazione;
- `Alert` e' un risultato preliminare prodotto dai detector;
- il formato finale degli alert verra' gestito piu' avanti dall'output layer.

La prima novelty inserita nel contratto e' il supporto a detector stateful e
context-aware. In pratica il detector non e' obbligato a reagire solo al singolo
evento: puo' trattenere un piccolo stato per correlare sequenze ravvicinate.
Per non violare il vincolo prestazionale della VM, la finestra temporale resta
corta:

```go
DefaultStateWindow = 2 * time.Second
MaxStateWindow     = 5 * time.Second
```

`Flush` permette al futuro engine di chiedere al detector di chiudere finestre
scadute e produrre eventuali alert pendenti. Se i benchmark mostrano regressioni
sopra il target operativo, questa feature andra' mitigata riducendo la finestra,
limitando il numero di detector stateful attivi o disabilitando temporaneamente
la correlazione.

Questa separazione evita di legare subito i detector al formato YAML o al
printer. Il detector decide se un evento o una sequenza e' rilevante; l'output
decidera' come mostrarlo.

## Fase 2.1: definizione estesa detector

Stato: prossimo step.

La definizione estesa va inserita in `demo_project/pkg/detectors/definition.go`
per evitare che `detector.go` diventi troppo ampio. `detector.go` deve restare
il contratto runtime, mentre `definition.go` deve descrivere cosa un detector
consuma, produce e rappresenta.

Strutture iniziali consigliate:

```go
type EventRequirement struct {
    EventName string
    Optional  bool
    Reason    string
}

type DetectorOutput struct {
    Title       string
    Description string
    Severity    Severity
}

type ThreatMetadata struct {
    Framework      string
    Version        string
    Tactics        []AttackTactic
    Techniques     []AttackTechnique
    DataSources    []string
    DataComponents []string
}

type AttackTactic struct {
    ID   string
    Name string
}

type AttackTechnique struct {
    ID           string
    Name         string
    SubTechnique string
}
```

Validazioni minime:

- ID detector obbligatorio;
- nome detector obbligatorio;
- severita' valida;
- eventi richiesti non vuoti;
- finestra stateful entro il limite massimo `5s`;
- tactic ID in formato `TA0000`, se presente;
- technique ID in formato `T0000` o `T0000.000`, se presente.

Nota MITRE: i metadati ATT&CK devono stare principalmente nella definizione del
detector, perche' il detector rappresenta la logica di detection. Le policy
potranno poi selezionare detector anche per tactic/technique.

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
window: 2s
group_by:
  - process_tree
threat:
  framework: MITRE ATT&CK Enterprise
  tactics:
    - TA0004
  techniques:
    - T1548.001
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
window: 5s
group_by:
  - process_tree
threat:
  framework: MITRE ATT&CK Enterprise
  tactics:
    - TA0002
    - TA0005
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
window: 2s
group_by:
  - uid
threat:
  framework: MITRE ATT&CK Enterprise
  tactics:
    - TA0005
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
