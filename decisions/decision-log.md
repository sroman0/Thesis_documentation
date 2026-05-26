# Decision log

Questo file raccoglie le decisioni architetturali importanti. Ogni decisione dovrebbe spiegare contesto, scelta e conseguenze.

## 2026-05-26 - Regola per scegliere tra syscall engine e hook semantici

**Contesto:** durante la scelta del prossimo hook process/security e' emerso il
dubbio se seguire sempre il modello Tracee-like, basato su syscall
`sys_enter`/`sys_exit`, oppure usare hook security piu' specifici come
`security_task_kill`.

**Decisione:** non esiste una scelta unica. La regola del progetto e':

- usare un modello syscall `sys_enter` + `sys_exit` quando il return value e'
  necessario per capire se l'operazione e' riuscita o fallita;
- usare hook semantici `kprobe`/LSM quando il kernel fornisce gia' l'oggetto
  rilevante e il valore principale e' la relazione security, non il risultato
  finale della syscall;
- usare tracepoint quando il kernel espone un evento lifecycle stabile e
  sufficiente.

**Conseguenze:**

- il progetto resta piu' piccolo e target-specific rispetto a Tracee;
- il syscall engine generico resta utile per feature future come `mprotect`,
  `ptrace`, `mount`, `setns`, `chmod` o `chown`;
- hook come `security_bprm_check`, `security_task_fix_setuid` e
  `security_task_kill` vengono trattati come eventi semantici, anche se non
  espongono direttamente il return value della syscall.

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

- i target Makefile principali generano prima l'oggetto BPF quando necessario;
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

**Stato aggiornato al 2026-05-13:** il runtime ora apre anche il perf buffer
`events` con `InitPerfBuf`, quindi il blocco userspace sugli eventi networking
e' stato rimosso. Rimane aperta la decisione architetturale sul canale unico.

## 2026-05-13 - Reader duale per ring buffer e perf buffer

**Contesto:** Tracee usa perf buffer come canale principale. Il progetto aveva
gia' hook process/security su ring buffer e hook networking su perf buffer.
La versione precedente leggeva solo ring buffer, impedendo agli eventi
networking di raggiungere la pipeline Go.

**Decisione:** inizializzare entrambi i reader in `Project.Init()`:

- `InitRingBuf("events_ringbuf", ...)`;
- `InitPerfBuf("events", ..., 1024)`.

Entrambi i canali inviano i byte raw a `handleRawEvent`, che centralizza
decoding, filtro evento, filtro `comm` e output.

**Conseguenze:**

- gli eventi ring buffer e perf buffer attraversano la stessa pipeline
  userspace;
- gli hook networking possono essere testati senza migrare subito il C code;
- la struttura resta temporanea: bisognera' scegliere se mantenere due canali
  oppure convergere verso perf buffer, piu' vicino a Tracee e piu' conservativo
  su Rocky Linux 4.18;
- il perf buffer non ha ancora un lost channel configurato, quindi i drop non
  sono osservabili lato userspace.

## 2026-05-13 - Filtro userspace per `comm`

**Contesto:** durante demo e test manuali, il tool produce eventi globali del
sistema. Per osservare comandi come `ls` o `whoami` serviva un filtro piu'
pratico rispetto al solo `--events`.

**Decisione:** aggiungere `Events.FilterComms` e la flag CLI `--comms`.
Il filtro viene applicato in userspace dopo la decodifica, usando il campo
`comm` dell'`EventContext`.

**Conseguenze:**

- e' piu' semplice mostrare eventi generati da specifici comandi;
- il filtro non riduce ancora il costo lato kernel;
- in futuro si potra' valutare un filtro equivalente in mappa eBPF, piu'
  simile al policy/filtering model di Tracee.

## 2026-05-18 - Docker come ambiente di build riproducibile

**Contesto:** il progetto richiede Go/CGO, clang, LLVM, `pkg-config`, header
`libelf`, `zlib` e build statica di `libbpf`. Installare e mantenere queste
dipendenze manualmente su ogni macchina rende piu' fragile la collaborazione e
la preparazione della demo.

**Decisione:** introdurre Docker come ambiente riproducibile per build e shell
di sviluppo, aggiungendo:

- `demo_project/Dockerfile`;
- `demo_project/.dockerignore`;
- target Makefile `docker-image`, `docker-build`, `docker-shell`,
  `docker-run`;
- documentazione dedicata in `documentation/implementation/docker.md`.

Docker non viene considerato isolamento completo del runtime eBPF: il container
usa comunque il kernel host.

**Conseguenze:**

- la build puo' essere riprodotta con `make docker-build`;
- `make docker-shell` offre una shell con dipendenze gia' installate;
- `make docker-run` esegue il tool in container privilegiato, usando kernel,
  BTF e bpffs dell'host;
- la build Docker usa host networking per evitare problemi DNS su VM;
- il repository viene montato nello stesso path assoluto dell'host per evitare
  path `pkg-config` non validi;
- i limiti del kernel Rocky/RHEL restano invariati: Docker standardizza
  userspace, non il kernel.

## 2026-05-19 - Preferire hook target-specific quando il kernel e' noto

**Contesto:** il progetto deve girare principalmente sulla VM Rocky/RHEL 8.10
con kernel `4.18.0-553.109.1.el8_10.x86_64`. Tracee deve mantenere alta
portabilita' su kernel molto diversi; questo progetto puo' invece sfruttare
il target noto come elemento di semplificazione e ottimizzazione.

**Decisione:** quando esiste un hook specifico e stabile sul kernel target,
preferirlo a un hook generico piu' filtro manuale. Il primo caso applicato e'
`execve`: invece di un secondo `raw_tracepoint/sys_enter` che gira per ogni
syscall, il progetto usa `tracepoint/syscalls/sys_enter_execve`. La stessa
scelta e' stata applicata a `execveat`, usando
`tracepoint/syscalls/sys_enter_execveat`.

`execveat` resta un evento separato da `execve`: non viene trattato come
semplice alias, perche' espone informazioni aggiuntive (`dirfd`, `flags`) utili
per capire exec relative a file descriptor o casi come `AT_EMPTY_PATH`.

**Conseguenze:**

- meno lavoro inutile lato kernel;
- semantica dell'evento piu' chiara;
- registry probe Go esteso con supporto a tracepoint classici;
- payload meno ridondante: `execve` espone il path, `execveat` espone anche
  `dirfd` e `flags`;
- maggiore dipendenza dal kernel target, accettata come parte della novelty
  del progetto;
- resta disponibile `raw_sys_enter` per bookkeeping syscall e casi piu'
  generici.

## 2026-05-19 - Mantenere il path standard per submit eventi

**Contesto:** durante l'ottimizzazione di `execve` e' stato valutato un path
leggero con initializer e submit dedicati. Il path funzionava per l'evento
semplice, ma introduceva una seconda variante poco versatile della pipeline.

**Decisione:** rimuovere il path `_light` e mantenere anche `execve` sul path
standard:

```text
init_program_data -> events_perf_submit
```

**Conseguenze:**

- meno codice speciale da mantenere;
- comportamento piu' uniforme tra eventi;
- costo maggiore rispetto al path leggero, ma piu' facile da spiegare,
  debuggare ed estendere;
- eventuali ottimizzazioni future dovranno essere piu' generali, non legate a
  un solo evento.

## 2026-05-19 - Filtrare i log libbpf in base a `--log-level`

**Contesto:** ogni run stampava molte righe libbpf relative alle relocation
CO-RE, per esempio `field_exists`, `byte_off` e fallback su campi mancanti del
kernel target. Questi log sono utili per debugging ma disturbano l'uso normale
e la demo.

**Decisione:** configurare i callback di logging di `libbpfgo` nel runtime:

- `debug`: mostra tutti i log libbpf;
- `info`: nasconde `LIBBPF_DEBUG`;
- `warn`/`error`: mostra solo warning libbpf.

**Conseguenze:**

- output runtime molto piu' leggibile;
- i dettagli CO-RE restano disponibili quando servono;
- l'help CLI e il README documentano il significato di `--log-level`.

## Collegamenti

- [Timeline](../timeline.md)
- [Event buffer](../implementation/event-buffer.md)
- [Userspace lifecycle](../implementation/userspace-lifecycle.md)
- [Decoder Go](../implementation/decoder.md)
