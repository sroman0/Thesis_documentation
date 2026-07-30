# Userspace Go e lifecycle eBPF

## Flusso attuale

Il flusso di avvio e':

```text
cmd/project/main.go
  -> cmd.Execute()
  -> rootCmd.RunE
  -> initialize.BPFObject(&cfg)
  -> appcmd.NewProjectRunner(cfg)
  -> load policy manager
  -> compile exact policy event allowlist
  -> load detector YAML files
  -> build detector engine
  -> runner.Run(ctx)
  -> ebpf.New(cfg, policy manager, detector engine)
  -> selectProbes(cfg.Events.Include, cfg.Events.Exclude)
  -> project.Init(ctx)
  -> detector engine Init(ctx)
  -> configure libbpf logging from --log-level
  -> resolve public events and internal probe dependencies
  -> disable autoload for every unselected registered program
  -> load selected eBPF programs
  -> open events perf buffer with InitPerfBuf
  -> attach selected probes
  -> project.Run(ctx)
  -> receive raw bytes from perf buffer
  -> handleRawEvent(raw)
  -> bufferdecoder.DecodeEvent(raw)
  -> event selection guard
  -> comm filter guard
  -> userspace policy filter
  -> output.Printer.Print(event)
  -> detector engine ProcessEvent(event)
  -> output.Printer.PrintAlert(alert)
  -> stdout
```

Nota: l'entrypoint operativo e' `demo_project/cmd/project`. Il vecchio
`main.go` nella root non deve essere usato come riferimento per il comando
principale.

## Logging runtime

Il logging applicativo deve restare separato dall'output di sicurezza.

Il modello previsto e':

```text
runtime logs  -> zap logger
raw events    -> pkg/output event printer
alerts        -> pkg/output alert printer
```

Questa distinzione e' necessaria per non confondere tre tipi di informazione:

- stato del runtime, per esempio startup, attach probe, errori e cleanup;
- eventi kernel decodificati, per esempio `execve` o `security_file_open`;
- alert generati dai detector.

La prima parte della migrazione ha introdotto
`demo_project/pkg/logging/logger.go`, che costruisce un `*zap.Logger` da una
configurazione minimale:

```go
type Config struct {
    Level  string
    Format string
}
```

Il logger supporta `debug`, `info`, `warn`, `error`, usa `info` come default,
scrive su `stderr` e supporta formato `console` o `json`.

La CLI espone ora anche `--log-format`, separato da `--output` e
`--alerts-output`:

- `--output` controlla gli eventi;
- `--alerts-output` controlla gli alert;
- `--log-format` controlla solo i log runtime.

Questa separazione e' importante in container/Kubernetes: eventi e alert
possono restare nel formato piu' comodo per il consumo del tool, mentre i log
runtime possono essere emessi in JSON per una pipeline centralizzata.

Il runner applicativo ora costruisce il logger da `cfg.LogLevel`, usa zap per i
messaggi di lifecycle del layer applicativo e passa il logger al runtime eBPF
con `projectebpf.WithLogger(logger)`. Nel runtime eBPF il logger e' una
dipendenza esplicita; se non viene fornito, il default e' `zap.NewNop()` per
mantenere test e usi programmatici silenziosi.

Le stampe interne residue in `pkg/ebpf/project.go` sono state migrate a zap per
i messaggi gestiti dal runtime:

- attach probe;
- primi eventi ricevuti/decodificati;
- drop reason in debug;
- eventi persi dal perf buffer;
- errori decode/output;
- errori detector e alert printer.

Anche il callback libbpf passa dal logger zap. I messaggi CO-RE restano
controllati da `--log-level` e vengono marcati con:

```text
source=libbpf
libbpf_level=<livello numerico libbpf>
```

Questo mantiene un solo sink diagnostico senza confondere i messaggi libbpf con
eventi o alert.

La flag `--log-level` continuera' a controllare la verbosita', ma i messaggi
runtime useranno livelli strutturati:

- `error`: solo errori rilevanti;
- `warn`: condizioni anomale non fatali;
- `info`: lifecycle essenziale;
- `debug`: diagnostica come attach probe, drop reason e primi eventi ricevuti.

Il piano dettagliato e' in
[Piano logging strutturato con zap](../next-steps/zap-logging-plan.md).

## Preparazione config

`initialize.BPFObject()` prepara:

- path assoluto dell'oggetto eBPF;
- path BTF;
- byte dell'oggetto eBPF in `cfg.BPFObjBytes`.

Questo separa:

- validazione statica della config;
- risoluzione filesystem;
- runtime eBPF.

La config contiene anche `Events.Include`, `Events.Exclude` e
`Events.FilterComms`, popolati dalle flag CLI. La flag `--list-events` e'
gestita prima del caricamento dell'oggetto eBPF, quindi puo' essere usata senza
privilegi root per ispezionare gli eventi disponibili.

- `--events`: eventi da abilitare, separati da virgola;
- `--drop-events`: eventi da disabilitare dopo la selezione iniziale;
- `--comms`: command names da mantenere dopo la decodifica.
- `--list-events`: stampa gli eventi supportati e termina.
- `--policy`: file o directory di policy YAML.
- `--detectors`: file o directory di detector YAML; abilita automaticamente il
  layer detector.
- `--alerts`: abilita la stampa degli alert prodotti dai detector.
- `--alerts-output`: formato degli alert, `json` o `table`.
- `--log-format`: formato dei log runtime, `console` o `json`.

Se `--events` non viene passato, il runtime abilita tutti gli eventi supportati.
`--drop-events cap_capable` e' utile per ridurre il rumore durante i test.
`--comms ls,whoami` e' utile per demo mirate, perche' stampa solo eventi il cui
`comm` decodificato corrisponde ai nomi indicati.

Le policy e i detector vengono preparati prima dell'avvio eBPF. Il runner
carica le policy nel `policy.Manager`, analizza le definizioni detector YAML,
applica gli eventuali selettori metadata dichiarati dalle policy e costruisce
il `detectors.Engine` soltanto con i detector scelti. Senza un selettore
detector, il comportamento resta compatibile e vengono caricati tutti i YAML
indicati dalla CLI.

La selezione supporta ID, severity, tactic/technique MITRE, tag e stato
stateful. Avviene nel bootstrap, quindi non aggiunge matching metadata alla hot
path di ogni evento e non alloca stato per detector esclusi.

Il catalogo degli ID non è globale: viene costruito dalle definizioni presenti
nei file o nelle directory ricevute tramite `--detectors`. Gli ID espliciti
della policy vengono validati contro questo insieme. Se `--detectors` non viene
fornito, il runner mantiene attivo il `policy.Manager`, salta la selezione
detector e registra un warning; non crea un engine detector vuoto.

Durante questa fase il runner valida anche i nomi evento dichiarati da policy e
detector contro `pkg/events/spec.go`. Se un file YAML usa un evento non presente
nel registry decoder, il tool fallisce prima dell'avvio eBPF. Questo evita run
ambigue in cui una policy o un detector sembra caricato, ma in realta' ascolta
un evento che userspace non puo' decodificare.

La config contiene anche `LogLevel`, popolato da `--log-level`. Questo valore
controlla sia il livello runtime sia il filtro dei log libbpf/CO-RE:

- `debug`: mostra anche le relocation verbose;
- `info`: nasconde i log `LIBBPF_DEBUG`;
- `warn`/`error`: mantiene solo warning libbpf.

I log libbpf ammessi dal filtro vengono emessi da zap: warning libbpf a livello
`warn`, diagnostica libbpf a livello `debug`.

## Load e attach

In `Project.Init()`:

1. rimozione limite memlock;
2. apertura del modulo eBPF con `libbpfgo`;
3. risoluzione dei probe pubblici e delle dipendenze interne;
4. disabilitazione autoload dei programmi registrati non selezionati;
5. caricamento dei programmi rimasti abilitati nell'oggetto eBPF;
6. apertura perf buffer `events`;
7. attach dei programmi selezionati, incluse le dipendenze interne.

La selezione dei programmi vive in `pkg/ebpf/probes/probes.go`. Ogni probe
collega il nome evento decodificato al programma eBPF e all'hook kernel da
usare.

Quando tutte le regole operative di policy dichiarano una lista `include`
esplicita, `policy.Manager.EffectiveEventSelection()` ne calcola l'unione. Se
anche la CLI specifica `--events`, viene usata l'intersezione. Il risultato
raggiunge `probes.Select()` e poi `ConfigureAutoload()` prima di
`BPFLoadObject()`: gli altri programmi registrati non vengono verificati,
caricati nel kernel o agganciati.

`internal` e `impliedBy` hanno ruoli diversi. `internal` nasconde un probe dal
contratto CLI/output; `impliedBy` dichiara quali eventi pubblici ne richiedono
l'attach. Solo le dipendenze interne risolte da `Select()` restano in autoload.
Non vengono aggiunte al set degli eventi pubblici e quindi non diventano
selezionabili con `--events`. Una policy con `include` vuota significa "tutti
gli eventi pubblici" e riduce molto meno il set di programmi.

I programmi networking `sock_alloc_file`, `sock_alloc_file_ret`,
`security_sk_clone`, `security_socket_recvmsg`, `security_socket_sendmsg` e
`cgroup_bpf_run_filter_skb` seguono questa regola: sono `internal` perché
mantengono contesto e mappe, e hanno `impliedBy` perché devono essere attaccati
quando viene selezionato ingress o egress cgroup.

Il registry distingue ora due categorie:

- probe pubblici, cioe' eventi con schema decoder in `pkg/events/spec.go`,
  visibili con `--list-events` e selezionabili con `--events`;
- probe interni, usati per aggiornare stato kernel-side o supportare altre
  feature, ma non esposti come eventi utente.

Questa separazione evita di promettere in CLI nomi che non producono record
decodificabili. I test del package `pkg/ebpf/probes` verificano anche che ogni
probe pubblico abbia una specifica decoder corrispondente.

Il registry supporta:

- raw tracepoint;
- tracepoint classici;
- kprobe;
- kretprobe.
- cgroup skb ingress/egress.

Questo permette di usare hook dedicati come `syscalls/sys_enter_execve` quando
il kernel target li espone, evitando di filtrare manualmente ogni syscall da un
hook generico. La presenza dei kretprobe permette inoltre di modellare eventi
come `do_init_module`, `register_kprobe` e `kallsyms_lookup_name`, dove il dato
piu' utile e' disponibile solo al ritorno della funzione kernel.

## Lettura e decoding eventi

In `Project.Run()`:

1. il runtime ascolta `perfBufChannel`;
2. per ogni record ricevuto chiama `handleRawEvent(raw)`;
3. `handleRawEvent` chiama `bufferdecoder.DecodeEvent(...)`;
4. scarta eventi non abilitati dalla selezione runtime;
5. scarta eventi non ammessi dal filtro `--comms`, se configurato;
6. applica policy e detector;
7. passa eventi e alert ai rispettivi printer.

Esempio di output osservato:

```json
{"event_name":"cap_capable","args":[{"name":"cap","type":1,"value":19}]}
```

`cap_capable` e' un hook molto rumoroso: vedere molte righe ripetute e'
normale.

## Nota su libbpfgo e CGO

Il runtime e' stato migrato verso `github.com/aquasecurity/libbpfgo`, in linea
con la direzione architetturale di Tracee. Questo richiede CGO e un `libbpf`
compilato localmente dal submodule:

```text
3rdparty/libbpfgo/libbpf/src
```

Il Makefile prepara le variabili:

- `PKG_CONFIG_PATH`;
- `PROJECT_CGO_CFLAGS`;
- `PROJECT_CGO_LDFLAGS`.

Per questo motivo comandi diretti come:

```bash
sudo go run ./cmd/project --help
```

possono fallire se non ricevono gli stessi flag CGO. I target Makefile
incapsulano questa configurazione.

## Nota sul transport eventi

Tutti gli hook, inclusi quelli networking, inviano i record con
`events_perf_submit`. Il runtime apre soltanto il perf buffer `events`. La
rimozione del secondo transport evita una mappa ring buffer inutilizzata da
16 MiB e semplifica init, loop di lettura e cleanup.

## Output layer

La stampa degli eventi non vive piu' direttamente in `Project.Run()`. Il runtime
usa `pkg/output` tramite l'interfaccia `Printer`.

Formati attuali:

- `json`: mantiene una riga JSON per evento, adatta a parsing automatico, ma
  usa una vista normalizzata invece del formato raw del decoder;
- `table`: produce una riga compatta leggibile a terminale, con `event`, `pid`,
  `tid`, `uid`, `comm` e argomenti.

Nel formato JSON, campi C-style come `comm` e `uts_name` vengono convertiti in
stringhe. Gli argomenti `cap` vengono arricchiti con una label simbolica, ad
esempio `CAP_SYS_ADMIN`. Lo stesso layer traduce errno, segnali, flag di
memoria, namespace, flag mount/umount e puntatori kernel. Gli eventi
`switch_task_ns`, `security_sb_mount`, `security_sb_umount`,
`security_inode_unlink`, `proc_create`, `register_kprobe` e
`kallsyms_lookup_name` non richiedono un reader dedicato: usano lo stesso perf
buffer, decoder indicizzato e printer degli altri eventi.

Questa scelta segue il ruolo del `sink` di Tracee in forma semplificata: il
runtime legge e decodifica, mentre il layer output decide come presentare
l'evento.

## Cleanup

`Project.Close()` chiude:

1. link eBPF;
2. perf buffer reader;
3. modulo eBPF.

Ordine importante: prima detach/close dei link, poi risorse condivise.

## Differenza rispetto a Tracee

Tracee usa un sistema `ProbeGroup` piu' ricco, con:

- gestione autoload;
- symbol table;
- fallback per kprobe;
- compatibilita' kernel.

Il progetto usa un registry statico di probe selezionabili, sufficiente per MVP.
Questa e' una deviazione intenzionale: mantiene il design vicino a Tracee, ma
riduce complessita' e rischio sul kernel Rocky Linux 4.18.

Dal 2026-05-19 questa deviazione e' stata resa piu' esplicita: quando il target
kernel e' noto, il progetto preferisce hook specifici e meno generici. La scelta
riduce portabilita' teorica, ma rende piu' chiara e piu' economica la pipeline
sulla VM di tesi.

## Collegamenti

- [Timeline](../timeline.md)
- [Decision log](../decisions/decision-log.md)
- [Comandi utili](../debugging/commands.md)
- [Decoder Go](decoder.md)
- [Output layer](output.md)
