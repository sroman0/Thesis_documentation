# Overview implementazione

## Obiettivo tecnico

Il progetto implementa un runtime security monitor basato su eBPF, ispirato a Tracee ma semplificato per il contesto di tesi.

La pipeline desiderata e':

```text
kernel hook eBPF
  -> costruzione evento
  -> ring buffer / perf buffer
  -> userspace Go
  -> decoder
  -> output
  -> detection / alert
```

Stato raggiunto per l'MVP:

```text
kernel hook eBPF
  -> costruzione evento
  -> ring buffer / perf buffer
  -> userspace Go
  -> decoder
  -> event selection
  -> comm filter
  -> output printer
```

## Componenti principali

### eBPF C

Percorso:

```text
demo_project/pkg/ebpf/c/
```

Contiene:

- `project.bpf.c`: hook eBPF;
- `types.h`: formato eventi e ID;
- `maps.h`: mappe eBPF;
- `common/`: helper condivisi.

### Userspace Go

Percorso:

```text
demo_project/pkg/ebpf/project.go
```

Responsabilita':

- caricare l'oggetto eBPF;
- caricare BTF;
- creare collection eBPF;
- aprire ring buffer;
- aprire perf buffer;
- selezionare quali eventi/probe abilitare;
- attaccare programmi agli hook selezionati;
- leggere eventi;
- decodificare record raw da ring buffer o perf buffer;
- applicare filtro eventi e filtro `comm`;
- inviare eventi decodificati al printer configurato.

### Decoder

Percorso:

```text
demo_project/pkg/bufferdecoder/
```

Responsabilita':

- leggere `event_context_t` da 128 byte;
- leggere `argnum`;
- decodificare argomenti a slot fissi;
- produrre `Event` Go serializzabile in JSON.

### Output

Percorso:

```text
demo_project/pkg/output/
```

Responsabilita':

- separare la stampa dal runtime eBPF;
- produrre output JSON line-oriented;
- produrre output `table` compatto per debug manuale;
- convertire campi C-style come `comm` e `uts_name` in stringhe leggibili;
- arricchire capability numeriche con nomi Linux come `CAP_SYS_ADMIN`.

### CLI

Percorsi:

- `demo_project/cmd/project/cmd/root.go`
- `demo_project/pkg/cmd/cobra/cobra.go`
- `demo_project/pkg/cmd/project.go`

Responsabilita':

- parsing flag;
- costruzione config;
- selezione eventi tramite `--events` e `--drop-events`;
- filtro command name tramite `--comms`;
- avvio runner;
- gestione segnali `SIGINT` / `SIGTERM`.

## Stato attuale

Il loader eBPF arriva al runtime loop, legge sia ring buffer sia perf buffer,
decodifica gli eventi raw e li passa a un printer configurabile.

Completato per MVP:

- load dell'oggetto eBPF;
- attach raw tracepoint e kprobe;
- registry selezionabile degli eventi/probe;
- ring buffer reader;
- perf buffer reader;
- decoder Go per context e argomenti attuali;
- output layer separato con formato `json` normalizzato e `table`.
- filtro userspace per `comm`.

Manca ancora:

- detection engine;
- mapping MITRE.
- arricchimento dell'output con mapping di syscall, resource limit e altre
  costanti kernel.

## Collegamenti

- [Timeline](../timeline.md)
- [Userspace lifecycle](userspace-lifecycle.md)
- [Event buffer](event-buffer.md)
- [Decoder Go](decoder.md)
- [Output layer](output.md)
- [Hook implementati](hooks.md)
