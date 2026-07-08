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

La prima novelty inserita nel contratto e' il supporto a detector stateful per
collective anomalies locali. In pratica il detector non e' obbligato a reagire
solo al singolo evento: puo' trattenere un piccolo stato per correlare sequenze
ravvicinate. Non si tratta pero' di contextual anomaly completa: il contesto
globale di cluster restera' responsabilita' del sistema centralizzato. Per non
violare il vincolo prestazionale della VM, la finestra temporale resta corta:

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

Stato: completata.

La definizione estesa va inserita in `demo_project/pkg/detectors/definition.go`
per evitare che `detector.go` diventi troppo ampio. `detector.go` deve restare
il contratto runtime, mentre `definition.go` deve descrivere cosa un detector
consuma, produce e rappresenta.

Strutture implementate:

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

Validazioni implementate:

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

Helper disponibili:

- `Definition.Validate()`;
- `Definition.EffectiveWindow()`;
- `Definition.ValidatePerformanceBounds()`;
- `Definition.EventNames()`;
- `Definition.RequiredEventNames()`;
- `DetectorOutput.NormalizedSeverity()`;
- `ThreatMetadata.IsMapped()`.

La scelta importante e' che i detector senza metadata MITRE restano validi, ma
sono distinguibili come `unmapped`. Questo permette di procedere senza bloccare
lo sviluppo mentre la coverage ATT&CK viene raffinata.

## Fase 2.2: schema YAML dei detector

Stato: completata.

Ora che la definizione interna e' stabile, e' stato creato lo schema esterno in:

```text
demo_project/pkg/detectors/yaml/schema.go
```

Lo schema YAML rappresenta il formato scritto dall'utente, separato dalla
definizione runtime interna. Questa separazione evita di mescolare parsing,
validazione e logica di detection.

Campi principali rappresentati:

- identificativi e descrizione del detector;
- eventi richiesti;
- eventi consumati;
- stato e finestra temporale;
- campi di correlazione (`group_by`);
- condizioni e sequenze future;
- output dell'alert;
- metadati MITRE ATT&CK;
- tag descrittivi.

Strutture implementate:

```text
File
EventRequirement
Condition
Step
Output
Threat
AttackTactic
AttackTechnique
```

E' stato aggiunto un test di unmarshalling in
`demo_project/pkg/detectors/yaml/schema_test.go`, con un esempio completo di
detector stateful mappato a MITRE ATT&CK.

La regola e' non mettere logica runtime nello schema. Lo schema deve solo
descrivere il file utente; il parser dello step successivo lo convertira' nella
`detectors.Definition` interna.

## Fase 2.3: parser YAML dei detector

Stato: completata.

E' stato implementato:

```text
demo_project/pkg/detectors/yaml/parser.go
```

API principali:

```go
ParseBytes(data []byte, opts ...ParseOption) (Parsed, error)
ParseFile(path string, opts ...ParseOption) (Parsed, error)
ParseFiles(paths []string, opts ...ParseOption) ([]Parsed, error)
ParseFileSchema(file File, opts ...ParseOption) (Parsed, error)
WithSupportedEvents(events []string) ParseOption
WithEventValidator(validator EventValidator) ParseOption
```

Il parser ora:

- legge uno o piu' file YAML;
- converte `yaml.File` in `detectors.Definition`;
- parsa `window` con `time.ParseDuration`;
- normalizza severita', eventi richiesti e metadata MITRE;
- chiama `Definition.Validate()` prima di restituire il risultato;
- conserva condizioni e step in una struttura intermedia, perche' verranno
  usati dal detector YAML runtime nello step successivo.

La validazione dei nomi evento e' opzionale tramite `WithSupportedEvents`.
Questa scelta evita di legare il package YAML al registry eBPF/libbpfgo. Il
loader o il registry futuro potranno passare la lista eventi supportati senza
rendere il parser dipendente dal runtime eBPF.

Sono stati aggiunti test per:

- parsing di un detector YAML completo;
- derivazione automatica di `Consumes` quando manca nel file;
- rifiuto di finestre temporali invalide;
- rifiuto di eventi non supportati quando viene passato il validatore;
- riuso della validazione interna di `detectors.Definition`;
- parsing da file.

## Fase 2.4: detector YAML runtime

Stato: completata.

E' stato implementato:

```text
demo_project/pkg/detectors/yaml/detector.go
```

Questo step trasforma il risultato del parser in un detector runtime che
implementa l'interfaccia:

```go
type Detector interface {
    Definition() Definition
    Init(context.Context) error
    OnEvent(context.Context, Event) ([]Alert, error)
    Flush(context.Context, time.Time) ([]Alert, error)
}
```

Componenti principali:

- `NewDetector(parsed Parsed)`;
- `Definition()`;
- `Init()`;
- `OnEvent()`;
- `Flush()`;
- risoluzione campi evento;
- valutazione condizioni;
- produzione alert.

Il detector YAML supporta condizioni su:

- nome evento: `event`, `event_name`, `name`;
- contesto processo: `comm`, `uid`, `pid`, `tid`, `ppid`;
- contesto host: `host_pid`, `host_tid`;
- argomenti evento: `args.<nome_argomento>` e `arg.<nome_argomento>`.

Operatori supportati:

- `eq`;
- `neq`;
- `contains`;
- `prefix`;
- `suffix`;
- `exists`;
- `not_exists`;
- `gt`;
- `lt`.

Limite intenzionale: questa prima implementazione non fa ancora correlazione
stateful completa tra eventi diversi. Se il detector contiene `steps`, lo step
viene valutato come matching locale sul singolo evento. La correlazione ordinata
su finestra temporale verra' gestita dal dispatcher/engine, per evitare che ogni
detector mantenga stato in modo indipendente.

Sono stati aggiunti test per:

- matching di condizioni globali;
- esclusione di eventi non consumati;
- matching di condizioni per-step su argomenti;
- confronto numerico;
- rifiuto di operatori non supportati;
- `Flush` no-op.

## Fase 2.5: registry detector

Stato: completata.

E' stato implementato:

```text
demo_project/pkg/detectors/registry.go
```

Il registry e' il punto centrale dove vengono registrati i detector disponibili.
Responsabilita' implementate:

- conservare `detectorID -> Detector`;
- rifiutare ID duplicati;
- validare le `Definition`;
- restituire la lista dei detector caricati;
- esporre gli eventi consumati globalmente dai detector;
- preparare il terreno per il dispatcher dello step successivo.

API principali:

```go
NewRegistry(detectors ...Detector) (*Registry, error)
Register(detector Detector) error
Get(id string) (Detector, bool)
List() []Detector
Len() int
EventNames() []string
```

Validazioni implementate:

- detector `nil` rifiutato;
- definizione detector invalida rifiutata;
- ID duplicato rifiutato;
- lista detector mantenuta in ordine stabile per ID;
- eventi consumati deduplicati e ordinati.

Questo step non chiama ancora `OnEvent`: rende solo ordinato e validato
l'insieme dei detector disponibili.

## Fase 2.6: dispatcher detector

Stato: completata.

E' stato implementato:

```text
demo_project/pkg/detectors/dispatch.go
```

Il dispatcher e' il primo componente che usa davvero il registry per instradare
eventi verso i detector corretti.

Responsabilita' implementate:

- costruire una mappa `eventName -> []Detector`;
- ricevere un evento decodificato;
- chiamare `OnEvent` solo sui detector interessati a quell'evento;
- raccogliere gli alert prodotti;
- gestire errori dei detector senza fermare l'intera pipeline;
- supportare detector senza eventi dichiarati come wildcard detector;
- chiamare `Flush` su tutti i detector registrati.

API principali:

```go
NewDispatcher(registry *Registry) (*Dispatcher, error)
Dispatch(ctx context.Context, event Event) DispatchResult
Flush(ctx context.Context, now time.Time) DispatchResult
```

Strutture aggiunte:

```go
DispatchResult
DetectorError
```

`DispatchResult` contiene alert prodotti, errori isolati e numero di detector
invocati. `DetectorError` conserva l'ID del detector che ha fallito e wrappa
l'errore originale, cosi' il runtime potra' loggare il problema senza perdere
gli alert degli altri detector.

Per la direzione architetturale attuale, il dispatcher sara' anche il punto
giusto dove preparare le future collective anomalies locali. Non deve ancora
implementare tutta la correlazione stateful, ma deve essere progettato come il
punto centrale dove questa correlazione verra' aggiunta.

## Fase 2.7: engine detector

Stato: completata.

File implementato:

```text
demo_project/pkg/detectors/engine.go
```

L'engine coordina registry e dispatcher e offre al futuro runtime principale
un'interfaccia unica. In questo modo `pkg/ebpf/project.go` non dovra' conoscere
i dettagli interni del registry, dell'indice eventi o degli errori dei singoli
detector.

API implementate:

```go
NewEngine(detectors ...Detector) (*Engine, error)
Register(detector Detector) error
Init(ctx context.Context) error
ProcessEvent(ctx context.Context, event Event) EngineResult
Flush(ctx context.Context, now time.Time) EngineResult
Metrics() Metrics
Registry() *Registry
```

Strutture aggiunte:

```go
Engine
EngineResult
Metrics
```

`ProcessEvent` riceve un evento gia' decodificato, lo passa al dispatcher e
restituisce alert ed errori in un unico risultato. Gli errori dei detector
restano isolati come `DetectorError`, quindi un detector difettoso non blocca
gli altri.

`Flush` prepara il supporto ai detector con finestre temporali corte e alle
future collective anomalies locali. Le metriche sono volutamente minime:
eventi processati, flush eseguiti, detector invocati, alert emessi ed errori.
Questa scelta mantiene basso l'overhead, coerentemente con i limiti
prestazionali osservati sulla VM.

L'engine non deve ancora essere collegato a `pkg/ebpf/project.go`: quello
arrivera' nello step dedicato alla pipeline runtime.

## Fase 2.8: output alert

Stato: completata.

File implementato:

```text
demo_project/pkg/output/alert.go
```

Questo step separa chiaramente l'output degli eventi dall'output degli
alert. Gli eventi descrivono cio' che il kernel ha osservato; gli alert
descrivono invece una decisione prodotta da un detector.

Responsabilita' implementate:

- `AlertRecord` come modello stabile per `detectors.Alert`;
- campi `detector_id`, `detector_name`, titolo, descrizione, severita',
  timestamp, numero di eventi correlati e metadata;
- `policy_names`, ricavati per ora dai metadata dell'alert;
- eventi correlati normalizzati usando lo stesso modello JSON degli eventi raw;
- `formatAlert` per una vista table compatta;
- test unitari per normalizzazione, copia metadata e rendering table.

Una riga table viene preparata in questa forma:

```text
alert=privilege-change detector=demo.privilege-change severity=high events=2
```

Il formato JSON conserva piu' dettagli, inclusi gli eventi correlati, perche'
sara' quello piu' adatto all'integrazione futura con sistemi centralizzati.

Nota: il printer globale non e' ancora stato esteso. Questo avviene nello step
successivo, per mantenere separata la definizione del DTO dall'integrazione nel
runtime di stampa.

## Fase 2.9: printer alert

Stato: completata.

File modificati:

```text
demo_project/pkg/output/printer.go
demo_project/pkg/output/json.go
demo_project/pkg/output/table.go
```

Obiettivo completato: estendere il contratto del printer senza rompere la stampa degli
eventi esistenti. Il runtime dovra' poter stampare eventi raw e alert detector
attraverso lo stesso boundary di output.

Responsabilita' implementate:

- aggiunto `PrintAlert(alert detectors.Alert)` all'interfaccia `Printer`;
- implementato `PrintAlert` in `JSONPrinter`;
- implementato `PrintAlert` in `TablePrinter`;
- mantenuto `Print(event)` per gli eventi raw esistenti;
- aggiunti test per JSON alert, table alert e uso attraverso l'interfaccia
  `Printer`.

Nota storica: questo step ha preparato il printer. Lo step successivo ha poi
collegato l'engine detector al loop eBPF.

## Fase 3: engine userspace

Stato: completata per l'MVP locale.

Il detector engine e' stato inserito dopo il decoder e dopo i filtri base.

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

Implementazione attuale:

- `pkg/cmd/project.go` carica i detector YAML da file o directory;
- ogni YAML viene convertito in un detector runtime;
- `detectors.Engine` viene costruito nel runner e passato a `pkg/ebpf`;
- `pkg/ebpf/project.go` inizializza l'engine in `Init`;
- `handleRawEvent` applica event filter, comm filter e policy filter;
- l'evento filtrato viene inviato a `Engine.ProcessEvent`;
- eventuali errori dei detector vengono scritti su stderr;
- se `--alerts` e' attivo, gli alert vengono stampati tramite
  `Printer.PrintAlert`.

Limite attuale: i detector YAML valutano condizioni locali sul singolo evento.
La correlazione stateful tra piu' eventi, con `group_by` e finestra temporale,
resta da implementare nel dispatcher/engine.

## Fase 3.1: validazione eventi usati da policy e detector

Stato: prossimo step.

Il prossimo file da analizzare e':

```text
demo_project/pkg/events/
```

Obiettivo: fornire helper ufficiali per validare i nomi evento usati da policy
e detector, evitando che YAML errati vengano accettati fino al runtime.

Responsabilita' consigliate:

- aggiungere una lista stabile dei nomi evento supportati;
- aggiungere `Exists(name string) bool`;
- aggiungere `ListNames() []string`;
- usare questi helper nel loader detector YAML;
- successivamente usarli anche nel loader policy.

## Fase 4: output alert

Stato: completata negli step 2.8 e 2.9.

E' stato aggiunto un tipo output separato per gli alert e il printer e' stato
esteso con `PrintAlert`.

Struttura implementata:

```go
type AlertRecord struct {
    Type         string
    DetectorID   string
    DetectorName string
    PolicyNames  []string
    Title        string
    Description  string
    Severity     string
    CreatedAt    time.Time
    EventCount   int
    Events       []eventRecord
    Metadata     map[string]string
}
```

Il formato `table` stampa una riga compatta:

```text
alert=Privilege change followed by exec severity=medium detector=setuid-exec-chain events=2
```

Il formato `json` mantiene tutti gli eventi correlati normalizzati.

## Fase 5: detector iniziali

Stato: iniziata.

Sono stati aggiunti primi esempi YAML testabili:

```text
demo_project/rules/detectors/root_exec.yaml
demo_project/rules/detectors/sensitive_file_open.yaml
demo_project/rules/policies/demo-detectors.yaml
```

Questi esempi seguono una struttura simile agli esempi dichiarativi di Tracee,
ma usano lo schema semplificato del nostro tool.

Detector disponibili:

- `root-exec`: genera alert su `sched_process_exec` eseguito da `uid=0`;
- `sensitive-file-open`: genera alert su evento `security_file_open` con
  `pathname` sotto `/etc/`.

Policy disponibile:

- `demo-detectors`: abilita gli eventi necessari ai due detector demo.

Comando di test:

```bash
sudo ./dist/project \
  --events sched_process_exec,security_file_open \
  --policy rules/policies/demo-detectors.yaml \
  --detectors rules/detectors \
  --alerts \
  --alerts-output table \
  --output table
```

Comandi esca:

```bash
sudo whoami
cat /etc/hostname
```

Nota: questi detector sono ancora locali sul singolo evento. La correlazione
multi-evento basata su `steps`, `group_by` e `window` resta una fase successiva.
Gli esempi usano volutamente hook gia' stabili nel runtime
(`sched_process_exec` e `security_file_open`), cosi' il test del layer
policy/detector non dipende dal debug dei tracepoint syscall `execve`/`open`.

Detector demo ancora da aggiungere dopo la correlazione stateful:

- `privilege-change-then-exec`;
- `sensitive-file-open-after-setuid`;
- `kernel-module-load-and-kprobe`.

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
