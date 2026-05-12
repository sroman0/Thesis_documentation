# Timeline della tesi

Questo file e' il punto di ingresso principale per tenere traccia del lavoro di tesi. Ogni giorno dovrebbe contenere:

- cosa e' stato fatto;
- quali file sono stati toccati;
- quali problemi sono emersi;
- quali decisioni sono state prese;
- quali link portano alle note tecniche piu' dettagliate.

L'obiettivo non e' scrivere un diario perfetto, ma accumulare materiale grezzo e ordinato per rendere piu' semplice la scrittura finale della tesi.

## Indice dei file collegati

### Diario giornaliero

- [2026-04-29 - Loader eBPF, verifier e struttura documentazione](daily/2026-04-29.md)
- [2026-05-04 - Decoder userspace e output JSON raw](daily/2026-05-04.md)
- [2026-05-05 - Selezione eventi e probe registry Tracee-light](daily/2026-05-05.md)
- [2026-05-06 - Oggetto eBPF embedded nel binario Go](daily/2026-05-06.md)
- [2026-05-12 - Merge libbpfgo, hook networking e output operativo](daily/2026-05-12.md)

### Implementazione

- [Overview implementazione](implementation/overview.md)
- [Userspace Go e lifecycle eBPF](implementation/userspace-lifecycle.md)
- [Protocollo eventi e buffer eBPF](implementation/event-buffer.md)
- [Decoder Go degli eventi eBPF](implementation/decoder.md)
- [Output layer e formati eventi](implementation/output.md)
- [Hook implementati](implementation/hooks.md)

### Debugging

- [Debugging verifier eBPF](debugging/ebpf-verifier.md)
- [Comandi utili](debugging/commands.md)

### Decisioni architetturali

- [Decision log](decisions/decision-log.md)

### Materiale tesi

- [Mappa capitoli tesi](thesis/thesis-outline.md)
- [Spunti per testo tesi](thesis/writing-notes.md)

### Riferimenti e note esistenti

- [Report tecnico corrente](report.md)
- [Domande aperte](Domande.md)
- [Note nuovi hook](04-NuoviHook.md)
- [Contesto operativo repository](CLAUDE.md)

## Timeline

### 2026-04-29

**Tema principale:** passaggio da skeleton userspace a runtime eBPF caricabile, debug del verifier, introduzione di una documentazione strutturata.

**Attivita' svolte:**

- Compilazione di `demo_project/pkg/ebpf/c/project.bpf.c` in `project.bpf.o`.
- Introduzione del runtime eBPF in Go in `demo_project/pkg/ebpf/project.go`.
- Aggiunta del caricamento dell'oggetto eBPF da filesystem.
- Aggiunta dell'attach esplicito di raw tracepoint e kprobe.
- Debug di errori del verifier legati a `args_buf.offset`.
- Modifica di `buffer.h` per usare slot fissi per argomenti scalari e stringhe.
- Verifica che il programma arrivi al loop runtime senza errori di load.
- Chiarimento iniziale: il programma non stampava ancora alert perche' mancavano decoder/output/detection.
- Creazione della struttura documentale principale.

**File tecnici principali:**

- `demo_project/pkg/ebpf/project.go`
- `demo_project/pkg/ebpf/c/common/buffer.h`
- `demo_project/pkg/ebpf/c/common/common.h`
- `demo_project/pkg/ebpf/c/common/arch.h`
- `demo_project/pkg/ebpf/c/common/task.h`
- `demo_project/pkg/cmd/initialize/bpfobject.go`
- `demo_project/pkg/cmd/cobra/cobra.go`
- `demo_project/cmd/project/cmd/root.go`

**Note collegate:**

- [Diario dettagliato del giorno](daily/2026-04-29.md)
- [Debugging verifier eBPF](debugging/ebpf-verifier.md)
- [Protocollo eventi e buffer eBPF](implementation/event-buffer.md)
- [Userspace Go e lifecycle eBPF](implementation/userspace-lifecycle.md)
- [Decoder Go degli eventi eBPF](implementation/decoder.md)

### 2026-05-04

**Tema principale:** integrazione del decoder userspace e primo output JSON raw.

**Attivita' svolte:**

- Aggiunto package `demo_project/pkg/bufferdecoder`.
- Introdotto `protocol.go` con `EventContext` Go da 128 byte.
- Introdotto `decoder.go` con primitive di lettura binaria.
- Introdotto `eventsreader.go` per decodificare eventi completi e argomenti.
- Aggiunti test minimi per argomenti scalari e stringhe.
- Collegato `Project.Run()` al decoder e a output JSON su stdout.
- Verificato output reale da `cap_capable`, con argomento `cap` decodificato.

**File tecnici aggiunti/toccati:**

- `demo_project/pkg/bufferdecoder/protocol.go`
- `demo_project/pkg/bufferdecoder/decoder.go`
- `demo_project/pkg/bufferdecoder/eventsreader.go`
- `demo_project/pkg/bufferdecoder/eventsreader_test.go`
- `demo_project/pkg/ebpf/project.go`

**Note collegate:**

- [Diario dettagliato del giorno](daily/2026-05-04.md)
- [Decoder Go degli eventi eBPF](implementation/decoder.md)
- [Userspace Go e lifecycle eBPF](implementation/userspace-lifecycle.md)
- [Protocollo eventi e buffer eBPF](implementation/event-buffer.md)

**Prossimo passo consigliato:**

Rendere l'output piu' leggibile:

- `comm` e `uts_name` come stringhe;
- capability numeriche convertite in nomi (`CAP_SYS_PTRACE`, ecc.);
- filtro o flag debug per hook rumorosi come `cap_capable`.

### 2026-05-05

**Tema principale:** introduzione di una selezione eventi/probe ispirata a Tracee, ma semplificata per l'MVP e per il kernel Rocky Linux 4.18.

**Attivita' svolte:**

- Aggiunta della configurazione `Events` in `demo_project/pkg/config/config.go`.
- Aggiunte le flag CLI `--events` e `--drop-events`.
- Spostata la lista statica degli hook in un registry unico in `demo_project/pkg/ebpf/probes/probes.go`.
- Introdotta la selezione dei probe da attaccare in base agli eventi richiesti.
- Aggiunto filtro userspace sugli eventi decodificati come controllo difensivo.
- Aggiunti test unitari per default, include/exclude, eventi non supportati e selezione vuota.
- Introdotto `demo_project/pkg/output` come layer separato per la stampa eventi.
- Aggiunti printer `json` e `table`, con test dedicati.
- Normalizzato il JSON di output: `comm` e `uts_name` sono stringhe, non array di byte.
- Aggiunto mapping delle Linux capabilities per eventi `cap_capable` (`21` -> `CAP_SYS_ADMIN`, ecc.).
- Segnalato il testo descrittivo della CLI ancora da rinominare da `Vesuvius` a `Project`.

**File tecnici aggiunti/toccati:**

- `demo_project/pkg/config/config.go`
- `demo_project/pkg/cmd/cobra/cobra.go`
- `demo_project/pkg/ebpf/project.go`
- `demo_project/pkg/ebpf/probes/probes.go`
- `demo_project/pkg/ebpf/probes/probes_test.go`
- `demo_project/pkg/output/printer.go`
- `demo_project/pkg/output/json.go`
- `demo_project/pkg/output/table.go`
- `demo_project/pkg/output/event.go`
- `demo_project/pkg/output/json_test.go`
- `demo_project/pkg/output/table_test.go`
- `demo_project/cmd/project/cmd/root.go`

**Decisione progettuale:**

Tracee usa un sistema piu' completo basato su definizioni evento, probe group e pipeline. Per questo progetto e' stato scelto un registry statico e selezionabile: riduce il rumore di hook come `cap_capable`, mantiene l'attach esplicito e resta piu' prevedibile sul kernel Rocky Linux 4.18.

**Note collegate:**

- [Diario dettagliato del giorno](daily/2026-05-05.md)
- [Userspace Go e lifecycle eBPF](implementation/userspace-lifecycle.md)
- [Output layer e formati eventi](implementation/output.md)
- [Decision log](decisions/decision-log.md)
- [Comandi utili](debugging/commands.md)

### 2026-05-06

**Tema principale:** embedding dell'oggetto eBPF nel binario Go, seguendo il
pattern usato da Tracee.

**Attivita' svolte:**

- Aggiunto `demo_project/embedded-ebpf.go` con `go:embed`.
- Spostato l'output eBPF di default a `demo_project/dist/project.bpf.o`.
- Aggiornato `Makefile`: `build`, `test` e `run` dipendono da `bpf`.
- Aggiornato `initialize.BPFObject()` per usare l'oggetto embedded quando non
  viene passato un path esplicito.
- Lasciati disponibili `--bpf-object` e `PROJECT_BPF_OBJECT` come override.
- Impostato `config.Default().BPFObjPath` a stringa vuota per indicare uso
  dell'embedded object.

**File tecnici aggiunti/toccati:**

- `demo_project/embedded-ebpf.go`
- `demo_project/Makefile`
- `demo_project/pkg/cmd/initialize/bpfobject.go`
- `demo_project/pkg/config/config.go`

**Verifiche:**

- `make bpf`
- `GOCACHE=/tmp/go-build go test ./...`
- `go run ./cmd/project --help`
- `go build -o /tmp/project ./cmd/project`

**Note collegate:**

- [Diario dettagliato del giorno](daily/2026-05-06.md)
- [Decision log](decisions/decision-log.md)
- [Comandi utili](debugging/commands.md)

### 2026-05-12

**Tema principale:** consolidamento post-merge con `libbpfgo`, integrazione
degli hook networking del branch collaboratore e chiarimento dei limiti attuali
tra ring buffer e perf buffer.

**Attivita' svolte:**

- Confermato `cmd/project` come entrypoint ufficiale del tool.
- Aggiornato il Makefile per buildare con `libbpfgo` e `libbpf` statico.
- Aggiunto target `filtered` per test rapidi di `security_socket_connect`.
- Esteso il registry probe con gli hook networking:
  `security_socket_create`, `security_socket_listen`,
  `security_socket_connect`, `security_socket_accept`,
  `security_socket_bind`, `security_socket_setsockopt`,
  `security_socket_recvmsg`, `security_socket_sendmsg`.
- Esteso `protocol.go` con gli schemi di decoding per gli eventi socket
  principali.
- Ripristinata `events_ringbuf_submit` per gli hook che inviano eventi sulla
  ring buffer.
- Corretto l'indice degli argomenti salvati dagli hook process/security, evitando
  errori come `argument index 1 out of range for event cap_capable`.
- Verificato output `table` leggibile per process lifecycle e security hooks.
- Chiarito come testare `security_settime64`, `security_task_prctl` e
  `security_socket_connect`.

**Decisione/nota tecnica:**

Il runtime Go legge attualmente solo `events_ringbuf`, mentre diversi hook
networking importati usano ancora `events_perf_submit`. Per questo motivo gli
hook possono essere compilati e attaccati, ma gli eventi networking su perf
buffer richiedono un prossimo intervento: aggiungere un perf-buffer reader o
migrare quegli hook a `events_ringbuf_submit`.

**File tecnici principali:**

- `demo_project/Makefile`
- `demo_project/pkg/ebpf/project.go`
- `demo_project/pkg/ebpf/probes/probes.go`
- `demo_project/pkg/bufferdecoder/protocol.go`
- `demo_project/pkg/ebpf/c/project.bpf.c`
- `demo_project/pkg/ebpf/c/common/buffer.h`

**Note collegate:**

- [Diario dettagliato del giorno](daily/2026-05-12.md)
- [Hook implementati](implementation/hooks.md)
- [Userspace lifecycle](implementation/userspace-lifecycle.md)
- [Comandi utili](debugging/commands.md)
- [Decision log](decisions/decision-log.md)
