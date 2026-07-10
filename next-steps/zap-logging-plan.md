# Piano logging strutturato con zap

## Contesto

Il tool oggi usa ancora diversi messaggi manuali basati su `fmt.Printf`,
`fmt.Fprintf(os.Stderr, ...)` o stampe ad hoc. Questo approccio e' sufficiente
per debug iniziale, ma diventa fragile quando il runtime cresce:

- i messaggi non hanno un formato unico;
- non sempre e' chiaro se un messaggio sia `debug`, `info`, `warning` o errore;
- le stampe diagnostiche possono confondersi con eventi e alert;
- stampare troppo nella pipeline eventi puo' aumentare CPU e rumore operativo.

L'obiettivo e' introdurre un logging applicativo centralizzato usando
[`go.uber.org/zap`](https://github.com/uber-go/zap), lasciando separati gli
output di sicurezza del tool:

```text
event=...
type=alert ...
```

Eventi e alert restano output di prodotto. I log zap devono descrivere lo stato
del runtime, errori interni, debug di attach, policy, detector e decoder.

## Principio architetturale

Separare tre flussi:

```text
runtime logs  -> logger zap
raw events    -> pkg/output event printer
alerts        -> pkg/output alert printer
```

Questa distinzione evita di trattare un evento kernel come semplice log
applicativo. Un evento `security_file_open` o un alert detector sono dati del
monitoraggio; un messaggio come "attached probes=2" e' invece log di runtime.

## Livelli

Usare i livelli gia' esposti dalla CLI con `--log-level`:

| Livello | Uso previsto |
| --- | --- |
| `error` | errori che impediscono o degradano il runtime |
| `warn` | condizioni anomale ma non fatali |
| `info` | lifecycle essenziale: startup, config, policy/detector caricati |
| `debug` | dettagli diagnostici: attach probe, drop reason, first event, detector debug |

Regola pratica: tutto cio' che puo' apparire per ogni evento deve essere
`debug`, non `info`.

## Scelta zap

Usare principalmente `*zap.Logger`, non `SugaredLogger`, nei componenti di
runtime.

Motivo:

- campi strutturati e tipizzati (`zap.String`, `zap.Int`, `zap.Error`);
- minore overhead rispetto a formattazione `printf`;
- migliore controllo sulle allocazioni nella hot path;
- output piu' facile da filtrare o convertire in JSON in futuro.

Il `SugaredLogger` puo' restare fuori dal runtime critico, ma non dovrebbe
essere il default nei percorsi che processano eventi.

## Fase 1 - Package logging

Creare:

```text
demo_project/pkg/logging/logger.go
demo_project/pkg/logging/logger_test.go
```

Responsabilita':

- costruire un `*zap.Logger` dalla configurazione del tool;
- tradurre stringhe `debug|info|warn|error` in livelli zap;
- scegliere output iniziale su `stderr`;
- usare formato console leggibile come default;
- rendere semplice un futuro formato JSON.

API proposta:

```go
type Config struct {
    Level  string
    Format string // console|json, opzionale in prima fase
}

func New(cfg Config) (*zap.Logger, error)
```

Test minimi:

- livello `debug` accettato;
- livello `info` accettato;
- livello invalido rifiutato;
- logger non nil;
- `Sync()` chiamabile senza panic nei test.

## Fase 2 - Wiring nel runner

Collegare il logger nel runner applicativo:

```text
demo_project/pkg/cmd/project.go
```

Il runner deve:

1. leggere `cfg.LogLevel`;
2. costruire il logger con `pkg/logging`;
3. passarlo al runtime eBPF;
4. fare `defer logger.Sync()`.

Non usare variabili globali per il logger. Il logger deve essere una dipendenza
esplicita, come policy manager e detector engine.

## Fase 3 - Wiring nel runtime eBPF

Modificare:

```text
demo_project/pkg/ebpf/project.go
```

Aggiungere:

```go
logger *zap.Logger
```

e una option:

```go
func WithLogger(logger *zap.Logger) Option
```

Se il logger non viene fornito, usare:

```go
zap.NewNop()
```

Questo mantiene i test silenziosi e impedisce nil pointer.

## Fase 4 - Migrazione log non-hot-path

Prima sostituire solo i messaggi di lifecycle:

- startup runtime;
- bpf object e BTF selezionati;
- policy layer configurato;
- detector layer configurato;
- alert output configurato;
- numero di probe attaccati;
- errori di init e cleanup.

Esempio:

```go
logger.Info("project runtime started",
    zap.String("bpf_object", cfg.BPFObject),
    zap.String("btf", cfg.BTF),
    zap.String("output", cfg.Output),
    zap.String("log_level", cfg.LogLevel),
)
```

Questa fase non deve cambiare il comportamento degli eventi o degli alert.

## Fase 5 - Migrazione diagnostica debug

Poi sostituire i messaggi diagnostici aggiunti durante il debug:

- `attached probe`;
- `received first raw event`;
- `decoded first event`;
- `drop event reason=...`;
- `detector error`;
- `print alert error`.

Esempio:

```go
logger.Debug("drop event",
    zap.Uint32("event_id", uint32(event.ID)),
    zap.String("event_name", event.EventName),
    zap.String("reason", "event_selection"),
)
```

Regola: non usare `fmt.Sprintf` per costruire messaggi debug. Se il livello
debug e' disabilitato, non dobbiamo pagare il costo della formattazione.

## Fase 6 - Hot path policy

Nel percorso:

```text
handleRawEvent -> DecodeEvent -> filters -> output -> detectors
```

loggare solo:

- errori;
- summary diagnostici in `debug`;
- drop reason con campi compatti;
- mai payload completi o struct intere.

Da evitare:

```go
logger.Debug(fmt.Sprintf("event=%+v", event))
```

Da preferire:

```go
logger.Debug("event decoded",
    zap.Uint32("event_id", uint32(event.ID)),
    zap.String("event_name", event.EventName),
    zap.String("comm", event.Context.CommString()),
    zap.Int("args", len(event.Args)),
)
```

## Fase 7 - libbpf logs

Tenere separati:

- log del nostro tool;
- log verbose di libbpf/CO-RE.

Prima versione:

```text
--log-level debug -> abilita debug tool + log verbose libbpf
--log-level info  -> lifecycle normale, libbpf silenzioso
--log-level error -> solo errori applicativi
```

Se servira' piu' controllo, aggiungere in futuro:

```text
--libbpf-log-level
```

Non introdurla nella prima migrazione.

## Fase 8 - Documentazione e benchmark

Dopo la migrazione:

- aggiornare `README.md`;
- aggiornare `documentation/debugging/commands.md`;
- aggiornare `documentation/implementation/userspace-lifecycle.md`;
- misurare CPU con:

```bash
watch -n 1 "ps -C project -o pid,%cpu,%mem,rss,nlwp,etime,cmd"
```

Confrontare almeno:

```text
--log-level error
--log-level info
--log-level debug
```

Il target operativo resta mantenere il tool sotto circa il 5% di un core nelle
run normali, quindi `debug` deve essere considerato modalita' diagnostica e non
profilo di produzione.

## Ordine consigliato

1. Creare `pkg/logging`.
2. Aggiungere test unitari del logger.
3. Collegare il logger nel runner.
4. Passare il logger al runtime eBPF.
5. Migrare log di lifecycle.
6. Migrare log diagnostici.
7. Aggiornare documentazione utente.
8. Eseguire test e benchmark rapidi.

## Criteri di accettazione

- `--log-level error` non stampa log informativi o debug;
- `--log-level info` mostra solo lifecycle essenziale;
- `--log-level debug` mostra attach, drop reason e diagnostica utile;
- eventi e alert continuano a essere stampati dai printer, non dal logger;
- i test Go restano verdi;
- il logger non introduce output nei test unitari;
- non vengono costruite stringhe pesanti nella hot path quando debug e'
  disabilitato.

