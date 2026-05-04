# Overview implementazione

## Obiettivo tecnico

Il progetto implementa un runtime security monitor basato su eBPF, ispirato a Tracee ma semplificato per il contesto di tesi.

La pipeline desiderata e':

```text
kernel hook eBPF
  -> costruzione evento
  -> ring buffer
  -> userspace Go
  -> decoder
  -> output
  -> detection / alert
```

Stato raggiunto per l'MVP:

```text
kernel hook eBPF
  -> costruzione evento
  -> ring buffer
  -> userspace Go
  -> decoder
  -> output JSON raw
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
- attaccare programmi agli hook;
- leggere eventi;
- decodificare record raw dalla ring buffer;
- stampare eventi JSON raw.

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

### CLI

Percorsi:

- `demo_project/cmd/project/cmd/root.go`
- `demo_project/pkg/cmd/cobra/cobra.go`
- `demo_project/pkg/cmd/project.go`

Responsabilita':

- parsing flag;
- costruzione config;
- avvio runner;
- gestione segnali `SIGINT` / `SIGTERM`.

## Stato attuale

Il loader eBPF arriva al runtime loop, legge la ring buffer, decodifica gli
eventi raw e stampa JSON su stdout.

Completato per MVP:

- load dell'oggetto eBPF;
- attach raw tracepoint e kprobe;
- ring buffer reader;
- decoder Go per context e argomenti attuali;
- output JSON raw.

Manca ancora:

- detection engine;
- mapping MITRE.
- output finale piu' leggibile/stabile;
- filtri per hook molto rumorosi come `cap_capable`.

## Collegamenti

- [Timeline](../timeline.md)
- [Userspace lifecycle](userspace-lifecycle.md)
- [Event buffer](event-buffer.md)
- [Decoder Go](decoder.md)
- [Hook implementati](hooks.md)
