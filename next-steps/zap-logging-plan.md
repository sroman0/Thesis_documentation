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
- output piu' facile da filtrare, incluso il formato JSON gia' esposto tramite
  `--log-format json`.

Il `SugaredLogger` puo' restare fuori dal runtime critico, ma non dovrebbe
essere il default nei percorsi che processano eventi.

## Fase 1 - Package logging

Stato: completata.

Creati:

```text
demo_project/pkg/logging/logger.go
demo_project/pkg/logging/logger_test.go
```

Responsabilita':

- costruire un `*zap.Logger` dalla configurazione del tool;
- tradurre stringhe `debug|info|warn|error` in livelli zap;
- scegliere output iniziale su `stderr`;
- usare formato console leggibile come default;
- supportare formato `console` e `json`.

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
- livelli `warn` ed `error` accettati;
- livello invalido rifiutato;
- formato `json` accettato;
- formato invalido rifiutato;
- logger non nil;
- `Sync()` chiamabile senza panic nei test.

Verifica:

```bash
GOCACHE=/tmp/go-build go test ./pkg/logging
```

La Fase 1 non cambia ancora il comportamento runtime del tool. Introduce solo
il costruttore del logger e la dipendenza `go.uber.org/zap`.

## Fase 2 - Wiring nel runner

Stato: completata.

Collegato il logger nel runner applicativo:

```text
demo_project/pkg/cmd/project.go
```

Il runner ora:

1. legge `cfg.LogLevel`;
2. costruisce il logger con `pkg/logging`;
3. stampa i log lifecycle del runner con campi zap;
4. passa il logger al runtime eBPF;
5. esegue `defer logger.Sync()`.

Non usare variabili globali per il logger. Il logger deve essere una dipendenza
esplicita, come policy manager e detector engine.

Verifica:

```bash
PKG_CONFIG_PATH=./dist/libbpf/obj \
CGO_CFLAGS="$(PKG_CONFIG_PATH=./dist/libbpf/obj pkg-config --cflags libbpf 2>/dev/null) -I$(pwd)/3rdparty/libbpfgo" \
CGO_LDFLAGS="$(PKG_CONFIG_PATH=./dist/libbpf/obj pkg-config --libs libbpf 2>/dev/null)" \
GOCACHE=/tmp/go-build \
go test ./pkg/logging ./pkg/cmd ./pkg/ebpf
```

## Fase 3 - Wiring nel runtime eBPF

Stato: completata per il wiring, non ancora per la migrazione dei log interni.

Modificato:

```text
demo_project/pkg/ebpf/project.go
```

Aggiunto:

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

Il prossimo passaggio non e' piu' wiring, ma migrazione graduale delle stampe
interne di `pkg/ebpf/project.go`.

## Fase 4 - Migrazione log non-hot-path

Stato: completata.

Sono stati migrati i messaggi runtime interni non gestiti dal callback libbpf:

- numero di probe attaccati;
- eventi persi dal perf buffer;
- errori di decode;
- errori di stampa evento;
- errori detector;
- errori di stampa alert.

Esempio implementato:

```go
p.logger.Warn("perf buffer lost events", zap.Uint64("lost_events", lost))
```

Questa fase non deve cambiare il comportamento degli eventi o degli alert.

## Fase 5 - Migrazione diagnostica debug

Stato: completata per la diagnostica eBPF interna.

Sono stati sostituiti i messaggi diagnostici aggiunti durante il debug:

- `attached probe`;
- `received first raw event`;
- `decoded first event`;
- `drop event reason=...`;

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

Stato: completata.

I log prodotti da libbpf sono stati collegati allo stesso logger zap del
runtime, ma restano distinguibili tramite campi strutturati:

```text
source=libbpf
libbpf_level=<livello numerico libbpf>
```

La policy di filtro resta quella originale:

```text
--log-level debug -> abilita debug tool + log verbose libbpf
--log-level info  -> lifecycle normale, libbpf silenzioso
--log-level warn  -> mostra warning libbpf
--log-level error -> mostra warning libbpf
```

I messaggi `LIBBPF_WARN` vengono mappati a `logger.Warn`, mentre gli altri
messaggi libbpf ammessi dal filtro vengono mappati a `logger.Debug`.

Se servira' piu' controllo, aggiungere in futuro:

```text
--libbpf-log-level
```

Non e' stata introdotta nella prima migrazione per evitare di aggiungere una
seconda flag prima di avere misure reali sul rumore prodotto.

## Fase 8 - Documentazione e benchmark

Stato: completata per la parte configurazione/documentazione.

E' stata aggiunta la flag:

```text
--log-format console|json
```

Il formato controlla solo i log runtime. Non modifica eventi e alert:

```text
--output        -> eventi
--alerts-output -> alert
--log-format    -> log runtime
```

Questo rende il tool piu' adatto a container e pod Kubernetes, dove i log
runtime possono essere raccolti in JSON da una pipeline centralizzata.

Resta da fare la parte benchmark:

- misurare CPU con:

```bash
watch -n 1 "ps -C project -o pid,%cpu,%mem,rss,nlwp,etime,cmd"
```

Confrontare almeno:

```text
--log-level error
--log-level info
--log-level debug
--log-format console
--log-format json
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
7. Instradare libbpf nel logger.
8. Aggiungere `--log-format`.
9. Aggiornare documentazione utente.
10. Eseguire test e benchmark rapidi.

## Criteri di accettazione

- `--log-level error` non stampa log informativi o debug;
- `--log-level info` mostra solo lifecycle essenziale;
- `--log-level debug` mostra attach, drop reason e diagnostica utile;
- eventi e alert continuano a essere stampati dai printer, non dal logger;
- i test Go restano verdi;
- il logger non introduce output nei test unitari;
- non vengono costruite stringhe pesanti nella hot path quando debug e'
  disabilitato.

## Stato finale della prima migrazione

La prima migrazione zap e' completa.

Implementato:

- package `pkg/logging`;
- flag `--log-level`;
- flag `--log-format console|json`;
- log runtime su `stderr`;
- libbpf/CO-RE instradato in zap;
- rimozione delle stampe runtime manuali principali;
- documentazione e test mirati.

Decisione architetturale finale: zap resta il sink della diagnostica runtime.
Eventi e alert restano nel package `pkg/output`, perche' sono dati di
monitoraggio e non log applicativi.

Prossimi miglioramenti, separati da zap:

- rendere il formato table degli alert ancora piu' esplicito;
- usare `--alerts-only` nelle demo detector-focused quando serve mostrare solo
  gli alert senza stampare gli eventi raw;
- documentare meglio lo schema JSON degli eventi;
- valutare destinazioni separate per eventi, alert e log runtime.
