# Overview implementazione

## Obiettivo tecnico

Il progetto implementa un runtime security monitor basato su eBPF, ispirato a Tracee ma semplificato per il contesto di tesi.

La pipeline desiderata e':

```text
kernel hook eBPF
  -> costruzione evento
  -> perf buffer
  -> userspace Go
  -> decoder
  -> output
  -> detection / alert
```

Stato raggiunto per l'MVP:

```text
kernel hook eBPF
  -> kernel UID pre-filter opzionale
  -> costruzione evento
  -> perf buffer
  -> userspace Go
  -> decoder
  -> event selection
  -> comm filter
  -> policy filter
  -> detector engine
  -> output printer
  -> alert printer
```

Stato raggiunto per il layer policy/detector userspace:

```text
policy paths / detector paths
  -> config + runner extensions
  -> policy loader + manager
  -> detector YAML parser
  -> detector runtime
  -> registry
  -> dispatcher
  -> engine
```

I componenti sono testati in isolamento e il wiring MVP e' inserito nel loop
eBPF principale: detector YAML, engine e alert printer sono collegati al runtime
eventi. Il detector engine applica anche un dedup temporale breve sugli alert
ripetuti. Resta fuori solo la correlazione stateful avanzata tra piu' eventi,
che andra' centralizzata nello stesso engine.

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
- aprire perf buffer;
- selezionare quali eventi/probe abilitare;
- risolvere eventi pubblici e probe interni richiesti come dipendenze;
- disabilitare l'autoload dei programmi registrati esclusi dalla selezione
  effettiva;
- configurare il filtro UID kernel-side minimale tramite `config_map`;
- attaccare programmi agli hook selezionati, inclusi kprobe, kretprobe, raw
  tracepoint e tracepoint classici;
- leggere eventi;
- decodificare record raw dal perf buffer;
- applicare filtro eventi e filtro `comm`;
- inviare eventi decodificati al printer configurato.
- inviare eventi decodificati al detector engine quando configurato;
- stampare alert tramite il printer dedicato, anche in modalita'
  `--alerts-only`.

### Decoder

Percorso:

```text
demo_project/pkg/bufferdecoder/
```

Responsabilita':

- leggere `event_context_t` da 136 byte;
- leggere `argnum`;
- decodificare argomenti indicizzati secondo `pkg/events/spec.go`;
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
- stampare alert detector in formato `json` o `table`;
- mantenere separati eventi, alert e log runtime.

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

### Docker workflow

Percorsi:

- `demo_project/Dockerfile`
- `demo_project/.dockerignore`
- `demo_project/Makefile`

Responsabilita':

- fornire un ambiente di build riproducibile con Go, clang, LLVM e librerie
  necessarie a `libbpf`;
- permettere `make docker-build` senza installare manualmente tutte le
  dipendenze sulla macchina;
- offrire una shell di sviluppo con `make docker-shell`;
- permettere una modalita' runtime containerizzata con `make docker-run`,
  ricordando pero' che l'eBPF viene caricato nel kernel host.

## Stato attuale

Il loader eBPF arriva al runtime loop, legge il perf buffer, decodifica gli
eventi raw e li passa a un printer configurabile.

Completato per MVP:

- load dell'oggetto eBPF;
- attach raw tracepoint, tracepoint classici, kprobe e kretprobe;
- registry selezionabile degli eventi/probe;
- perf buffer reader;
- compilazione delle allowlist evento delle policy nella selezione di autoload
  e attach, incluse soltanto le dipendenze interne richieste;
- decoder Go per context e argomenti attuali;
- decoder Go per array di stringhe (`StrArrT`) e array Tracee-like
  null-delimited (`ArgsArrT`);
- output layer separato con formato `json` normalizzato e `table`;
- filtro userspace per `comm`;
- workflow Docker per build, shell e demo runtime;
- config e flag CLI per `--policy`, `--detectors`, `--alerts` e
  `--alerts-output`;
- loader YAML e manager userspace per policy;
- contratto runtime dei detector;
- schema e parser YAML dei detector;
- detector YAML eseguibile su evento singolo;
- registry, dispatcher ed engine detector userspace;
- dedup temporale degli alert ripetuti nel detector engine;
- condizioni YAML compilate al caricamento e pruning periodico dello stato dei
  detector collective;
- primo detector collective MVP con sequenza
  `security_task_fix_setuid -> sched_process_exec`;
- modalita' `--alerts-only` per stampare solo alert senza rimuovere gli eventi
  dalla pipeline interna;
- logging runtime strutturato con zap, separato da eventi e alert;
- detector demo YAML per root execution, file sensibili, cambio effective UID a
  root e inizializzazione moduli kernel;
- eventi `execve` e `execveat` su tracepoint dedicati
  `syscalls/sys_enter_execve` e `syscalls/sys_enter_execveat`;
- payload `argv` per `execve`/`execveat`;
- filtro dei log libbpf/CO-RE tramite `--log-level`;
- eventi `fork`, `vfork` e `clone` con correlazione entry/exit;
- eventi `setuid`, `setgid`, `setreuid`, `setregid`, `setresuid`,
  `setresgid`, `setfsuid`, `setfsgid` e `prlimit64`;
- evento `security_bprm_creds_for_exec` per osservare la fase di preparazione
  delle credenziali nel percorso exec;
- eventi syscall enter/exit per `chmod`, `chown`, `open` e `memfd_create`;
- eventi di gestione memoria `mmap`, `mprotect` e `pkey_mprotect`;
- evento `process_vm_writev` per osservare scritture dirette nella memoria di
  un processo;
- evento `process_vm_readv` per osservare letture dirette dalla memoria di un
  processo;
- evento `commit_creds` con confronto strutturato delle credenziali applicate;
- evento `setns` con tipo di namespace simbolico ed esito finale della
  transizione;
- evento `unshare` con bitmask `CLONE_*` ed esito finale della separazione;
- evento `switch_task_ns` differenziale per osservare i namespace realmente
  sostituiti dal kernel;
- eventi `security_sb_mount` e `security_sb_umount` con target risolti,
  filesystem type e flag simboliche;
- evento `security_inode_unlink` con path, device, inode e ctime del file;
- eventi `security_inode_rename`, `security_inode_symlink` e
  `security_inode_mknod` per rename atomici, link simbolici e file speciali;
- eventi `security_mmap_file` e `security_file_mprotect` per mapping e
  transizioni di protezione della memoria;
- eventi `security_bpf`, `security_bpf_map` e `security_bpf_prog` per syscall
  e oggetti eBPF;
- eventi `security_kernel_read_file` e `security_kernel_post_read_file` per
  file letti dal kernel;
- eventi `security_file_ioctl` e `security_file_permission`;
- eventi cgroup `cgroup_attach_task`, `cgroup_mkdir` e `cgroup_rmdir`;
- eventi `call_usermodehelper` e `do_sigaction`;
- eventi `module_load`, `module_free` e `do_init_module`;
- evento `process_execute_failed` per tentativi falliti di `execve` e
  `execveat`;
- eventi di hardening kernel `proc_create`, `register_kprobe` e
  `kallsyms_lookup_name`;
- correlazione degli argomenti tra entry ed exit tramite `args_map`;
- mapping human-readable per errno, file descriptor, `MFD_*`, `MAP_*` e
  `PROT_*`, namespace `CLONE_NEW*`, flag mount/umount, eventi eBPF,
  permessi file, modalita' usermodehelper, puntatori kernel e byte trasferiti.

Manca ancora:

- una correlazione process-tree persistente completa: oggi `group_by:
  process_tree` correla stesso processo e parent-child usando il context locale
  dell'evento, ma non mantiene un grafo processi globale;
- supporto YAML per operatori piu' espressivi, ad esempio bitmask e range;
- eventuale configurazione esplicita della finestra di dedup alert, se i test
  reali mostrano che il default di 5 secondi e' troppo aggressivo o troppo
  permissivo;
- uso del mapping MITRE anche per selezione policy/detector e report di
  copertura ATT&CK;
- arricchimento dell'output con mapping di syscall, alcune opzioni `prctl`,
  socket option e costanti driver-specific;
- filtri kernel-side minimi dopo la stabilizzazione del layer policy/detector
  userspace.

Nota build: il Makefile ora considera anche gli header `.h` sotto
`pkg/ebpf/c` come dipendenze dell'oggetto eBPF. Questo evita mismatch tra
`types.h`, `ids.go` e `dist/project.bpf.o` quando cambiano ID o layout degli
eventi.

## Collegamenti

- [Timeline](../timeline.md)
- [Userspace lifecycle](userspace-lifecycle.md)
- [Event buffer](event-buffer.md)
- [Roadmap hook process e security](hook-roadmap.md)
- [Decoder Go](decoder.md)
- [Output layer](output.md)
- [Hook implementati](hooks.md)
- [Docker nel progetto](docker.md)
- [Misurazione prestazioni](performance.md)
- [Prossimi step del tool](../next-steps/README.md)
