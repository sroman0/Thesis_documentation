# Userspace Go e lifecycle eBPF

## Flusso attuale

Il flusso di avvio e':

```text
cmd/project/main.go
  -> cmd.Execute()
  -> rootCmd.RunE
  -> initialize.BPFObject(&cfg)
  -> appcmd.NewProjectRunner(cfg)
  -> runner.Run(ctx)
  -> ebpf.New(cfg)
  -> selectProbes(cfg.Events.Include, cfg.Events.Exclude)
  -> project.Init(ctx)
  -> open events_ringbuf with InitRingBuf
  -> open events perf buffer with InitPerfBuf
  -> attach selected probes
  -> project.Run(ctx)
  -> receive raw bytes from ring buffer or perf buffer
  -> handleRawEvent(raw)
  -> bufferdecoder.DecodeEvent(raw)
  -> event selection guard
  -> comm filter guard
  -> output.Printer.Print(event)
  -> stdout
```

Nota: l'entrypoint operativo e' `demo_project/cmd/project`. Il vecchio
`main.go` nella root non deve essere usato come riferimento per il comando
principale.

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
`Events.FilterComms`, popolati dalle flag CLI:

- `--events`: eventi da abilitare, separati da virgola;
- `--drop-events`: eventi da disabilitare dopo la selezione iniziale;
- `--comms`: command names da mantenere dopo la decodifica.

Se `--events` non viene passato, il runtime abilita tutti gli eventi supportati.
`--drop-events cap_capable` e' utile per ridurre il rumore durante i test.
`--comms ls,whoami` e' utile per demo mirate, perche' stampa solo eventi il cui
`comm` decodificato corrisponde ai nomi indicati.

## Load e attach

In `Project.Init()`:

1. rimozione limite memlock;
2. apertura del modulo eBPF con `libbpfgo`;
3. caricamento dell'oggetto eBPF;
4. apertura ring buffer `events_ringbuf`;
5. apertura perf buffer `events`;
6. attach dei soli programmi selezionati.

La selezione dei programmi vive in `pkg/ebpf/probes/probes.go`. Ogni probe collega il
nome evento decodificato al programma eBPF e all'hook kernel da usare.

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

Opzioni future:

- mantenere temporaneamente il modello duale;
- migrare tutti gli hook a perf buffer, seguendo Tracee piu' da vicino;
- migrare gli hook networking a `events_ringbuf_submit`, se il ring buffer
  resta affidabile sul kernel Rocky 4.18;
- mantenere entrambi i canali, ma documentando chiaramente quali eventi usano
  quale canale.

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
esempio `CAP_SYS_ADMIN`.

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

## Collegamenti

- [Timeline](../timeline.md)
- [Decision log](../decisions/decision-log.md)
- [Comandi utili](../debugging/commands.md)
- [Decoder Go](decoder.md)
- [Output layer](output.md)
