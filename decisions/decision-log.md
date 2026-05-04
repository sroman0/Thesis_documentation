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
- il runtime puo' stampare JSON raw;
- nuovi hook richiedono aggiornamento dello schema in `protocol.go`.

## Collegamenti

- [Timeline](../timeline.md)
- [Event buffer](../implementation/event-buffer.md)
- [Userspace lifecycle](../implementation/userspace-lifecycle.md)
- [Decoder Go](../implementation/decoder.md)
