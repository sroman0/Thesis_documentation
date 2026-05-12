# Decision log

Questo file raccoglie le decisioni architetturali importanti. Ogni decisione dovrebbe spiegare contesto, scelta e conseguenze.

## 2026-04-29 - Usare `cilium/ebpf` invece di `libbpfgo`

**Contesto:** Tracee usa `libbpfgo`, ma il progetto di tesi vuole un MVP piu' leggero in Go.

**Decisione:** usare `github.com/cilium/ebpf`.

**Conseguenze:**

- niente CGO;
- API Go piu' semplice;
- attach e load diversi da Tracee;
- alcune feature Tracee vanno reimplementate manualmente.

**Stato aggiornato al 2026-05-12:** questa decisione e' stata superata dalla
migrazione verso `libbpfgo`, necessaria per allinearsi meglio alla versione
finale del tool e al branch collaboratore con hook networking.

## 2026-05-12 - Migrazione runtime verso `libbpfgo`

**Contesto:** il branch collaboratore e la direzione finale del progetto usano
`libbpfgo`, piu' vicino a Tracee. Il codice basato su `cilium/ebpf` era piu'
semplice, ma avrebbe richiesto adattamenti crescenti per integrare hook e mappe
derivati dal modello Tracee.

**Decisione:** usare `github.com/aquasecurity/libbpfgo` nel runtime Go e
compilare `libbpf` dal submodule `3rdparty/libbpfgo`.

**Conseguenze:**

- il Makefile deve costruire `libbpf.a`;
- i comandi Go richiedono `PKG_CONFIG_PATH`, `CGO_CFLAGS` e `CGO_LDFLAGS`;
- `make build`, `make run`, `make run_table` e `make help` diventano il modo
  consigliato per eseguire il tool;
- l'attach dei probe passa attraverso API `libbpfgo`;
- la build diventa piu' vicina a Tracee, ma meno "pure Go".

## 2026-04-29 - Oggetto eBPF letto da filesystem

**Contesto:** Tracee embedda l'oggetto eBPF, ma per il debugging e' piu' comodo compilare e sostituire `project.bpf.o`.

**Decisione:** leggere l'oggetto da path configurabile.

**Conseguenze:**

- sviluppo piu' rapido;
- CLI deve ricevere `--bpf-object`;
- in futuro si potra' valutare embedding.

## 2026-05-06 - Oggetto eBPF embedded nel binario Go

**Contesto:** Tracee non cerca `tracee.bpf.o` a runtime: lo genera in `dist/`
durante la build e lo include nel binario Go con `go:embed`. Il progetto aveva
ancora bisogno di passare `--bpf-object` o un path via configurazione.

**Decisione:** aggiungere `embedded-ebpf.go` e usare `go:embed` per includere
`dist/project.bpf.o` nel binario. `BPFObjPath` vuoto ora significa "usa
l'oggetto embedded". `--bpf-object` e `PROJECT_BPF_OBJECT` restano disponibili
come override espliciti da filesystem.

**Conseguenze:**

- i target `make build`, `make test` e `make run` generano prima l'oggetto BPF;
- il comando standard non richiede piu' `--bpf-object`;
- il binario contiene i byte dell'oggetto eBPF;
- `dist/project.bpf.o` deve esistere al momento della compilazione Go;
- il comportamento resta compatibile con override manuale del path.

## 2026-04-29 - Attach esplicito con lista statica

**Contesto:** Tracee ha `ProbeGroup`, ma per MVP serve solo un sottoinsieme di hook.

**Decisione:** lista statica di raw tracepoint e kprobe in `pkg/ebpf/project.go`.

**Conseguenze:**

- implementazione semplice;
- meno flessibile di Tracee;
- in futuro conviene introdurre un `ProbeGroup`.

## 2026-04-29 - Slot fissi nel buffer eventi

**Contesto:** il verifier rifiutava offset dinamici su `args_buf`.

**Decisione:** usare slot fissi per scalari e stringhe.

**Conseguenze:**

- load eBPF piu' semplice;
- wire format meno compatto;
- decoder Go dovra' conoscere il formato a slot.

## 2026-05-04 - Decoder MVP custom invece di copia Tracee

**Contesto:** Tracee ha un decoder completo, ma il progetto usa un protocollo
MVP diverso: `event_context_t` da 128 byte e argomenti a slot fissi.

**Decisione:** implementare `demo_project/pkg/bufferdecoder` come decoder
custom ispirato a Tracee, ma adattato al formato del progetto.

**Conseguenze:**

- il decoder legge correttamente il context da 128 byte;
- non include `policies_version` e `matched_policies`;
- decodifica gli argomenti usando lo schema statico degli eventi correnti;
- il runtime puo' produrre eventi decodificati da passare al layer output;
- nuovi hook richiedono aggiornamento dello schema in `protocol.go`.

## 2026-05-05 - Registry probe selezionabile invece di policy manager completo

**Contesto:** dopo l'integrazione del decoder, hook come `cap_capable`
generavano molte righe JSON ravvicinate. Tracee risolve questo problema con
un sistema ricco di eventi, policy, probe group e pipeline, ma copiarlo per
intero sarebbe eccessivo per l'MVP.

**Decisione:** introdurre un registry statico in `pkg/ebpf/probes.go`, dove ogni
evento supportato e' collegato al programma eBPF e all'hook kernel. La CLI puo'
abilitare eventi con `--events` e disabilitarli con `--drop-events`.

**Conseguenze:**

- il runtime attacca solo i probe selezionati;
- `cap_capable` puo' essere escluso senza modificare codice;
- la struttura resta vicina al concetto Tracee di probe/event definitions;
- non viene ancora introdotto un policy manager completo;
- la scelta riduce complessita' e rischio sul kernel Rocky Linux 4.18.

## 2026-05-05 - Output layer separato invece di serializzazione nel runtime

**Contesto:** dopo il decoder, `Project.Run()` serializzava direttamente
`bufferdecoder.Event` con `json.Marshal`. Questo produceva output corretto ma
troppo vicino al formato kernel: `comm` e `uts_name` apparivano come array di
byte, e valori come capability restavano numerici.

**Decisione:** introdurre `pkg/output` con un'interfaccia `Printer`, una factory
per scegliere il formato e due implementazioni iniziali: `json` e `table`.
Il JSON usa una vista normalizzata dell'evento, mentre `table` privilegia la
leggibilita' da terminale.

**Conseguenze:**

- `Project.Run()` non conosce piu' i dettagli di serializzazione;
- il formato JSON e' machine-readable ma piu' leggibile;
- `comm` e `uts_name` sono stringhe;
- le capability vengono arricchite con label come `CAP_SYS_ADMIN`;
- nuovi formati o arricchimenti possono essere aggiunti senza toccare il
  runtime eBPF.

## 2026-05-12 - Canale eventi da uniformare: ring buffer vs perf buffer

**Contesto:** gli hook process/security del progetto inviano eventi tramite
`events_ringbuf_submit` e il runtime Go legge `events_ringbuf`. Gli hook
networking importati dal branch collaboratore usano ancora in gran parte
`events_perf_submit`.

**Decisione attuale:** non forzare subito una sostituzione globale. Mantenere
entrambi i percorsi nel codice C, documentando pero' che il runtime standard
legge solo la ring buffer.

**Conseguenze:**

- gli eventi process/security sono visibili nell'output attuale;
- gli eventi networking possono essere compilati e attaccati, ma quelli inviati
  su perf buffer richiedono lavoro userspace aggiuntivo;
- la prossima decisione tecnica sara' scegliere se aggiungere un perf-buffer
  reader oppure migrare gli hook networking a ring buffer.

## Collegamenti

- [Timeline](../timeline.md)
- [Event buffer](../implementation/event-buffer.md)
- [Userspace lifecycle](../implementation/userspace-lifecycle.md)
- [Decoder Go](../implementation/decoder.md)
