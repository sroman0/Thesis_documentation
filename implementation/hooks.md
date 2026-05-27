# Hook implementati

## Regole decisionali per nuovi hook

Quando si aggiunge un nuovo hook, la scelta del punto di osservazione deve
partire dal significato che vogliamo dare all'evento, non solo dalla syscall che
lo genera.

Regole pratiche:

- se serve sapere se l'operazione e' riuscita o fallita, usare un modello
  syscall `sys_enter` + `sys_exit`, perche' il return value e' noto solo a fine
  syscall;
- se serve l'oggetto kernel gia' risolto, preferire un hook semantico
  `kprobe`/LSM quando disponibile;
- se l'evento rappresenta un controllo security prima della decisione finale
  del kernel, usare l'hook LSM/security e documentare chiaramente che l'evento
  descrive la validazione, non necessariamente il completamento della syscall;
- se esiste un tracepoint stabile e sufficiente, preferirlo a un kprobe per gli
  eventi di lifecycle;
- evitare dispatcher generici quando il kernel target offre un hook specifico
  piu' economico e piu' leggibile.

Schema mentale:

```text
Mi serve il return value?
    si  -> syscall enter/exit, modello Tracee-like
    no  -> esiste un hook kernel piu' semantico?
              si  -> kprobe/LSM/security hook
              no  -> tracepoint o syscall enter dedicata
```

Esempi:

- `openat`, `mprotect`, `ptrace`, `mount`, `setns`: spesso il return value e'
  importante, quindi il modello `sys_enter` + `sys_exit` e' piu' corretto;
- `security_bprm_check`, `security_task_fix_setuid`,
  `security_socket_connect`, `security_task_kill`: l'hook security offre gia'
  un significato kernel piu' alto e spesso un oggetto risolto;
- `sched_process_fork`, `sched_process_exec`, `sched_process_exit`: il kernel
  espone tracepoint di lifecycle gia' adatti.

## Process lifecycle

### `sched_process_fork`

Tipo attach:

```text
raw_tracepoint/sched_process_fork
```

Scopo:

- intercettare creazione di processi/thread;
- raccogliere parent PID/TID;
- raccogliere child PID/TID;
- raccogliere start time del child.

### `sched_process_exec`

Tipo attach:

```text
raw_tracepoint/sched_process_exec
```

Scopo:

- intercettare exec;
- leggere `linux_binprm->filename`;
- inviare il path allo userspace;
- rappresentare una exec riuscita.

### `execve`

Tipo attach:

```text
tracepoint/syscalls/sys_enter_execve
```

Scopo:

- intercettare il tentativo di esecuzione prima che il processo cambi immagine;
- leggere il primo argomento della syscall, cioe' il path richiesto;
- leggere `argv` come array di stringhe;
- inviare l'evento tramite perf buffer usando il path standard
  `init_program_data` + `events_perf_submit`.

Nota: `execve` e `sched_process_exec` non sono duplicati perfetti.
`execve` vede il tentativo, mentre `sched_process_exec` vede l'esecuzione
riuscita. Questa distinzione e' utile per auditing e detection: un tentativo
fallito puo' essere comunque informativo.

Payload attuale:

- `pathname`: path richiesto dalla syscall;
- `argv`: argomenti del nuovo programma, serializzati come array di stringhe.

La prima implementazione di `execve` usava un secondo
`raw_tracepoint/sys_enter` e filtrava manualmente `SYSCALL_EXECVE`. E' stata
poi sostituita dal tracepoint dedicato per evitare che il programma venga
eseguito su ogni syscall del sistema.

### `execveat`

Tipo attach:

```text
tracepoint/syscalls/sys_enter_execveat
```

Scopo:

- intercettare tentativi di exec fd-relative o con flag dedicati;
- salvare `dirfd`, `pathname`, `flags` e `argv`;
- distinguere casi come path relativo a directory file descriptor o
  `AT_EMPTY_PATH`.

`execveat` non sostituisce `execve`: sono syscall diverse. Il payload e'
volutamente diverso per evitare ridondanza:

- `execve`: `pathname`, `argv`;
- `execveat`: `dirfd`, `pathname`, `flags`, `argv`.

Il decoder userspace ora supporta `StrArrT`, quindi `argv` viene esposto come
lista Go/JSON di stringhe e nel formato `table` come:

```text
argv=["cmd","arg1","arg2"]
```

Per testarlo non basta lanciare comandi comuni come `ls` o `whoami`, perche'
di solito usano `execve`. Serve un programma che invochi esplicitamente
`SYS_execveat`.

### `sched_process_exit`

Tipo attach:

```text
raw_tracepoint/sched_process_exit
```

Scopo:

- intercettare uscita processo/thread;
- raccogliere exit code;
- capire se sta terminando il thread group.

## Security-related hooks

### `task_rename`

Tipo attach:

```text
raw_tracepoint/task_rename
```

Scopo:

- intercettare cambio nome task;
- salvare vecchio nome e nuovo nome.

### `cap_capable`

Tipo attach:

```text
kprobe/cap_capable
```

Scopo:

- intercettare controlli di capability;
- salvare capability richiesta.

### `security_bprm_check`

Tipo attach:

```text
kprobe/security_bprm_check
```

Scopo:

- intercettare la validazione security del binario durante il percorso di exec;
- salvare il path risolto dal kernel;
- salvare metadati stabili del file (`dev`, `inode`);
- salvare `filename`, `argc` ed `envc` dal `linux_binprm`.

Posizione nella pipeline di exec:

```text
execve / execveat
    -> prepare linux_binprm
    -> security_bprm_check
    -> sched_process_exec, se l'exec riesce
```

Questo evento e' quindi diverso da `execve` e da `sched_process_exec`.
`execve`/`execveat` descrivono la richiesta fatta da userspace,
`security_bprm_check` descrive il file eseguibile che il kernel sta validando,
mentre `sched_process_exec` conferma che il cambio immagine e' avvenuto.

Campi emessi:

- `pathname`: path dell'eseguibile ricostruito da `linux_binprm->file`;
- `dev`: device del filesystem che contiene il file;
- `inode`: inode del file eseguibile;
- `filename`: valore di `linux_binprm->filename`;
- `argc`: numero di argomenti del nuovo programma;
- `envc`: numero di variabili d'ambiente del nuovo programma.

`dev` e `inode` sono utili per identificare il file in modo piu' stabile del
solo path, per esempio in presenza di rename o hard link.

Nota tecnica: Tracee salva anche `argv` ed eventualmente `envp` nel percorso
exec. Il progetto ora supporta gli array di stringhe nel decoder e li usa per
`execve`/`execveat`. Su `security_bprm_check` la raccolta di `argv` non e'
stata ancora aggiunta perche' richiede una scelta piu' precisa sulle dipendenze
tra hook syscall e hook LSM. `envp` resta intenzionalmente escluso per ora:
produce molto rumore e puo' contenere dati sensibili.

### `security_file_open`

Tipo attach:

```text
kprobe/security_file_open
```

Scopo:

- intercettare aperture file dopo la risoluzione VFS;
- osservare il path risolto dal kernel, non solo il path richiesto da
  userspace;
- salvare metadati stabili del file (`dev`, `inode`, `ctime`);
- preparare future detection su accesso a file sensibili, credential files,
  log, configurazioni e artefatti eseguibili.

Campi emessi:

- `pathname`: path risolto dal kernel a partire da `struct file->f_path`;
- `flags`: flag di apertura presenti in `file->f_flags`;
- `dev`: device del filesystem;
- `inode`: inode del file;
- `ctime`: change time dell'inode in nanosecondi;
- `syscall_pathname`: path originariamente passato alla syscall, quando
  recuperabile da `open`, `openat`, `openat2`, `execve` o `execveat`.

Decisione tecnica: per questo evento non viene emesso il return value. Il valore
principale dell'hook e' il `struct file *` gia' risolto dal kernel. Per
distinguere aperture riuscite e fallite e' stato affiancato l'evento syscall
`open`, descritto sotto.

### `open`

Tipo attach:

```text
tracepoint/syscalls/sys_enter_open
tracepoint/syscalls/sys_exit_open
tracepoint/syscalls/sys_enter_openat
tracepoint/syscalls/sys_exit_openat
```

Scopo:

- osservare tentativi di apertura file prima e dopo l'esecuzione della syscall;
- unificare `open` e `openat` in un singolo evento logico;
- salvare `returnValue`, quindi distinguere aperture riuscite, file descriptor
  restituiti ed errori come `EPERM`/`EACCES`/`ENOENT`;
- completare `security_file_open`, che resta utile per path risolto e metadati
  stabili del file.

Campi emessi:

- `operation`: variante syscall che ha generato l'evento (`open`, `openat`);
- `dirfd`: directory file descriptor (`AT_FDCWD` per `open`);
- `pathname`: path passato dalla syscall;
- `flags`: flag di apertura richieste;
- `mode`: mode richiesto quando le flag creano un file;
- `returnValue`: file descriptor o errore negativo.

Decisione tecnica: Tracee modella le syscall con enter/exit e return value.
Qui seguiamo lo stesso pattern, ma manteniamo un solo evento user-facing
`open` con campo `operation`. Su Rocky Linux 4.18 non registriamo `openat2`,
perche' non e' parte del kernel target e potrebbe rendere fragile l'attach di
default.

### `chmod`

Tipo attach:

```text
tracepoint/syscalls/sys_enter_chmod
tracepoint/syscalls/sys_exit_chmod
tracepoint/syscalls/sys_enter_fchmod
tracepoint/syscalls/sys_exit_fchmod
tracepoint/syscalls/sys_enter_fchmodat
tracepoint/syscalls/sys_exit_fchmodat
```

Scopo:

- osservare cambi di permessi su file e file descriptor;
- unificare `chmod`, `fchmod` e `fchmodat` in un singolo evento logico;
- emettere l'evento solo all'uscita della syscall, quando e' disponibile
  `returnValue`;
- preparare detection su file resi eseguibili, permessi troppo permissivi o
  tentativi falliti su file sensibili.

Campi emessi:

- `operation`: variante syscall che ha generato l'evento (`chmod`, `fchmod`,
  `fchmodat`);
- `dirfd`: directory file descriptor per `chmod`/`fchmodat`;
- `fd`: file descriptor per `fchmod`;
- `pathname`: path passato dalla syscall, quando disponibile;
- `mode`: permessi richiesti in formato ottale nel layer di output;
- `returnValue`: esito finale della syscall.

Decisione tecnica: Tracee modella `chmod`, `fchmod` e `fchmodat` come eventi
syscall separati e offre anche un hook semantico `chmod_common`. Nel progetto
li abbiamo mantenuti come evento logico unico per ridurre la superficie CLI,
ma abbiamo conservato il pattern enter/exit con return value. Su Rocky Linux
4.18 `fchmodat` non espone un argomento `flags` come `fchmodat2`, quindi la
prima versione non salva `flags`.

### `chown`

Tipo attach:

```text
tracepoint/syscalls/sys_enter_chown
tracepoint/syscalls/sys_exit_chown
tracepoint/syscalls/sys_enter_fchown
tracepoint/syscalls/sys_exit_fchown
tracepoint/syscalls/sys_enter_fchownat
tracepoint/syscalls/sys_exit_fchownat
tracepoint/syscalls/sys_enter_lchown
tracepoint/syscalls/sys_exit_lchown
```

Scopo:

- osservare cambi di owner e group su file e file descriptor;
- unificare `chown`, `fchown`, `fchownat` e `lchown` in un singolo evento
  logico;
- emettere l'evento solo all'uscita della syscall, quando e' disponibile
  `returnValue`;
- preparare detection su trasferimenti sospetti di ownership, tentativi falliti
  su file sensibili e modifiche ai metadati di file eseguibili o di sistema.

Campi emessi:

- `operation`: variante syscall che ha generato l'evento (`chown`, `fchown`,
  `fchownat`, `lchown`);
- `dirfd`: directory file descriptor per `chown`, `fchownat` e `lchown`;
- `fd`: file descriptor per `fchown`;
- `pathname`: path passato dalla syscall, quando disponibile;
- `owner`: UID richiesto come nuovo owner;
- `group`: GID richiesto come nuovo group;
- `flags`: flag di `fchownat`, se presenti;
- `returnValue`: esito finale della syscall.

Decisione tecnica: l'implementazione segue lo stesso modello di `chmod`, cioe'
enter/exit con salvataggio temporaneo degli argomenti e submit all'exit. Questo
mantiene il valore di ritorno, che e' importante per distinguere un cambio owner
effettivo da un tentativo negato. Le quattro syscall restano un solo evento
user-facing (`chown`) e vengono distinte tramite `operation`.

### `security_task_setrlimit`

Tipo attach:

```text
kprobe/security_task_setrlimit
```

Scopo:

- intercettare modifiche ai resource limits;
- salvare target PID, resource, soft limit e hard limit.

### `security_settime64`

Tipo attach:

```text
kprobe/security_settime64
```

Scopo:

- intercettare tentativi di modificare il tempo di sistema.

### `security_task_prctl`

Tipo attach:

```text
kprobe/security_task_prctl
```

Scopo:

- intercettare chiamate `prctl` passate dal security hook;
- utile per future detection su manipolazioni di processo.

Argomenti attuali:

- `option`: operazione `prctl` richiesta;
- `arg2`, `arg3`, `arg4`, `arg5`: significato variabile in base a `option`.

Nota: questi valori sono ancora stampati in forma numerica. Un miglioramento
futuro sara' mappare costanti come `PR_SET_VMA`, `PR_SET_NAME`,
`PR_SET_NO_NEW_PRIVS` e `PR_SET_SECCOMP`.

### `security_task_fix_setuid`

Tipo attach:

```text
kprobe/security_task_fix_setuid
```

Scopo:

- intercettare cambiamenti degli attributi UID del task durante il percorso
  `set*uid`;
- confrontare le credenziali vecchie e nuove prima che vengano installate;
- emettere l'evento solo se cambia almeno uno tra `uid`, `euid`, `suid` e
  `fsuid`;
- osservare transizioni di privilegio come `root -> nobody`, `utente -> root`
  o cambi temporanei di effective UID eseguiti da programmi come `sudo` e
  `sshd`.

Campi emessi:

- `old_uid`, `new_uid`: real UID prima e dopo la transizione;
- `old_euid`, `new_euid`: effective UID prima e dopo la transizione;
- `old_suid`, `new_suid`: saved UID prima e dopo la transizione;
- `old_fsuid`, `new_fsuid`: filesystem UID prima e dopo la transizione;
- `flags`: tipo di operazione `set*uid` che ha attivato l'hook.

Il campo `flags` usa i valori `LSM_SETID_*` definiti dal kernel:

```text
LSM_SETID_ID   = 1  setuid/setgid
LSM_SETID_RE   = 2  setreuid/setregid
LSM_SETID_RES  = 4  setresuid/setresgid
LSM_SETID_FS   = 8  setfsuid/setfsgid
```

Nel kernel moderno questi valori sono definiti in `include/linux/security.h`.
Il codice di `kernel/sys.c` mostra poi dove vengono usati: per esempio
`setuid` chiama `security_task_fix_setuid(..., LSM_SETID_ID)`, `setreuid`
chiama `security_task_fix_setuid(..., LSM_SETID_RE)`, `setresuid` chiama
`security_task_fix_setuid(..., LSM_SETID_RES)` e `setfsuid` chiama
`security_task_fix_setuid(..., LSM_SETID_FS)`.

Nota operativa: sul kernel target Rocky Linux 4.18 della VM e' presente il
simbolo `security_task_fix_setuid`, mentre non e' presente
`security_task_fix_setgid`. Per questo il progetto ha implementato per ora solo
la variante UID. Per tracciare cambi GID in modo robusto su questa VM, una
prossima strada e' osservare `commit_creds` e filtrare le transizioni in cui
cambiano `gid`, `egid`, `sgid` o `fsgid`.

Fonti utili:

- Linux Kernel API, `security_task_fix_setuid` e `security_task_fix_setgid`:
  <https://docs.kernel.org/6.15/core-api/kernel-api.html>
- Definizione di `LSM_SETID_*` in `include/linux/security.h`:
  <https://codebrowser.dev/linux/linux/include/linux/security.h.html>
- Uso delle flag nelle syscall `set*uid` in `kernel/sys.c`:
  <https://codebrowser.dev/linux/linux/kernel/sys.c.html>
- SafeSetID LSM, contesto security sulle transizioni UID/GID:
  <https://docs.kernel.org/6.2/admin-guide/LSM/SafeSetID.html>

### `security_task_kill`

Tipo attach:

```text
kprobe/security_task_kill
```

Scopo:

- intercettare tentativi di invio segnale verso un task target;
- rappresentare la relazione `processo sorgente -> segnale -> processo target`;
- arricchire l'evento con il task target gia' risolto dal kernel;
- preparare future detection su terminazione, sospensione o controllo di
  processi sensibili.

Campi emessi:

- `target_host_pid`: TGID del processo target nel namespace host;
- `target_host_tid`: TID del task target nel namespace host;
- `target_uid`: real UID del target;
- `target_comm`: nome breve del task target;
- `signal`: numero del segnale richiesto. Lo strato di output lo arricchisce
  con una label simbolica quando possibile, ad esempio `SIGTERM(15)`.

Il processo sorgente non viene duplicato negli argomenti perche' e' gia'
presente nel contesto standard dell'evento (`pid`, `tid`, `uid`, `comm`).

Limite noto: questo hook viene eseguito durante la validazione security
dell'invio del segnale. Non espone il return value finale della syscall che ha
generato il segnale. L'evento indica quindi che il kernel sta valutando una
relazione di signal delivery, non che la syscall sia gia' terminata con
successo. Se in futuro servira' distinguere successo/fallimento in modo
rigoroso, andra' valutato un evento syscall `kill`/`tgkill`/`pidfd_send_signal`
con modello `sys_enter` + `sys_exit`.

### `ptrace`

Tipo attach:

```text
tracepoint/syscalls/sys_enter_ptrace
tracepoint/syscalls/sys_exit_ptrace
```

Scopo:

- intercettare tentativi di tracing o manipolazione di un altro processo;
- osservare primitive usate da debugger, `strace`, `gdb` e tecniche di
  process inspection/injection;
- mantenere il valore di ritorno della syscall per distinguere richieste
  riuscite e fallite;
- preparare detection come anti-debugging (`PTRACE_TRACEME`) o code injection
  (`PTRACE_POKETEXT`, `PTRACE_POKEDATA`).

Modello implementativo:

- `sys_enter_ptrace` salva gli argomenti `request`, `pid`, `addr` e `data`
  nella `args_map`;
- `sys_exit_ptrace` recupera gli argomenti, aggiunge `returnValue` e invia un
  singolo evento userspace;
- questo segue il modello enter/exit usato quando il return value e'
  semanticamente importante.

Campi emessi:

- `request`: operazione ptrace richiesta, arricchita in output con label come
  `PTRACE_ATTACH(16)`;
- `pid`: PID target indicato dalla syscall;
- `addr`: indirizzo passato alla syscall;
- `data`: valore/puntatore passato alla syscall;
- `returnValue`: risultato finale della syscall.

Nota: a differenza di `security_task_kill`, qui non usiamo l'hook security
semantico. La scelta e' intenzionale: per `ptrace` e' importante sapere se il
tentativo e' stato autorizzato o negato dal kernel.

## Networking hooks

Gli hook networking sono stati integrati dal branch collaboratore e registrati
in `pkg/ebpf/probes/probes.go`.

### `security_socket_create`

Tipo attach:

```text
kprobe/security_socket_create
```

Scopo:

- intercettare la creazione di socket;
- salvare family, type, protocol e flag `kern`.

### `security_socket_listen`

Tipo attach:

```text
kprobe/security_socket_listen
```

Scopo:

- intercettare socket messi in ascolto;
- salvare file descriptor, indirizzo locale e backlog.

### `security_socket_connect`

Tipo attach:

```text
kprobe/security_socket_connect
```

Scopo:

- intercettare tentativi di connessione;
- salvare file descriptor, tipo socket e indirizzo remoto.

### `security_socket_accept`

Tipo attach:

```text
kprobe/security_socket_accept
```

Scopo:

- intercettare accept su socket server;
- salvare file descriptor e indirizzo locale.

### `security_socket_bind`

Tipo attach:

```text
kprobe/security_socket_bind
```

Scopo:

- intercettare bind su socket;
- salvare file descriptor e indirizzo locale.

### `security_socket_setsockopt`

Tipo attach:

```text
kprobe/security_socket_setsockopt
```

Scopo:

- intercettare modifiche alle opzioni socket;
- salvare file descriptor, level, optname e indirizzo locale.

### `security_socket_recvmsg` e `security_socket_sendmsg`

Tipo attach:

```text
kprobe/security_socket_recvmsg
kprobe/security_socket_sendmsg
```

Scopo:

- associare socket e contesto task tramite inode map;
- preparare informazioni utili per eventi networking piu' completi.

## Nota su ring buffer e perf buffer

Gli hook attuali inviano gli eventi tramite `events_perf_submit`, quindi passano
dal perf buffer `events`.

La ring buffer `events_ringbuf` e la helper `events_ringbuf_submit` restano nel
progetto per versatilita' e per possibili esperimenti/fallback futuri. Anche il
runtime Go continua ad aprire `InitRingBuf("events_ringbuf", ...)`, ma la
direzione operativa corrente e' usare il perf buffer come canale principale.

Questa scelta riduce la frammentazione tra hook storici, syscall enter/exit e
networking: tutti usano lo stesso transport, mentre il supporto ring buffer non
viene rimosso.

## Nota sui tipi di attach

Il registry Go in `pkg/ebpf/probes/probes.go` supporta ora tre tipi principali:

- raw tracepoint, tramite `AttachRawTracepoint`;
- tracepoint classico, tramite `AttachTracepoint`;
- kprobe, tramite `AttachKprobe`.

L'aggiunta dei tracepoint classici permette di usare hook syscall specifici,
come `syscalls/sys_enter_execve`, quando il kernel target li espone. Questo
riduce il lavoro inutile rispetto a un raw tracepoint generico piu' un filtro
manuale nel programma eBPF.

## Collegamenti

- [Timeline](../timeline.md)
- [Overview implementazione](overview.md)
- [Debugging verifier](../debugging/ebpf-verifier.md)
