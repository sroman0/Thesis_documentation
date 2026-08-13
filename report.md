# Report tecnico - Loader eBPF userspace e compilazione `project.bpf.c`

## 1. Obiettivo del lavoro

L'obiettivo operativo e' stato portare il progetto al punto in cui:

1. il programma eBPF `pkg/ebpf/c/project.bpf.c` viene compilato in un oggetto CO-RE;
2. lo userspace Go legge l'oggetto compilato;
3. la collection eBPF viene caricata nel kernel;
4. i programmi eBPF vengono attaccati ai rispettivi hook;
5. il perf buffer `events` viene drenato dallo userspace.

Il lavoro e' ispirato a Tracee, ma adattato a un MVP piu' piccolo. La prima
versione del runtime usava `github.com/cilium/ebpf`; dopo il merge con il
branch networking il runtime e' stato migrato verso
`github.com/aquasecurity/libbpfgo`, piu' vicino alla struttura di Tracee.

## 2. Contesto tecnico

Target principale:

- OS: Rocky Linux 8.10 / RHEL-compatible
- Kernel: `4.18.0-553.137.1.el8_10.x86_64`
- Feature eBPF disponibili/usate sul target: BTF/CO-RE, raw tracepoint,
  tracepoint, kprobe, kretprobe e perf event array
- BTF nativo atteso in: `/sys/kernel/btf/vmlinux`

Il progetto genera l'oggetto eBPF in `dist/project.bpf.o` e lo include nel
binario Go tramite `go:embed`. Il path esplicito resta disponibile per debug
tramite CLI o variabile d'ambiente.

## 3. Startup userspace

### 3.1 Configurazione

La struct `config.Config` contiene ora:

- `BPFObjPath`: path dell'oggetto eBPF compilato;
- `BTFObjPath`: path del file BTF;
- `BPFObjBytes`: contenuto dell'oggetto eBPF letto da disco;
- `Output`;
- `Events`, incluse selezione eventi e filtro `comm`;
- `LogLevel`.

La validazione statica controlla solo valori come output format e log level. La risoluzione dei path avviene in una fase successiva.

### 3.2 Inizializzazione dell'oggetto eBPF

E' stato introdotto `pkg/cmd/initialize/bpfobject.go`, che:

- legge `PROJECT_BPF_OBJECT`, se impostata;
- altrimenti usa il path CLI/config, se fornito;
- se non viene fornito nessun path, usa l'oggetto embedded;
- legge i byte dell'oggetto eBPF;
- risolve il BTF path usando `PROJECT_BTF_FILE`, configurazione o default `/sys/kernel/btf/vmlinux`.

Questo permette di lanciare il programma dalla root del repository con:

```bash
make run
```

oppure, dopo la build:

```bash
sudo ./dist/project
```

Il path esplicito resta supportato per debug.

## 4. Runtime eBPF Go

E' stato introdotto `pkg/ebpf/project.go`, che contiene il ciclo di vita eBPF:

1. `New(cfg)` valida che la config sia pronta;
2. `Init(ctx)` configura autoload, carica la collection e apre il perf buffer;
3. `Run(ctx)` legge il canale finche' il contesto viene cancellato;
4. `Close()` chiude link, perf buffer e modulo eBPF.

### 4.1 Rimozione del limite memlock

Prima del load della collection viene chiamato:

```go
rlimit.RemoveMemlock()
```

Questo evita errori come:

```text
map create: operation not permitted (MEMLOCK may be too low)
```

Nota: questa chiamata richiede privilegi adeguati. Nella sandbox Codex fallisce per mancanza di capability, mentre sulla VM va eseguita con `sudo`.

### 4.2 Attach esplicito dei programmi

Un punto fondamentale: caricare una collection eBPF non basta. I programmi devono essere attaccati ai rispettivi hook.

Gli attach passano dal registry in `pkg/ebpf/probes/probes.go`. Ogni voce
associa:

- nome evento esposto alla CLI;
- nome del programma eBPF dentro `project.bpf.o`;
- hook kernel;
- strategia di attach.

Il registry supporta oggi:

| Tipo probe | Uso nel progetto |
|---|---|
| raw tracepoint | lifecycle processi, cgroup, moduli, raw syscall |
| tracepoint | syscall enter/exit con return value |
| kprobe | hook security/LSM-like, VFS, memoria, BPF, moduli |
| kretprobe | eventi in cui serve il valore di ritorno della funzione kernel |

La copertura attuale include gruppi diversi:

- process lifecycle: `sched_process_fork`, `sched_process_exec`,
  `sched_process_exit`, `task_rename`;
- exec e credenziali: `execve`, `execveat`, `process_execute_failed`,
  `security_bprm_check`, `security_bprm_creds_for_exec`, `commit_creds`;
- creazione processi e privilege changes: `fork`, `vfork`, `clone`,
  `setuid`, `setgid`, `setreuid`, `setregid`, `setresuid`, `setresgid`,
  `setfsuid`, `setfsgid`, `security_task_fix_setuid`;
- file e filesystem: `open`, `chmod`, `chown`, `security_file_open`,
  `security_file_permission`, `security_file_ioctl`,
  `security_inode_unlink`, `security_inode_rename`,
  `security_inode_symlink`, `security_inode_mknod`;
- memoria e process injection surface: `mmap`, `mprotect`, `pkey_mprotect`,
  `security_mmap_file`, `security_file_mprotect`, `memfd_create`,
  `process_vm_readv`, `process_vm_writev`;
- namespace e mount: `setns`, `unshare`, `switch_task_ns`,
  `security_sb_mount`, `security_sb_umount`;
- eBPF/kernel/module hardening: `security_bpf`, `security_bpf_map`,
  `security_bpf_prog`, `security_kernel_read_file`,
  `security_kernel_post_read_file`, `module_load`, `module_free`,
  `do_init_module`, `proc_create`, `register_kprobe`,
  `kallsyms_lookup_name`;
- cgroup e kernel helper: `cgroup_attach_task`, `cgroup_mkdir`,
  `cgroup_rmdir`, `call_usermodehelper`;
- signal/security operations: `security_task_kill`, `do_sigaction`,
  `security_task_prctl`, `security_task_setrlimit`, `security_settime64`,
  `cap_capable`;
- networking security hooks: `security_socket_create`,
  `security_socket_listen`, `security_socket_connect`,
  `security_socket_accept`, `security_socket_bind`,
  `security_socket_setsockopt`; `security_socket_recvmsg` e
  `security_socket_sendmsg` restano probe interni di attribuzione per gli
  eventi cgroup e non sono eventi pubblici autonomi.

Gli handle sono gestiti tramite `libbpfgo`. La selezione `--events` evita di
attaccare programmi non richiesti, quindi il costo runtime dipende anche dal
set di eventi scelto.

## 5. Confronto con Tracee

### 5.1 Pattern rispettati

Tracee definisce i probe in `pkg/ebpf/probes/probe_group.go` e poi li attacca in `pkg/ebpf/probes/trace.go`.

Pattern equivalente:

- Tracee registra coppie evento-programma;
- il nostro MVP registra coppie hook-programma in slice statiche;
- Tracee conserva gli handle dei link;
- il nostro MVP conserva `[]link.Link`;
- Tracee distrugge i link in detach;
- il nostro MVP chiude i link in `Close()`.

Esempi Tracee:

- `SchedProcessFork`: raw tracepoint `sched:sched_process_fork`
- `SchedProcessExec`: raw tracepoint `sched:sched_process_exec`
- `SchedProcessExit`: raw tracepoint `sched:sched_process_exit`
- `TaskRename`: raw tracepoint `task:task_rename`
- `CapCapable`: kprobe `cap_capable`
- `SecurityTaskSetrlimit`: kprobe `security_task_setrlimit`
- `SecuritySettime64`: kprobe `security_settime64`

Il nostro codice replica lo stesso concetto, ma senza introdurre ancora un sistema completo di `ProbeGroup`.
In piu', per il target Rocky/RHEL noto, usa tracepoint syscall dedicati per
`execve` ed `execveat`, evitando un secondo dispatcher generico su
`raw_tracepoint/sys_enter`.

### 5.2 Differenze importanti

La differenza iniziale era che Tracee usava `libbpfgo`, mentre il progetto
usava `cilium/ebpf`. Questa distanza e' stata ridotta con la migrazione del
runtime verso `libbpfgo`. Restano comunque differenze importanti:

- Tracee ha una pipeline eventi e policy piu' completa;
- Tracee gestisce i kprobe tramite symbol table e puo' attaccarsi per address se un simbolo ha piu' indirizzi;
- il progetto usa ancora un registry statico piu' semplice;
- il progetto usa il perf buffer come unico transport degli eventi.

Tracee usa inoltre un'infrastruttura molto piu' ampia:

- autoload per probe;
- compatibilita' kernel;
- tail calls;
- policy/filtering;
- metriche;
- gestione eventi completa;
- decoder userspace.

Il progetto mantiene solo il necessario per un MVP.

## 6. Compilazione di `project.bpf.c`

Comando usato:

```bash
clang -g -O2 -target bpf -D__TARGET_ARCH_x86 -I pkg/ebpf/c \
  -c pkg/ebpf/c/project.bpf.c -o pkg/ebpf/c/project.bpf.o
```

Output atteso:

```text
pkg/ebpf/c/project.bpf.o
```

Al momento l'oggetto viene generato correttamente.

## 7. Problemi incontrati in compilazione C

### 7.1 `bpf_task_pt_regs` non dichiarata

Errore iniziale:

```text
call to undeclared function 'bpf_task_pt_regs'
```

Tracee prova a usare `bpf_task_pt_regs()` quando disponibile, con fallback manuale. Sulla toolchain della VM, l'enum helper esiste nel BTF/header, ma la funzione non e' dichiarata dagli header libbpf installati.

Scelta fatta per MVP:

- rimuovere il path `bpf_task_pt_regs()`;
- usare sempre il fallback manuale basato su `task->stack + THREAD_SIZE`.

Questa scelta e' meno portabile di Tracee, ma adatta al target Rocky/RHEL del progetto.

### 7.2 Firma di `bpf_core_field_exists`

Errore iniziale:

```text
too many arguments provided to function-like macro invocation
```

Il codice usava:

```c
bpf_core_field_exists(struct task_struct, start_boottime)
```

La macro installata sulla VM accetta un solo argomento. Il codice e' stato adattato a:

```c
bpf_core_field_exists(task->start_boottime)
```

La semantica resta la stessa: controllare se il campo `start_boottime` esiste nel `task_struct` del kernel target.

### 7.3 `barrier()` non definita

Errore iniziale:

```text
call to undeclared function 'barrier'
```

E' stata aggiunta in `common/common.h`:

```c
#ifndef barrier
#define barrier() asm volatile("" ::: "memory")
#endif
```

## 8. Errori del verifier eBPF

Dopo la compilazione, il caricamento nel kernel ha evidenziato diversi errori del verifier.

### 8.1 Errore su `tracepoint__sched__sched_process_exec`

Errore:

```text
invalid access to map value, value_size=4280 off=65664 size=1
```

Interpretazione:

- il verifier vedeva `buf->offset` come `u16` potenzialmente fino a `65535`;
- l'accesso a `buf->args[buf->offset]` poteva quindi uscire dai limiti della scratch map;
- il valore `65664` deriva dall'offset base dentro `event_data_t` piu' un possibile `65535`.

Mitigazione:

- leggere `buf->offset` in una variabile locale `off`;
- controllare `off` contro `ARGS_BUF_SIZE`;
- usare `off` per indicizzare il buffer.

### 8.2 Errore su `trace_cap_capable`

Errore:

```text
invalid access to map value, value_size=4280 off=65664 size=4
```

Questo errore avveniva nel path `save_to_submit_buf`, usato per argomenti scalari.

Mitigazione:

- sostituire `bpf_probe_read` con copie a dimensione costante via `__builtin_memcpy`;
- gestire esplicitamente dimensioni supportate:
  - `u8`
  - `u16`
  - `u32`
  - `u64`

Questo aiuta il verifier perche' la dimensione della copia diventa costante in ogni branch.

### 8.3 Errore su `tracepoint__task__task_rename`

Errore:

```text
invalid access to map value, value_size=4280 off=65664 size=4096
```

Questo errore riguardava `save_str_to_buf`: il verifier vedeva una possibile scrittura di `4096` byte a offset variabile.

Mitigazione:

- introdotto `MAX_ARG_STRING_SIZE = 512`;
- limitata la dimensione effettiva passata a `bpf_probe_read_str`;
- mantenuti i controlli su `off` e sullo spazio residuo.

Trade-off:

- path/nomi lunghi possono essere troncati a 512 byte;
- per il MVP e' accettabile;
- in futuro si puo' reintrodurre un path buffer dedicato, come fa Tracee, oppure usare una strategia a chunk.

### 8.4 Errore su `tracepoint__sched__sched_process_exit`

Errore:

```text
invalid access to map value, value_size=4280 off=65664 size=1
```

Questo errore e' emerso dopo le prime mitigazioni sui writer degli argomenti. La causa piu' probabile non e' piu' solo la scrittura dentro `args`, ma anche la size passata a:

```c
bpf_ringbuf_output(&events, event, size, 0)
```

In `events_ringbuf_submit()` la size era calcolata a partire da:

```c
event->args_buf.offset
```

`offset` e' un `u16`, quindi il verifier lo considera potenzialmente fino a `65535`. Anche se a runtime il codice prova a mantenerlo entro `ARGS_BUF_SIZE`, il verifier deve poterlo dimostrare localmente nel punto della chiamata alla helper. In caso contrario vede una possibile copia dalla scratch map fino a offset `65664`, fuori da `value_size=4280`.

Mitigazione applicata:

- leggere `event->args_buf.offset` in `args_off`;
- controllare esplicitamente `args_off <= ARGS_BUF_SIZE`;
- calcolare `size` usando `args_off`;
- controllare esplicitamente `size <= MAX_EVENT_SIZE`;
- rimuovere il clamp `update_min(size, MAX_EVENT_SIZE)` in questo punto, preferendo branch espliciti piu' leggibili per il verifier.

Nuovo schema:

```c
u32 args_off = event->args_buf.offset;
if (args_off > ARGS_BUF_SIZE)
    return 0;

u32 size = sizeof(event_context_t) + sizeof(u8) + args_off;
if (size > MAX_EVENT_SIZE)
    return 0;
```

Questa modifica segue lo stesso principio delle mitigazioni precedenti: rendere i limiti locali e immediatamente visibili al verifier prima di ogni accesso/copia su memoria eBPF.

### 8.5 Secondo errore su `tracepoint__task__task_rename`

Errore:

```text
invalid access to map value, value_size=4280 off=65664 size=512
```

Anche dopo aver ridotto la size massima delle stringhe da 4096 a 512 byte, il verifier continuava a rifiutare `bpf_probe_read_str()`. Il problema residuo era la destinazione:

```c
&(buf->args[off + 1 + sizeof(u32)])
```

`off` derivava ancora dallo stato dinamico del buffer. Anche se il codice controllava `off`, il verifier non riusciva a dimostrare in modo stabile che la destinazione della helper restasse sempre dentro la `scratch_map`.

Mitigazione applicata:

- introdotti slot stringa a offset costante;
- ogni stringa occupa uno slot fisso:

```c
1 byte type tag + 4 byte length + 512 byte payload
```

- massimo iniziale: `MAX_STRING_ARGS = 4`;
- `save_str_to_buf()` sceglie lo slot usando `argnum`:
  - argomento 0 -> offset `0`;
  - argomento 1 -> offset `STRING_ARG_SLOT_SIZE`;
  - argomento 2 -> offset `STRING_ARG_SLOT_SIZE * 2`;
  - argomento 3 -> offset `STRING_ARG_SLOT_SIZE * 3`.

Trade-off:

- il wire format delle stringhe non e' piu' completamente compatto: ogni stringa riserva 512 byte anche se la stringa reale e' piu' corta;
- il campo length resta presente e indica quanti byte sono realmente significativi;
- per il MVP questo e' accettabile per sbloccare il verifier;
- quando verra' implementato il decoder Go, dovra' sapere che gli argomenti stringa usano slot fissi oppure il formato dovra' essere riportato a una serializzazione compatta con una strategia verifier-safe piu' raffinata.

Nota sullo stato corrente: questa mitigazione descrive una fase storica del
debug verifier. La pipeline attuale non usa piu' slot fissi come formato
principale degli argomenti; usa record indicizzati compatti
`[index][payload]`, decodificati tramite `pkg/events/spec.go`.

### 8.6 Secondo errore su `tracepoint__sched__sched_process_exit`

Errore:

```text
invalid access to map value, value_size=4280 off=65664 size=1
```

Questo errore e' ricomparso su `sched_process_exit` dopo la modifica agli slot stringa. L'evento `sched_process_exit` serializza argomenti scalari:

- `exit_code` come `s64`;
- `group_dead` come `u8`.

Il rifiuto sul secondo argomento (`size=1`) indica che anche `save_to_submit_buf()` era ancora troppo dinamica per il verifier. Nonostante i controlli su `off`, l'offset effettivo continuava a derivare da `buf->offset`, campo residente nella scratch map e considerato potenzialmente fino a `65535`.

Mitigazione applicata:

- anche gli argomenti scalari vengono ora scritti in slot fissi;
- ogni slot scalare occupa `16` byte;
- massimo iniziale: `MAX_SCALAR_ARGS = 8`;
- lo slot viene scelto tramite `argnum`, con offset costanti:
  - argomento 0 -> `0`;
  - argomento 1 -> `16`;
  - argomento 2 -> `32`;
  - ecc.

Il writer continua a salvare:

- `type_tag` nel primo byte dello slot;
- valore scalare subito dopo il tag;
- padding inutilizzato nel resto dello slot.

Trade-off:

- il formato wire diventa meno compatto anche per gli scalari;
- ogni argomento scalare occupa sempre 16 byte, anche se il valore e' un `u8`;
- per il MVP questo e' accettabile per rendere gli offset dimostrabili dal verifier;
- il decoder Go dovra' conoscere questa forma a slot fissi oppure verra' riprogettato quando il protocollo sara' stabilizzato.

Nota sullo stato corrente: anche per gli scalari il formato operativo e' ora
indicizzato e compatto. Le sezioni sugli slot fissi restano utili per spiegare
il percorso di debug del verifier, ma non descrivono piu' il wire format finale
usato dal decoder.

## 9. Stato attuale

Verifiche tipiche:

```bash
make bpf
```

Risultato: compilazione riuscita.

```bash
make build
```

Risultato: build Go riuscita con `libbpfgo` e oggetto eBPF in `dist/`.

Il comando da eseguire sulla VM e':

```bash
cd /home/simone/project/demo_project
make run
```

Esempi di output osservabili oggi:

```text
event=execve ... args=pathname=/usr/bin/whoami,argv=["whoami"]
event=open ... args=operation=openat(2),pathname=/etc/hostname,returnValue=fd:3
event=security_bprm_check ... args=pathname=/usr/bin/bash,dev=...,inode=...,argc=...
event=register_kprobe ... args=symbol_name=cap_capable,pre_handler=0x...,returnValue=success
```

Questo indica che il programma ha superato stabilmente le fasi che
inizialmente erano bloccanti:

- lettura dell'oggetto eBPF embedded o esterno;
- load della collection;
- apertura del perf buffer;
- attach selettivo dei programmi eBPF;
- ingresso nel loop runtime;
- decode userspace;
- output `table`/`json`;
- mapping human-readable di molti valori kernel.

Il runtime eBPF puo' ora stampare anche alert quando il layer detector e'
configurato. Il runtime riceve i record dal perf buffer. Dopo il decode applica
selezione eventi, filtro `comm`, policy userspace, output evento, detector
engine e output alert.

Loop aggiornato in `pkg/ebpf/project.go`:

```go
select {
case raw := <-p.perfBufChannel:
    p.handleRawEvent(ctx, raw)
}
```

Ogni evento ricevuto viene decodificato e viene emessa una riga nel formato
configurato (`json` o `table`). Tutti gli hook usano `events_perf_submit`.

Conclusione dello stato corrente:

- il loader eBPF e' arrivato a uno stato operativo;
- la pipeline userspace degli eventi e' presente;
- gli hook producono eventi decodificati e stampabili;
- il runtime riceve eventi dal perf buffer;
- l'output e' separato dal runtime eBPF;
- la copertura process/security e' ampia per una demo tecnica;
- policy manager, detector engine e output alert sono collegati alla pipeline
  eventi e possono produrre alert da detector YAML.
- le allowlist evento esplicite delle policy vengono compilate nella selezione
  di autoload e attach, evitando di caricare i programmi registrati esclusi e
  mantenendo soltanto le dipendenze interne richieste;
- policy e detector YAML vengono validati contro il registry eventi del
  decoder prima dell'avvio eBPF, evitando configurazioni che referenziano eventi
  inesistenti o non decodificabili.
- l'evento userspace espone helper `Arg`, `ArgString`, `ArgInt64`, `ArgUint64`
  e `ArgBool`, usati dai detector per leggere argomenti decodificati in modo
  uniforme.
- il set demo include detector YAML mappati a MITRE per root execution, accesso
  a file sensibili, cambio effective UID a root e inizializzazione di moduli
  kernel.

Test runtime osservato:

```text
type=alert alert=Sensitive system file opened severity=medium detector=sensitive-file-open events=1 detector_name=Sensitive file open source_event=security_file_open source_pid=... source_uid=... source_comm=cat source_args=pathname=/etc/passwd,...
event=security_file_open ... args=pathname=/etc/passwd,...
```

Questo mostra il funzionamento end-to-end: evento kernel, decode userspace,
policy selection, dispatcher detector, match YAML e output alert. Il limite
principale emerso non era funzionale ma di leggibilita': eventi raw e alert
erano stampati nello stesso stream. Questo punto e' stato mitigato con
`--alerts-only`, che stampa solo gli alert lasciando comunque gli eventi nella
pipeline interna per policy e detector.

Il detector `sensitive-file-open` e' stato poi reso piu' stringente: invece di
fare match su tutto il prefisso `/etc/`, usa l'operatore YAML `in` su una lista
esplicita di file critici (`/etc/passwd`, `/etc/shadow`, `/etc/sudoers`,
`/etc/ssh/sshd_config`, `/root/.ssh/authorized_keys`). Questo riduce alert
rumorosi prodotti da aperture lecite e frequenti come `/etc/hosts` e
`/etc/ld.so.cache`.

Successivamente i detector demo piu' rumorosi sono stati raffinati con filtri
espliciti sugli helper legittimi `sudo` e `unix_chkpwd`. Questo vale per
`root-exec`, `sensitive-file-open`, `privileged-uid-change`,
`privilege-exec-chain` e `privilege-sensitive-file-chain`. L'obiettivo e'
ottenere alert piu' leggibili per demo e threat hunting locale, lasciando
eventuali policy piu' conservative a configurazioni YAML dedicate.

Inoltre il detector engine applica un dedup temporale breve sugli alert
ripetuti. La finestra default e' 5 secondi e la chiave include detector, titolo,
severita', evento sorgente, pid, uid, comm e argomenti sorgente. Gli eventi
continuano a essere processati, ma gli alert identici prodotti nella stessa
finestra vengono soppressi. La metrica `AlertsSuppressed` rende osservabile
questa riduzione del rumore.

Il primo detector collective e' stato aggiunto con
`privilege-exec-chain`. La sequenza osservata e':

```text
security_task_fix_setuid(new_euid=0)
  -> sched_process_exec(uid=0)
```

entro 5 secondi. L'alert contiene entrambi gli eventi sorgente, quindi non
descrive piu' solo un evento puntuale ma una breve catena comportamentale.
Il detector inizialmente usava `group_by: global`, utile per test rapidi ma
troppo permissivo. La versione corrente usa `group_by: process_tree`, che
correla eventi dello stesso processo o di una relazione parent-child usando
`host_pid`, `host_ppid`, `start_time` e `parent_start_time`. La correlazione
resta locale e corta: non richiede contesto Kubernetes o conoscenza globale del
cluster.

Sono stati aggiunti altri tre detector collective:

- `privilege-sensitive-file-chain`: cambio privilegi seguito da apertura root di
  file sensibili come `/etc/passwd`, `/etc/shadow`,
  `/etc/ssh/sshd_config` o `/root/.ssh/authorized_keys`. La catena esclude
  helper legittimi come `sudo` e `unix_chkpwd` per ridurre rumore;
- `memfd-exec-chain`: `memfd_create` riuscita seguita da exec da path memfd;
- `kernel-module-kprobe-chain`: inizializzazione modulo kernel seguita da
  registrazione kprobe dinamica.

Il motore collective e' stato successivamente generalizzato. `group_by` non
indica piu' un campo arbitrario, ma una lista di strategie compilate:
`process`, `process_tree`, `resource` e `cgroup`. Le strategie possono essere
composte e tutti i componenti devono coincidere. `resource` usa esclusivamente
`dev + inode`, quindi puo' correlare processi differenti senza usare pathname
come identita'.

Il detector `temp-script-write-exec` dimostra questo percorso correlando
`security_file_permission` e `security_bprm_check` sullo stesso file
temporaneo. Lo stato resta locale al nodo, con finestra massima di cinque
secondi, pruning periodico e limite di 4096 sequenze incomplete per detector.
`user_session` non e' implementata perche' il context non contiene un session
ID affidabile e UID non viene trattato come sessione.

Il mapping MITRE ATT&CK dichiarato nei detector YAML viene ora propagato fino
all'output alert. Nel formato table viene mostrata una sintesi compatta, ad
esempio:

```text
mitre=TA0004|TA0007|T1548|T1083
```

Nel formato JSON viene invece esposto un blocco strutturato `threat` con
framework, versione, tattiche, tecniche, data source e data component. Questa
scelta rende l'output leggibile in demo, ma anche integrabile in sistemi SOC,
SIEM o reportistica tecnica.

Le policy possono inoltre usare questi metadata per restringere il catalogo
passato con `--detectors`. La sezione `detectors` supporta ID, severity,
tactic, technique, tag e stato stateful. Il filtro viene compilato durante il
bootstrap, prima della costruzione di registry, dispatcher e stato collective.
I detector esclusi non vengono istanziati.

Le liste della stessa proprietà hanno semantica OR; proprietà differenti hanno
semantica AND. `exclude` prevale su `include` all'interno della singola policy,
mentre più policy attive compongono i detector scelti tramite unione. Le policy
prive della sezione mantengono il comportamento precedente.

Gli ID espliciti vengono verificati contro il catalogo ricavato dai detector
YAML forniti alla CLI. Se una policy contiene un selettore ma non viene passato
`--detectors`, le regole evento restano operative e soltanto la selezione
detector viene saltata con un warning. Se invece esiste un catalogo ma un ID
richiesto non è presente, il bootstrap fallisce per intercettare typo o
deployment incompleti.

Per rendere i test meno ambigui, la directory `rules/policies` e' stata poi
estesa con policy preset per scenario:

```text
demo-detectors.yaml
full-demo.yaml
collective-privilege-exec.yaml
collective-local-chains.yaml
mitre-privilege-collective.yaml
point-process-security.yaml
sensitive-file-access.yaml
kernel-module-activity.yaml
```

Questa organizzazione permette di avviare solo il detector o la famiglia di
detector necessari alla demo. Per esempio, il collective detector puo' essere
provato con `collective-privilege-exec.yaml` e
`rules/detectors/privilege_exec_chain.yaml`, senza caricare anche detector su
file sensibili o moduli kernel.

## 10. Limitazioni attuali

### 10.1 Decoder MVP

Il runtime ora decodifica gli eventi con il package:

```text
demo_project/pkg/bufferdecoder/
```

Il decoder implementa:

- parsing di `event_context_t` da 136 byte;
- parsing di `argnum`;
- parsing degli argomenti indicizzati (`INT_T`, `UINT_T`, `STR_T`, ecc.);
- produzione di `bufferdecoder.Event`, poi passato al layer `pkg/output`.

Esempio JSON normalizzato:

```json
{"timestamp":4074609472044753,"event_name":"cap_capable","process":{"pid":1374462,"tid":1374462,"ppid":1348641,"uid":1000,"comm":"cpuUsage.sh","uts_name":"security-thesis"},"host":{"pid":1374462,"tid":1374462,"ppid":1348641},"kernel":{"syscall":56,"processor_id":0,"mnt_id":4026531840,"pid_id":4026531836},"args":[{"name":"cap","type":1,"value":21,"label":"CAP_SYS_ADMIN"}]}
```

Limitazioni residue:

- alcune opzioni `prctl`, socket option e comandi driver-specific sono ancora
  numerici;
- nuovi hook richiedono un ID coerente tra C e Go, una voce nello schema
  statico in `pkg/events/spec.go` e la registrazione della probe;
- il policy manager e il detector engine sono collegati al loop eBPF, ma la
  correlazione stateful multi-evento e' ancora MVP e supporta solo sequenze
  ordinate corte;
- il dedup alert e' presente ma non ancora configurabile da CLI;
- le correlazioni collective supportano chiavi locali di processo, parent
  immediato, risorsa e cgroup, ma non mantengono un albero processi completo o
  persistente e non correlano contesto di cluster;
- il mapping MITRE è visibile nell'output alert e può selezionare detector
  all'avvio; manca ancora una vista automatica di coverage ATT&CK e la
  validazione semantica rispetto a una release ufficiale.

Il decoder supporta inoltre array di stringhe, payload NUL-delimited,
sockaddr, credenziali strutturate e puntatori. Gli eventi
namespace/filesystem/kernel-hardening aggiunti piu' di recente usano gli stessi
record indicizzati:

- `switch_task_ns` espone soltanto i namespace realmente cambiati;
- `security_sb_mount` e `security_sb_umount` espongono mountpoint, filesystem
  type e flag;
- `security_inode_unlink` espone path, inode, device e ctime.
- `proc_create`, `register_kprobe` e `kallsyms_lookup_name` espongono nomi,
  puntatori kernel e return value quando necessario.

Un problema recente di debug ha confermato l'importanza dell'allineamento
binario tra C e Go. Gli eventi venivano ricevuti e riconosciuti correttamente,
ma l'output mostrava `args=-` perche' il decoder Go avanzava ancora di 128 byte
dopo il context, mentre il context reale era diventato di 136 byte con
`policies_version` e `matched_policies`. Il fix e' stato aggiornare
`eventContextSize` a 136 in `pkg/bufferdecoder/protocol.go`. Da questo punto in
poi ogni estensione di `event_context_t` deve essere accompagnata da una
modifica esplicita del decoder.

### 10.2 Registry e attach delle probe

Il runtime usa `libbpfgo` e il registry in:

```text
demo_project/pkg/ebpf/probes/probes.go
```

Ogni voce associa:

- nome evento esposto dalla CLI;
- nome del programma nell'oggetto eBPF;
- hook kernel;
- tipo di attach: raw tracepoint, tracepoint, kprobe o kretprobe.

Non tutte le voci del registry sono eventi pubblici. Alcuni programmi eBPF
servono solo come probe interni, per esempio per aggiornare mappe kernel-side o
per mantenere correlazioni di supporto. Questi probe restano nel registry, ma
non sono mostrati da `--list-events` e non sono selezionabili con `--events`.
La regola operativa e': un probe puo' essere pubblico solo se esiste una
specifica decoder corrispondente in `pkg/events/spec.go`.

Il campo `internal` definisce la visibilita' pubblica, mentre `impliedBy`
definisce la dipendenza di attach. La selezione restituisce quindi separatamente
il set degli eventi esposti e l'elenco dei programmi da caricare: un probe
interno puo' essere necessario nel secondo senza comparire nel primo.

I probe networking di attribuzione, tra cui `sock_alloc_file`,
`security_sk_clone`, gli helper `security_socket_recvmsg/sendmsg` e
`cgroup_bpf_run_filter_skb`, sono classificati contemporaneamente come
`internal` e `impliedBy` dagli eventi cgroup. Non producono record normali
autonomi, ma vengono caricati quando servono a costruire il contesto degli
eventi pubblici.

Il contratto ora e' coperto anche da test automatici:

- ogni probe pubblico deve avere uno schema decoder;
- ogni evento presente nel decoder deve essere esposto da almeno un probe
  pubblico;
- ogni schema decoder deve avere ID e nome coerenti con il registry
  `pkg/events`.

Il target dedicato e':

```bash
make test-registry
```

La selezione con `--events` evita di attaccare programmi non richiesti. Il
limite ancora presente e' che la compatibilita' del simbolo kprobe viene
verificata principalmente durante l'attach. Un miglioramento futuro potra'
aggiungere discovery preventiva dei simboli e fallback espliciti per kernel
diversi dal target Rocky Linux.

### 10.3 Stringhe e payload voluminosi

Le stringhe serializzate restano limitate lato protocollo per mantenere il
verifier sotto controllo e impedire payload eccessivi.

Per path e `argv` questa scelta e' accettabile. `envp` non viene raccolto di
default perche' e' rumoroso e puo' contenere dati sensibili.

### 10.4 Selezione eventi ancora userspace/attach-time

La CLI permette di usare `--events` e `--drop-events`. Inoltre e' stato
introdotto un primo policy manager userspace, capace di valutare policy caricate
da YAML su nome evento, `comm` e `uid`.

La selezione attuale riduce il numero di programmi attaccati e prepara il
filtering dichiarativo. E' stato aggiunto anche un primo filtro kernel-side
minimale su UID, configurato tramite `config_map` e applicato in
`init_program_data()` prima della costruzione dell'evento. Restano invece da
implementare filtri kernel-side piu' ricchi per PID, namespace, container,
liste UID multiple e `comm`.

### 10.5 Output arricchito ma non completo

Il runtime stampa eventi JSON o table decodificati. L'output e' piu' leggibile
rispetto al JSON raw iniziale, ma resta pensato per debugging:

- una riga per evento;
- filtri disponibili per include/exclude eventi e `comm`;
- enrichment presente per capability, segnali, errno, memoria, namespace,
  mount/umount, BPF command/type, kernel-read-file type, file permission mask,
  usermodehelper wait mode, signal handler mode e puntatori kernel;
- mapping ancora mancante o parziale per alcune opzioni `prctl`, socket
  option, syscall e valori driver-specific.

## 11. Prossimi step consigliati

La roadmap dettagliata e' ora raccolta in
[next-steps/](next-steps/README.md). I punti principali sono:

1. mantenere output separato e leggibile per eventi e alert;
2. usare `--alerts-only` per demo e run detector-focused, in modo da stampare
   solo gli alert senza perdere gli eventi necessari ai detector;
3. mantenere logging runtime strutturato con `zap`, separato da eventi e
   alert;
4. rendere configurabile il dedup alert solo se i test reali mostrano che il
   default di 5 secondi non e' adeguato;
5. rafforzare la correlazione `process_tree` con stato parent-child persistente
   e benchmark dedicati;
6. mantenere catene di correlazione corte e leggibili;
7. estendere la selezione detector MITRE già implementata con un report di
   coverage ATT&CK e una modalità di spiegazione della policy;
8. estendere con cautela i filtri kernel-side minimi dopo la prima allowlist
   UID, mantenendo policy e detector in userspace;
9. continuare a misurare CPU, volume eventi e riduzione rumore.

Restano inoltre validi alcuni task infrastrutturali:

- aggiungere lost channel e metriche per il perf buffer;
- rendere configurabile la dimensione del perf buffer;
- mantenere separati log runtime, eventi e alert: zap gestisce la diagnostica,
  mentre `pkg/output` resta il sink dei dati di monitoraggio;
- completare mapping human-readable per `prctl`, socket option, syscall e
  valori driver-specific.

## 12. Comandi utili

Compilazione eBPF:

```bash
make bpf
```

Build/test Go:

```bash
make build
```

Run:

```bash
make run
```

Build binario:

```bash
sudo ./dist/project --events sched_process_exec --output table
```

## 13. Sintesi

Il progetto e' passato da uno userspace che restava semplicemente in attesa a un runtime che:

- prepara la config;
- legge l'oggetto eBPF embedded o da path esplicito;
- carica la collection;
- apre il perf buffer tramite `libbpfgo`;
- compila le allowlist evento esatte delle policy nella selezione dei programmi
  eBPF da caricare e agganciare;
- seleziona e attacca solo i programmi eBPF richiesti;
- supporta raw tracepoint, tracepoint, kprobe e kretprobe;
- decodifica payload scalari, stringhe, array di stringhe, sockaddr,
  credenziali e puntatori;
- stampa output `table` e `json` arricchito;
- prepara config policy/detector/alert;
- carica policy YAML e le normalizza in un manager userspace;
- carica e valida definizioni detector YAML;
- dispone di registry, dispatcher ed engine detector collegati al runtime;
- sopprime alert ripetuti tramite dedup temporale breve;
- supporta `--alerts-only` per stampare solo gli alert detector;
- gestisce cleanup ordinato.

Le modifiche sono coerenti con l'architettura di Tracee a livello di pattern,
ma restano volutamente piu' semplici e target-specific per la tesi.

Il punto raggiunto e' importante: il problema non e' piu' caricare eBPF o
stampare un evento singolo. Il tool ha ora una pipeline end-to-end ampia, con
copertura process/security, filesystem, memoria, cgroup, moduli, BPF e segnali
di kernel hardening. In parallelo e' stato preparato e collegato al runtime il
layer userspace per policy, detector e alert. Il logging runtime e' stato
strutturato con zap mantenendo separati eventi e alert. La prossima fase
riguarda primi detector correlati, configurabilita' selettiva del dedup se
necessaria e misurazione sistematica delle prestazioni.

## 14. Note storiche su verifier e helper comuni

Le modifiche iniziali a buffer e helper comuni restano importanti per capire
come il progetto e' arrivato allo stato attuale:

- gli argomenti sono stati serializzati con layout verifier-friendly;
- gli offset dinamici sono stati sostituiti o limitati con controlli locali;
- le helper di salvataggio scalari/stringhe sono state adattate per evitare
  accessi fuori mappa;
- `barrier()` e' stato aggiunto come nota al compilatore per mantenere piu'
  prevedibile l'ordine delle operazioni in punti sensibili.

Queste scelte hanno permesso di superare i primi rigetti del verifier e sono
alla base del protocollo evento ancora usato dal decoder Go.
