# Userspace Go e lifecycle eBPF

## Flusso attuale

Il flusso di avvio e':

```text
main.go
  -> cmd.Execute()
  -> rootCmd.RunE
  -> cmdcobra.GetProjectRunner(cliCfg)
  -> initialize.BpfObject(&cfg)
  -> appcmd.NewProjectRunner(cfg)
  -> runner.Run(ctx)
  -> ebpf.New(cfg)
  -> selectProbes(cfg.Events.Include, cfg.Events.Exclude)
  -> project.Init(ctx)
  -> attach selected probes
  -> project.Run(ctx)
  -> bufferdecoder.DecodeEvent(record.RawSample)
  -> event selection guard
  -> output.Printer.Print(event)
  -> stdout
```

## Preparazione config

`initialize.BpfObject()` prepara:

- path assoluto dell'oggetto eBPF;
- path BTF;
- byte dell'oggetto eBPF in `cfg.BPFObjBytes`.

Questo separa:

- validazione statica della config;
- risoluzione filesystem;
- runtime eBPF.

La config contiene anche `Events.Include` e `Events.Exclude`, popolati dalle
flag CLI:

- `--events`: eventi da abilitare, separati da virgola;
- `--drop-events`: eventi da disabilitare dopo la selezione iniziale.

Se `--events` non viene passato, il runtime abilita tutti gli eventi supportati.
`--drop-events cap_capable` e' utile per ridurre il rumore durante i test.

## Load e attach

In `Project.Init()`:

1. rimozione limite memlock;
2. load collection spec da `BPFObjBytes`;
3. load BTF;
4. creazione collection;
5. apertura ring buffer;
6. attach dei soli programmi selezionati.

La selezione dei programmi vive in `pkg/ebpf/probes/probes.go`. Ogni probe collega il
nome evento decodificato al programma eBPF e all'hook kernel da usare.

## Lettura e decoding eventi

In `Project.Run()`:

1. il runtime legge `ringbuf.Record` da `events`;
2. prende `record.RawSample`;
3. chiama `bufferdecoder.DecodeEvent(...)`;
4. scarta eventi non abilitati dalla selezione runtime;
5. passa l'evento al printer configurato;
6. stampa una riga su stdout.

Questo sostituisce il loop precedente che drenava la ring buffer e scartava i
record.

Esempio di output osservato:

```json
{"event_name":"cap_capable","args":[{"name":"cap","type":1,"value":19}]}
```

`cap_capable` e' un hook molto rumoroso: vedere molte righe ripetute e'
normale.

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
3. collection.

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
