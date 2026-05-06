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
