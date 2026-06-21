La pipeline del tool parte dalla configurazione passata da riga di comando. Questa parte viene gestita dalla CLI in `cmd/project/cmd/root.go` e dalla configurazione in `pkg/config/config.go`. Qui decidiamo quali eventi vogliamo osservare e quale oggetto eBPF utilizzare: può essere un file esterno oppure l’oggetto già incluso nel binario.

Subito dopo viene risolto l’oggetto eBPF da caricare. Questa logica è in `pkg/cmd/initialize/bpfobject.go`: se l’utente specifica un path lo usiamo, altrimenti prendiamo l’oggetto embedded definito in `embedded-ebpf.go`. In questo modo il tool può essere eseguito senza dover passare ogni volta manualmente il file `.bpf.o`.

Successivamente entra in gioco il runtime Go basato su `libbpfgo`, implementato soprattutto in `pkg/ebpf/project.go`. Qui il programma carica l’oggetto eBPF, seleziona solo le probe richieste tramite opzioni come `--events` e `--drop-events`, e poi aggancia gli hook al kernel. La lista e la selezione delle probe sono definite in `pkg/ebpf/probes/probes.go`. Questo è importante perché evita di attivare tutto indiscriminatamente e ci permette di ridurre il rumore già in fase di attach.

A quel punto gli hook sono attivi nel kernel. Quando avviene un evento rilevante, per esempio una `exec`, una `fork`, oppure una chiamata legata ai socket, il programma eBPF in `pkg/ebpf/c/project.bpf.c` raccoglie il contesto e gli argomenti dell’evento. Le strutture condivise e le mappe usate per comunicare con userspace sono nei file C sotto `pkg/ebpf/c`, in particolare `types.h`, `maps.h` e gli header in `pkg/ebpf/c/common`.

Tra gli hook più recenti abbiamo aggiunto `mmap`, `mprotect` e
`pkey_mprotect`, che permettono di osservare creazione e cambio dei permessi
delle regioni di memoria. Abbiamo inoltre aggiunto `process_vm_writev`, utile
per vedere quando un processo tenta di scrivere direttamente nella memoria di
un altro processo. Per questi eventi salviamo gli argomenti all'ingresso della
syscall e produciamo l'evento all'uscita, includendo quindi successo, errno o
numero di byte trasferiti.

Abbiamo poi esteso la parte namespace e filesystem. `setns` e `unshare`
mostrano la richiesta e il risultato della syscall, mentre `switch_task_ns`
conferma quali namespace sono stati realmente sostituiti dal kernel. Gli hook
`security_sb_mount`, `security_sb_umount` e `security_inode_unlink` lavorano
invece su oggetti kernel gia' risolti e producono path, tipo di filesystem,
flag simboliche e metadati inode. Questa parte e' implementata sempre in
`pkg/ebpf/c/project.bpf.c`, con ID e schemi definiti rispettivamente in
`pkg/ebpf/c/types.h`, `pkg/events/ids.go` e `pkg/events/spec.go`.

Gli eventi vengono poi inviati verso userspace principalmente tramite perf buffer. Questa scelta si vede lato C in `pkg/ebpf/c/project.bpf.c`, dove gli hook correnti chiamano `events_perf_submit`, e lato Go in `pkg/ebpf/project.go`, dove viene inizializzato il reader sulla mappa `events`. Manteniamo comunque anche ring buffer e relativo reader perché ci dà versatilità per fallback o confronti futuri, senza obbligarci a riscrivere il runtime.

Infine, lato userspace, il runtime riceve i byte grezzi e li passa al decoder in `pkg/bufferdecoder`. In particolare `decoder.go`, `eventsreader.go` e `protocol.go` trasformano il record binario in una struttura evento leggibile, con contesto, nome evento e argomenti. Dopo la decodifica vengono applicati i filtri userspace, per esempio sul nome evento o sul comando, e l’evento arriva al layer di output.

Il layer di output è in `pkg/output`: `event.go` normalizza la struttura dell’evento, `json.go` produce l’output JSON e `table.go` genera una versione più compatta per la demo da terminale. Quindi possiamo stampare gli eventi in formato `table`, più leggibile durante la presentazione, oppure in formato `json`, più adatto a integrazioni future.

Un altro punto importante è la containerizzazione. Nel `Dockerfile` abbiamo separato tre momenti: uno stage `dev`, pensato per avere una shell con Go, clang, LLVM e le dipendenze utili allo sviluppo; uno stage `builder`, che copia il codice del progetto e compila sia l’oggetto eBPF sia il binario Go; e uno stage `runtime`, più leggero, che contiene solo le librerie necessarie e il binario finale in `/usr/local/bin/project`. Questo rende il progetto più vicino a un deployment reale, perché l’immagine runtime non dipende dal codice sorgente montato dall’esterno: può essere eseguita direttamente, anche in prospettiva Kubernetes, usando comunque il kernel del nodo host per caricare e agganciare i programmi eBPF.

In sintesi, la pipeline mostra che il tool riesce a fare il percorso completo: configurazione, attach degli hook, raccolta kernel-side, trasporto verso userspace, decoding e output leggibile.
