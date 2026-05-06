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

## 2026-04-29 - Oggetto eBPF letto da filesystem

**Contesto:** Tracee embedda l'oggetto eBPF, ma per il debugging e' piu' comodo compilare e sostituire `project.bpf.o`.

**Decisione:** leggere l'oggetto da path configurabile.

**Conseguenze:**

- sviluppo piu' rapido;
- CLI deve ricevere `--bpf-object`;
- in futuro si potra' valutare embedding.

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

## Collegamenti

- [Timeline](../timeline.md)
- [Event buffer](../implementation/event-buffer.md)
- [Userspace lifecycle](../implementation/userspace-lifecycle.md)
- [Decoder Go](../implementation/decoder.md)
