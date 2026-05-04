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
  -> project.Init(ctx)
  -> project.Run(ctx)
  -> bufferdecoder.DecodeEvent(record.RawSample)
  -> json.Marshal(event)
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

## Load e attach

In `Project.Init()`:

1. rimozione limite memlock;
2. load collection spec da `BPFObjBytes`;
3. load BTF;
4. creazione collection;
5. apertura ring buffer;
6. attach programmi.

## Lettura e decoding eventi

In `Project.Run()`:

1. il runtime legge `ringbuf.Record` da `events`;
2. prende `record.RawSample`;
3. chiama `bufferdecoder.DecodeEvent(...)`;
4. converte l'evento in JSON;
5. stampa una riga su stdout.

Questo sostituisce il loop precedente che drenava la ring buffer e scartava i
record.

Esempio di output osservato:

```json
{"event_name":"cap_capable","args":[{"name":"cap","type":1,"value":19}]}
```

`cap_capable` e' un hook molto rumoroso: vedere molte righe ripetute e'
normale.

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

Il progetto usa una lista statica di programmi, sufficiente per MVP.

## Collegamenti

- [Timeline](../timeline.md)
- [Decision log](../decisions/decision-log.md)
- [Comandi utili](../debugging/commands.md)
- [Decoder Go](decoder.md)
