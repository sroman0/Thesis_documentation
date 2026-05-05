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

### Implementazione

- [Overview implementazione](implementation/overview.md)
- [Userspace Go e lifecycle eBPF](implementation/userspace-lifecycle.md)
- [Protocollo eventi e buffer eBPF](implementation/event-buffer.md)
- [Decoder Go degli eventi eBPF](implementation/decoder.md)
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
