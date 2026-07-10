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
  -> load detector YAML files
  -> build detector engine
  -> runner.Run(ctx)
  -> ebpf.New(cfg, policy manager, detector engine)
  -> selectProbes(cfg.Events.Include, cfg.Events.Exclude)
  -> project.Init(ctx)
  -> detector engine Init(ctx)
  -> configure libbpf logging from --log-level
  -> open events_ringbuf with InitRingBuf
  -> open events perf buffer with InitPerfBuf
  -> attach selected probes
  -> project.Run(ctx)
  -> receive raw bytes from ring buffer or perf buffer
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

La prossima migrazione introdurra' `go.uber.org/zap` come logger strutturato
del runtime. Il logger verra' costruito nel runner applicativo e passato al
runtime eBPF come dipendenza esplicita. Non deve essere una variabile globale.

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

Se `--events` non viene passato, il runtime abilita tutti gli eventi supportati.
`--drop-events cap_capable` e' utile per ridurre il rumore durante i test.
`--comms ls,whoami` e' utile per demo mirate, perche' stampa solo eventi il cui
`comm` decodificato corrisponde ai nomi indicati.

Le policy e i detector vengono preparati prima dell'avvio eBPF. Il runner
carica le policy nel `policy.Manager`, carica i detector YAML, costruisce il
`detectors.Engine` e passa entrambi a `pkg/ebpf/project.go`.

La config contiene anche `LogLevel`, popolato da `--log-level`. Questo valore
controlla sia il livello runtime sia il filtro dei log libbpf/CO-RE:

- `debug`: mostra anche le relocation verbose;
- `info`: nasconde i log `LIBBPF_DEBUG`;
- `warn`/`error`: mantiene solo warning libbpf.

## Load e attach

In `Project.Init()`:

1. rimozione limite memlock;
2. apertura del modulo eBPF con `libbpfgo`;
3. caricamento dell'oggetto eBPF;
4. apertura ring buffer `events_ringbuf`;
5. apertura perf buffer `events`;
6. attach dei soli programmi selezionati.

La selezione dei programmi vive in `pkg/ebpf/probes/probes.go`. Ogni probe
collega il nome evento decodificato al programma eBPF e all'hook kernel da
usare.

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

Questo permette di usare hook dedicati come `syscalls/sys_enter_execve` quando
il kernel target li espone, evitando di filtrare manualmente ogni syscall da un
hook generico. La presenza dei kretprobe permette inoltre di modellare eventi
come `do_init_module`, `register_kprobe` e `kallsyms_lookup_name`, dove il dato
piu' utile e' disponibile solo al ritorno della funzione kernel.

## Lettura e decoding eventi

In `Project.Run()`:

1. il runtime ascolta `ringBufChannel`;
2. ascolta anche `perfBufChannel`;
3. per ogni record ricevuto chiama `handleRawEvent(raw)`;
4. `handleRawEvent` chiama `bufferdecoder.DecodeEvent(...)`;
5. scarta eventi non abilitati dalla selezione runtime;
6. scarta eventi non ammessi dal filtro `--comms`, se configurato;
7. passa l'evento al printer configurato;
8. stampa una riga su stdout.

Questo sostituisce il loop precedente basato solo sulla ring buffer.

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

## Nota su eventi networking

La versione precedente leggeva solo `events_ringbuf`, mentre gli hook networking
integrati dal branch collaboratore usavano in gran parte `events_perf_submit`.
La versione attuale apre anche il perf buffer `events`, quindi gli eventi
networking inviati con `events_perf_submit` possono arrivare allo stesso decoder
e allo stesso output.

Stato attuale:

- gli hook correnti usano `events_perf_submit`;
- il reader perf buffer e' quindi il percorso operativo principale;
- ring buffer e `events_ringbuf_submit` restano nel codice per versatilita',
  fallback o confronti futuri.

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
2. ring buffer reader;
3. perf buffer reader;
4. modulo eBPF.

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
