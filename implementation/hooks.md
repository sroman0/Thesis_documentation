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

### `process_execute_failed`

Tipo attach:

```text
tracepoint/syscalls/sys_enter_execve
tracepoint/syscalls/sys_exit_execve
tracepoint/syscalls/sys_enter_execveat
tracepoint/syscalls/sys_exit_execveat
```

Scopo:

- osservare solo tentativi di exec falliti;
- correlare entry ed exit della syscall per avere il `returnValue`;
- distinguere `execve` ed `execveat` tramite il campo `operation`;
- mantenere il payload compatto: path richiesto, `dirfd`, `flags` e risultato.

Campi emessi:

- `operation`: variante che ha generato l'evento (`execve`, `execveat`);
- `dirfd`: directory file descriptor, usato da `execveat`;
- `pathname`: path richiesto dalla syscall;
- `flags`: flag di `execveat`, zero per `execve`;
- `returnValue`: errore negativo restituito dal kernel.

Decisione tecnica: nel progetto di riferimento il percorso exec fallito e'
integrato in una pipeline piu' ampia. Qui e' stato scelto un modello piu'
diretto e adatto al kernel target: salviamo lo stato in entry e stampiamo solo
in exit quando il valore di ritorno e' negativo. In questo modo l'evento non
duplica `sched_process_exec`, che rappresenta invece exec riuscite.

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

### `security_bprm_creds_for_exec`

Tipo attach:

```text
kprobe/security_bprm_creds_for_exec
```

Scopo:

- osservare una fase successiva del percorso di exec, quando il kernel sta
  preparando le credenziali del nuovo programma;
- completare `security_bprm_check` con un checkpoint piu' vicino al cambio
  effettivo di credenziali;
- raccogliere gli stessi metadati principali del binario (`pathname`, `dev`,
  `inode`, `filename`, `argc`, `envc`, `ctime`).

Campi emessi:

- `pathname`: path dell'eseguibile ricostruito dal `struct file`;
- `dev`, `inode`, `ctime`: identificazione stabile del file;
- `filename`: valore di `linux_binprm->filename`;
- `argc`, `envc`: numero di argomenti e variabili d'ambiente.

Nota progettuale: Tracee usa questo hook soprattutto come supporto interno per
il percorso exec. Nel progetto viene esposto come evento dedicato per rendere
visibile il momento in cui il kernel prepara le credenziali associate al nuovo
programma.

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

### `security_bpf`

Tipo attach:

```text
kprobe/security_bpf
```

Scopo:

- osservare richieste alla syscall `bpf(2)` nel punto in cui attraversano il
  controllo security del kernel;
- distinguere operazioni come `BPF_MAP_CREATE`, `BPF_PROG_LOAD`,
  `BPF_PROG_ATTACH` e simili;
- preparare future detection su caricamento di programmi eBPF o creazione di
  mappe sospette.

Campi emessi:

- `cmd`: comando `bpf(2)`, arricchito in output con il nome simbolico quando
  noto;
- `size`: dimensione dell'attributo userspace passato alla syscall.

Decisione tecnica: questa prima versione non copia `union bpf_attr`. La scelta
riduce costo e complessita' del verifier, mantenendo comunque un segnale utile:
quale operazione eBPF e' stata richiesta.

### `security_bpf_map`

Tipo attach:

```text
kprobe/security_bpf_map
```

Scopo:

- osservare operazioni che attraversano il controllo security su una mappa eBPF
  gia' esistente;
- raccogliere identificatori stabili della mappa senza copiare strutture kernel
  interne;
- preparare future policy su accesso, freeze, pinning o manipolazione di mappe
  eBPF.

Campi emessi:

- `map_id`: ID kernel della mappa eBPF;
- `map_name`: nome della mappa, quando disponibile.

Decisione tecnica: l'evento e' volutamente compatto. Per ora non vengono
copiati attributi o contenuti della mappa; l'obiettivo e' rendere visibile il
controllo security e collegarlo a un oggetto identificabile.

### `security_bpf_prog`

Tipo attach:

```text
kprobe/security_bpf_prog
```

Scopo:

- osservare operazioni security su programmi eBPF gia' presenti nel kernel;
- raccogliere ID, nome e tipo del programma;
- distinguere programmi `KPROBE`, `TRACEPOINT`, `LSM`, `XDP`, `CGROUP_*` e
  altre famiglie tramite output human-readable.

Campi emessi:

- `prog_id`: ID kernel del programma eBPF;
- `prog_name`: nome del programma;
- `prog_type`: tipo del programma, arricchito in output con il nome
  `BPF_PROG_TYPE_*` quando noto.

Questa informazione e' utile per una futura policy engine perche' permette di
distinguere non solo il fatto che `bpf(2)` venga usata, ma anche quale oggetto
eBPF viene toccato.

### `security_kernel_read_file`

Tipo attach:

```text
kprobe/security_kernel_read_file
```

Scopo:

- osservare letture di file eseguite dal kernel stesso;
- rendere visibili casi ad alto valore come moduli kernel, firmware, policy e
  immagini kexec;
- salvare metadati stabili del file letto dal kernel.

Campi emessi:

- `pathname`: path risolto del file, quando disponibile;
- `dev`, `inode`, `ctime`: identificazione stabile del file;
- `type`: motivo della lettura, ad esempio `READING_MODULE`,
  `READING_FIRMWARE` o `READING_POLICY`.

Nota: questo hook e' diverso da `security_file_open`. Qui il soggetto
dell'operazione e' il kernel che legge un file per una propria funzionalita',
non un processo userspace che apre un normale file descriptor.

### `security_kernel_post_read_file`

Tipo attach:

```text
kprobe/security_kernel_post_read_file
```

Scopo:

- completare `security_kernel_read_file` con un checkpoint successivo alla
  lettura;
- osservare quanti byte sono stati effettivamente letti dal kernel;
- mantenere lo stesso contesto file (`pathname`, `dev`, `inode`, `ctime`) e lo
  stesso tipo di lettura (`READING_MODULE`, `READING_FIRMWARE`, ecc.).

Campi emessi:

- `pathname`: path risolto del file, quando disponibile;
- `size`: numero di byte letti;
- `type`: motivo della lettura, arricchito in output con `READING_*`;
- `dev`, `inode`, `ctime`: metadati stabili del file.

Nota operativa: su alcuni test manuali `security_kernel_read_file` puo' non
emettere eventi se il file richiesto e' gia' cacheato o se il comando scelto non
attraversa realmente quel percorso kernel. `security_kernel_post_read_file`
aiuta a osservare il completamento della lettura quando il percorso viene
effettivamente raggiunto.

### `security_file_ioctl`

Tipo attach:

```text
kprobe/security_file_ioctl
```

Scopo:

- osservare controlli security sulle richieste `ioctl`;
- collegare il comando `ioctl` al file/device risolto dal kernel;
- preparare detection future su uso sospetto di device, pseudo-file e
  interfacce speciali.

Campi emessi:

- `pathname`: path risolto del file o device;
- `cmd`: comando ioctl, mostrato in esadecimale quando non noto;
- `arg`: argomento opaco passato a ioctl, mostrato come puntatore/valore raw;
- `dev`, `inode`, `ctime`: metadati stabili del file.

Decisione tecnica: i comandi ioctl sono spesso specifici del driver, quindi il
tool non prova a decodificarli tutti. L'output mantiene il valore raw in
esadecimale e mappa solo i comandi noti quando sono presenti.

### `security_file_permission`

Tipo attach:

```text
kprobe/security_file_permission
```

Scopo:

- osservare controlli di permesso su file gia' aperti;
- completare `security_file_open`, che vede l'apertura iniziale;
- distinguere controlli `MAY_READ`, `MAY_WRITE`, `MAY_EXEC`,
  `MAY_APPEND`, `MAY_OPEN`, ecc.

Campi emessi:

- `pathname`: path risolto del file;
- `mask`: bitmask di permesso, arricchita in output con nomi `MAY_*`;
- `dev`, `inode`, `ctime`: metadati stabili del file.

Nota operativa: questo hook puo' essere molto rumoroso. Durante i test manuali
conviene abilitarlo da solo e usare `--comms` per limitare l'output al processo
di interesse.

### `cgroup_attach_task`

Tipo attach:

```text
raw_tracepoint/cgroup_attach_task
```

Scopo:

- osservare quando un task viene spostato in un cgroup diverso;
- collegare l'evento al task target, non solo al processo che genera il
  movimento;
- preparare il progetto a una lettura piu' container-aware dei processi.

Campi emessi:

- `cgroup_id`: identificatore kernel del cgroup di destinazione;
- `hierarchy_id`: gerarchia cgroup coinvolta;
- `cgroup_path`: path del cgroup di destinazione;
- `target_comm`: nome del task spostato;
- `target_host_pid` e `target_host_tid`: PID/TID host del task target;
- `threadgroup`: indica se lo spostamento riguarda l'intero thread group.

Questo hook e' un raw tracepoint e viene inviato tramite perf buffer come gli
altri hook process/security correnti.

### `cgroup_mkdir` e `cgroup_rmdir`

Tipo attach:

```text
raw_tracepoint/cgroup_mkdir
raw_tracepoint/cgroup_rmdir
```

Scopo:

- osservare creazione e rimozione di cgroup;
- associare ogni evento a `cgroup_id`, gerarchia e path kernel;
- completare `cgroup_attach_task`, che vede invece lo spostamento dei task.

Campi emessi:

- `cgroup_id`: identificatore kernel del cgroup;
- `path`: path del cgroup;
- `hierarchy_id`: gerarchia cgroup coinvolta.

Questi hook sono utili per contesti container e orchestrazione, perche'
permettono di vedere non solo quando un processo entra in un cgroup, ma anche
quando il cgroup viene creato o rimosso.

### `call_usermodehelper`

Tipo attach:

```text
kprobe/call_usermodehelper
```

Scopo:

- intercettare esecuzioni di helper userspace avviate dal kernel;
- distinguere questo percorso dagli `execve` ordinari richiesti da userspace;
- osservare path, argomenti e modalita' di attesa usata dal kernel.

Campi emessi:

- `path`: helper userspace richiesto dal kernel;
- `argv`: argomenti passati all'helper;
- `envp`: ambiente passato all'helper;
- `wait`: modalita' `UMH_*`, arricchita nello strato output quando possibile.

Esempi di casi che possono passare da questo hook sono richieste kernel verso
helper come modprobe o altri programmi chiamati tramite `call_usermodehelper`.

### `module_load`, `module_free` e `do_init_module`

Tipo attach:

```text
raw_tracepoint/module_load
raw_tracepoint/module_free
kprobe/do_init_module
kretprobe/do_init_module
```

Scopo:

- osservare il ciclo di vita dei moduli kernel caricati dinamicamente;
- distinguere la richiesta di inizializzazione (`do_init_module`) dagli eventi
  di load/free esposti dai tracepoint;
- raccogliere metadati del modulo come nome, versione e `srcversion`;
- usare il kretprobe di `do_init_module` per avere il valore di ritorno
  dell'inizializzazione.

Campi emessi:

- `name`: nome del modulo;
- `version`: versione dichiarata dal modulo, quando disponibile;
- `srcversion`: identificatore sorgente del modulo, quando disponibile;
- `returnValue`: presente su `do_init_module`, indica successo o errore.

Decisione tecnica: per ora non e' stata importata la logica piu' avanzata di
ricerca di moduli nascosti. L'obiettivo di questa fase e' rendere visibile il
percorso standard di load/unload, mantenendo payload e verifier complexity
contenuti su Rocky Linux 4.18.

### `proc_create`

Tipo attach:

```text
kprobe/proc_create
```

Scopo:

- osservare la creazione di entry procfs da parte del kernel o di moduli;
- evidenziare possibili superfici di controllo esposte sotto `/proc`;
- affiancare gli eventi sui moduli kernel con un segnale utile per rootkit e
  componenti che pubblicano interfacce custom.

Campi emessi:

- `name`: nome della entry procfs richiesta;
- `proc_ops_addr`: puntatore alla struttura di operazioni associata alla entry.

Decisione tecnica: l'evento resta volutamente compatto. Non tenta di
ricostruire l'intera gerarchia procfs e non copia le funzioni della struttura
`proc_ops`; espone pero' abbastanza informazione per correlare creazione di
entry, caricamento moduli e possibili superfici di controllo.

### `register_kprobe`

Tipo attach:

```text
kprobe/register_kprobe
kretprobe/register_kprobe
```

Scopo:

- osservare registrazioni dinamiche di kprobe;
- capire quale simbolo kernel viene osservato;
- raccogliere gli indirizzi dei callback associati alla probe;
- distinguere registrazioni riuscite e fallite tramite return value.

Campi emessi:

- `symbol_name`: simbolo kernel richiesto dalla kprobe;
- `pre_handler`: indirizzo del callback eseguito prima del simbolo target;
- `post_handler`: indirizzo del callback eseguito dopo il simbolo target;
- `returnValue`: esito della registrazione, arricchito in output come
  `success` o errno.

Decisione tecnica: questo hook usa una coppia kprobe/kretprobe per mantenere il
return value. L'entry salva temporaneamente il `struct kprobe *`, mentre la
return probe emette l'evento con simbolo, handler e risultato finale.

### `kallsyms_lookup_name`

Tipo attach:

```text
kprobe/kallsyms_lookup_name
kretprobe/kallsyms_lookup_name
```

Scopo:

- osservare lookup runtime di simboli kernel interni;
- individuare moduli o codice kernel che risolvono funzioni sensibili;
- correlare il nome richiesto con l'indirizzo restituito dal kernel.

Campi emessi:

- `symbol_name`: simbolo richiesto;
- `address`: indirizzo restituito da `kallsyms_lookup_name`, mostrato in
  esadecimale nello strato output.

Nota operativa: il lookup di simboli kernel puo' essere legittimo. L'evento
diventa piu' interessante quando viene correlato con `module_load`,
`do_init_module`, `register_kprobe`, `proc_create` o future superfici debugfs.

### `do_sigaction`

Tipo attach:

```text
kprobe/do_sigaction
```

Scopo:

- osservare modifiche alla gestione dei segnali di un processo;
- vedere se un segnale viene impostato a default, ignorato o gestito tramite
  handler custom;
- fornire un punto di osservazione utile per comportamenti evasivi o processi
  che alterano signal handling.

Campi emessi:

- `signal`: numero del segnale, arricchito con nome simbolico nello strato di
  output;
- `new_action`: indica se e' stata fornita una nuova azione;
- `new_sa_flags`, `new_sa_mask`, `new_handle_method`, `new_sa_handler`:
  stato richiesto per il nuovo handler;
- `old_action_requested`: indica se il chiamante ha richiesto il vecchio stato;
- `old_sa_flags`, `old_sa_mask`, `old_handle_method`, `old_sa_handler`:
  snapshot dello stato precedente.

Nota tecnica: nel nostro schema vengono sempre emessi tutti gli slot del nuovo
e vecchio handler. Quando una parte non e' inizializzata viene salvato
zero/null. Questa scelta mantiene semplice e stabile il decoder userspace.

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

### `memfd_create`

Tipo attach:

```text
tracepoint/syscalls/sys_enter_memfd_create
tracepoint/syscalls/sys_exit_memfd_create
```

Scopo:

- osservare la creazione di file anonimi residenti in memoria;
- rendere visibili primitive utili per esecuzione fileless, unpacking di
  payload o caricamento dinamico senza un normale file su disco;
- distinguere tentativi riusciti e falliti tramite il valore di ritorno.

Campi emessi:

- `name`: nome diagnostico richiesto dal processo;
- `flags`: flag `MFD_*`, ad esempio `MFD_CLOEXEC`, `MFD_ALLOW_SEALING` o
  `MFD_HUGETLB`;
- `returnValue`: file descriptor creato, oppure errore negativo.

Decisione tecnica: l'evento usa il modello syscall enter/exit. L'entry salva
`name` e `flags`, mentre l'exit aggiunge il risultato e invia un singolo evento
tramite perf buffer. Questo e' preferibile a un evento solo-entry, perche' un
tentativo di `memfd_create` fallito non produce alcun file descriptor anonimo.

Compatibilita': il kernel target Rocky Linux 4.18 espone i tracepoint syscall
dedicati a `memfd_create`, quindi non e' necessario usare il raw syscall engine
generico.

### `mmap`

Tipo attach:

```text
tracepoint/syscalls/sys_enter_mmap
tracepoint/syscalls/sys_exit_mmap
```

Scopo:

- osservare la creazione di nuove regioni di memoria virtuale;
- identificare mapping anonimi, file-backed ed eseguibili;
- mantenere il risultato finale, distinguendo un indirizzo valido da un errno.

Campi emessi:

- `addr`: indirizzo richiesto dal processo;
- `length`: dimensione della regione;
- `prot`: permessi `PROT_*`, tradotti simbolicamente nell'output;
- `flags`: tipo e opzioni `MAP_*`, tradotti simbolicamente nell'output;
- `fd`: file descriptor sorgente, oppure `-1` per mapping anonimi;
- `offset`: offset nel file sorgente;
- `returnValue`: indirizzo della nuova regione oppure errore negativo.

### `mprotect` e `pkey_mprotect`

Tipo attach:

```text
tracepoint/syscalls/sys_enter_mprotect
tracepoint/syscalls/sys_exit_mprotect
tracepoint/syscalls/sys_enter_pkey_mprotect
tracepoint/syscalls/sys_exit_pkey_mprotect
```

Scopo:

- osservare cambi ai permessi di regioni di memoria gia' esistenti;
- rendere visibili transizioni sensibili come memoria scrivibile resa
  eseguibile;
- distinguere modifiche riuscite e fallite tramite il return value;
- includere la protection key nel caso di `pkey_mprotect`.

Campi emessi:

- `addr`: inizio della regione;
- `length`: dimensione della regione;
- `prot`: nuovi permessi `PROT_*`;
- `pkey`: protection key, presente solo in `pkey_mprotect`;
- `returnValue`: `success` per zero oppure errno simbolico.

Decisione tecnica: i tre eventi seguono il modello syscall enter/exit. Gli
argomenti vengono conservati in `args_map` all'ingresso e ricomposti all'uscita
prima del submit sul perf buffer. Sul kernel Rocky Linux 4.18 target sono
presenti tutti e sei i tracepoint dedicati; questa soluzione e' quindi piu'
diretta del raw syscall dispatcher generico e conserva comunque il valore di
ritorno.

### `process_vm_writev`

Tipo attach:

```text
tracepoint/syscalls/sys_enter_process_vm_writev
tracepoint/syscalls/sys_exit_process_vm_writev
```

Scopo:

- osservare scritture dirette nello spazio di memoria di un processo;
- rendere visibile una primitiva usata legittimamente da debugger e runtime,
  ma anche sfruttabile per process injection;
- mantenere il valore di ritorno per distinguere un tentativo negato da una
  scrittura realmente avvenuta;
- preparare una futura policy che segnali soprattutto il caso in cui PID
  sorgente e PID target sono differenti.

Campi emessi:

- `pid`: PID del processo target;
- `local_iov`: indirizzo dell'array `iovec` nel processo chiamante;
- `liovcnt`: numero di vettori locali;
- `remote_iov`: indirizzo dell'array `iovec` riferito al processo target;
- `riovcnt`: numero di vettori remoti;
- `flags`: flag della syscall, attualmente richieste a zero dal kernel;
- `returnValue`: numero di byte scritti oppure errno negativo.

Il layer di output mostra i puntatori in esadecimale e il risultato positivo
come `bytes:N`.

Decisione tecnica: l'evento segue il modello syscall enter/exit usato dagli
altri eventi che richiedono l'esito finale. Il kernel Rocky Linux 4.18 target
espone entrambi i tracepoint dedicati, quindi non e' necessario attivare il raw
syscall dispatcher globale. La prima versione conserva i metadati degli
`iovec`, ma non copia il contenuto scritto: questa scelta limita costo, volume
degli eventi e rischio di esporre dati sensibili.

### `process_vm_readv`

Tipo attach:

```text
tracepoint/syscalls/sys_enter_process_vm_readv
tracepoint/syscalls/sys_exit_process_vm_readv
```

Scopo:

- osservare letture dirette dalla memoria di un processo;
- completare la copertura fornita da `process_vm_writev`;
- distinguere letture riuscite, parziali e negate;
- preparare policy su credential theft, memory inspection e accessi
  cross-process anomali.

Campi emessi:

- `pid`: PID target;
- `local_iov` e `liovcnt`: destinazione locale e numero di vettori;
- `remote_iov` e `riovcnt`: sorgente remota e numero di vettori;
- `flags`: flag della syscall;
- `returnValue`: byte letti oppure errno negativo.

Come per `process_vm_writev`, il payload non viene copiato. Questa scelta evita
di acquisire contenuti sensibili e mantiene contenuto il costo dell'evento.
L'implementazione usa i tracepoint dedicati presenti sul kernel target e
correla entry ed exit tramite `args_map`.

### `fork`, `vfork` e `clone`

Tipo attach:

```text
tracepoint/syscalls/sys_enter_clone
tracepoint/syscalls/sys_exit_clone
tracepoint/syscalls/sys_enter_fork
tracepoint/syscalls/sys_exit_fork
tracepoint/syscalls/sys_enter_vfork
tracepoint/syscalls/sys_exit_vfork
```

Scopo:

- osservare direttamente le syscall di creazione processo/thread;
- mantenere il valore di ritorno, cioe' il PID/TID del child oppure errno;
- completare `sched_process_fork`, che resta il tracepoint lifecycle piu'
  semantico ma non rappresenta il risultato della syscall dal punto di vista
  del chiamante.

Campi emessi:

- `fork`/`vfork`: `returnValue`;
- `clone`: `flags`, `stack`, `parent_tid`, `child_tid`, `tls`,
  `returnValue`.

Decisione tecnica: `fork` e `vfork` non hanno argomenti utili da salvare
all'enter, quindi l'evento viene costruito all'exit. `clone` invece salva gli
argomenti all'ingresso e aggiunge il risultato all'uscita. L'output traduce i
flag `CLONE_*` e mostra i risultati positivi come `child_pid:N`.

### `commit_creds`

Tipo attach:

```text
kprobe/commit_creds
```

Scopo:

- osservare le credenziali realmente applicate al task;
- confrontare identita', namespace utente, securebits e capability prima e
  dopo il commit;
- rilevare primitive importanti per privilege escalation;
- completare `security_task_fix_setuid`, che osserva una fase precedente della
  preparazione delle credenziali.

Campi emessi:

- `old_cred`: UID/GID reali, effective, saved e filesystem, user namespace,
  securebits e capability precedenti;
- `new_cred`: gli stessi campi dopo la transizione.

L'evento viene inviato solo se almeno un campo cambia. Le capability sono
normalizzate in maschere a 64 bit con CO-RE, supportando sia il layout storico
del kernel target sia quello usato dai kernel piu' recenti.

Decisione tecnica: il progetto usa il tipo wire `slim_cred_t`, gia' presente
nel protocollo C, e ha completato il decoder Go per `CredT`. Questo mantiene
old e new credentials come due oggetti strutturati invece di esporre decine di
argomenti indipendenti.

Un singolo comando come `sudo` puo' generare diverse transizioni legittime. Il
filtro kernel elimina i commit senza cambiamenti, ma la riduzione semantica
resta responsabilita' di una futura policy. Esempi ad alto valore sono il
passaggio da `euid != 0` a `euid == 0` e l'acquisizione di nuove capability.

### `setns`

Tipo attach:

```text
tracepoint/syscalls/sys_enter_setns
tracepoint/syscalls/sys_exit_setns
```

Scopo:

- osservare quando un processo tenta di entrare in un namespace esistente;
- distinguere transizioni riuscite da tentativi negati;
- rendere visibili operazioni rilevanti per isolamento container, debugging e
  possibili tecniche di container escape;
- preparare policy che combinino namespace target, identita' del processo ed
  esito della syscall.

Campi emessi:

- `fd`: file descriptor che identifica il namespace target;
- `nstype`: tipo richiesto, mostrato come `CLONE_NEWNS`, `CLONE_NEWNET`,
  `CLONE_NEWPID`, `CLONE_NEWUSER` e altri valori `CLONE_NEW*`;
- `returnValue`: `success` quando la transizione e' accettata oppure errno
  simbolico quando viene negata.

Il valore `nstype=0` viene mostrato come `AUTO`: in questo caso il kernel
determina il tipo di namespace dal file descriptor.

Decisione tecnica: `setns` segue il modello syscall entry/exit perche' il valore
di ritorno e' essenziale. Osservare soltanto l'ingresso non permetterebbe di
distinguere un cambio di namespace effettivo da un tentativo fallito. Sul
kernel Rocky Linux 4.18 target sono disponibili entrambi i tracepoint dedicati.
I campi del contesto C usano la larghezza riportata dal formato tracepoint
reale del kernel e vengono convertiti a `int` prima della serializzazione.

### `unshare`

Tipo attach:

```text
tracepoint/syscalls/sys_enter_unshare
tracepoint/syscalls/sys_exit_unshare
```

Scopo:

- osservare quando un processo separa una parte del proprio execution context;
- rilevare la creazione di nuovi namespace tramite flag come
  `CLONE_NEWUSER`, `CLONE_NEWNS`, `CLONE_NEWNET` e `CLONE_NEWPID`;
- distinguere operazioni riuscite da richieste negate;
- fornire una base per policy su sandbox, isolamento container ed evasione.

Campi emessi:

- `flags`: bitmask delle risorse da separare, convertita in nomi `CLONE_*`;
- `returnValue`: `success` oppure errno simbolico.

`setns` e `unshare` coprono due operazioni complementari: `setns` sposta il
processo in un namespace gia' esistente identificato da un file descriptor,
mentre `unshare` separa risorse del processo corrente e, con i flag
`CLONE_NEW*`, puo' creare nuovi namespace.

Decisione tecnica: anche `unshare` usa il modello entry/exit, perche' il solo
tentativo non dimostra che l'isolamento sia stato modificato. Sul kernel target
il tracepoint espone `unshare_flags` in uno slot a 64 bit; l'evento conserva
quindi una bitmask `unsigned long`, evitando di troncare combinazioni di flag.
L'output mantiene eventuali bit non riconosciuti in forma esadecimale.

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
la variante UID. I cambi GID sono ora coperti da `commit_creds`, che espone le
transizioni effettive di `gid`, `egid`, `sgid` e `fsgid`.

Fonti utili:

- Linux Kernel API, `security_task_fix_setuid` e `security_task_fix_setgid`:
  <https://docs.kernel.org/6.15/core-api/kernel-api.html>
- Definizione di `LSM_SETID_*` in `include/linux/security.h`:
  <https://codebrowser.dev/linux/linux/include/linux/security.h.html>
- Uso delle flag nelle syscall `set*uid` in `kernel/sys.c`:
  <https://codebrowser.dev/linux/linux/kernel/sys.c.html>
- SafeSetID LSM, contesto security sulle transizioni UID/GID:
  <https://docs.kernel.org/6.2/admin-guide/LSM/SafeSetID.html>

### `setuid`, `setgid`, `setreuid`, `setregid`, `setresuid`, `setresgid`, `setfsuid`, `setfsgid`

Tipo attach:

```text
tracepoint/syscalls/sys_enter_setuid
tracepoint/syscalls/sys_exit_setuid
...
tracepoint/syscalls/sys_enter_setfsgid
tracepoint/syscalls/sys_exit_setfsgid
```

Scopo:

- osservare le syscall userspace che richiedono cambi di identita' UID/GID;
- distinguere tentativi riusciti e negati tramite `returnValue`;
- affiancare `security_task_fix_setuid` e `commit_creds`, che osservano fasi
  diverse e piu' semantiche della transizione credenziali.

Campi emessi:

- `setuid`: `uid`, `returnValue`;
- `setgid`: `gid`, `returnValue`;
- `setreuid`: `ruid`, `euid`, `returnValue`;
- `setregid`: `rgid`, `egid`, `returnValue`;
- `setresuid`: `ruid`, `euid`, `suid`, `returnValue`;
- `setresgid`: `rgid`, `egid`, `sgid`, `returnValue`;
- `setfsuid`: `fsuid`, `returnValue`;
- `setfsgid`: `fsgid`, `returnValue`.

Decisione tecnica: questi eventi seguono il modello enter/exit perche' il
return value e' essenziale. Il kernel puo' negare la transizione richiesta; in
quel caso l'hook LSM puo' comunque essere stato raggiunto, ma la syscall non
ha modificato davvero le credenziali del processo.

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

### `prlimit64`

Tipo attach:

```text
tracepoint/syscalls/sys_enter_prlimit64
tracepoint/syscalls/sys_exit_prlimit64
```

Scopo:

- osservare letture e modifiche dei resource limits;
- distinguere richieste riuscite e fallite tramite `returnValue`;
- completare `security_task_setrlimit`, che osserva il controllo security su
  una modifica, ma non copre allo stesso modo tutte le richieste syscall.

Campi emessi:

- `pid`: processo target, con `0` per il processo corrente;
- `resource`: risorsa `RLIMIT_*`;
- `new_limit`: puntatore alla nuova coppia soft/hard limit, se presente;
- `old_limit`: puntatore dove il kernel scrive il vecchio limite, se presente;
- `returnValue`: `success` oppure errno simbolico.

Nota tecnica: sul kernel Rocky Linux 4.18 i campi del tracepoint `prlimit64`
sono esposti come slot a 8 byte. La struct eBPF usa quindi tipi larghi per
evitare truncation dei puntatori.

### `switch_task_ns`

Tipo attach:

```text
kprobe/switch_task_namespaces
```

Scopo:

- osservare il momento in cui il kernel sostituisce il `nsproxy` di un task;
- completare `setns` e `unshare` con la transizione namespace effettiva;
- emettere soltanto i namespace realmente cambiati.

Campi:

- `pid`: PID del task interessato;
- `new_mnt`, `new_pid`, `new_uts`, `new_ipc`, `new_net`, `new_cgroup`: inode
  number dei nuovi namespace. I campi invariati non vengono serializzati.

### `security_sb_mount` e `security_sb_umount`

Tipo attach:

```text
kprobe/security_sb_mount
kprobe/security_sb_umount
```

`security_sb_mount` espone sorgente, mountpoint risolto, filesystem type e
flag `MS_*`. `security_sb_umount` ricava dal `vfsmount` il mountpoint e il tipo
di filesystem, aggiungendo le flag `MNT_*`.

Sono hook LSM pre-decisione: descrivono la richiesta validata dal kernel, non
il return value finale di `mount(2)` o `umount2(2)`. Questa scelta privilegia
gli oggetti kernel risolti; se una policy richiedera' l'esito rigoroso, gli
eventi potranno essere correlati con syscall entry/exit.

### `security_inode_unlink`

Tipo attach:

```text
kprobe/security_inode_unlink
```

Scopo:

- osservare la cancellazione di un file dopo la risoluzione del dentry;
- raccogliere `pathname`, inode, device e ctime prima della rimozione;
- fornire una base stabile per policy su file sensibili o cancellazioni
  sospette.

Anche questo hook rappresenta la validazione LSM e non include il risultato
finale della syscall `unlink`/`unlinkat`.

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

## Nota sul perf buffer

Gli hook attuali inviano gli eventi tramite `events_perf_submit`, quindi passano
dal perf buffer `events`.

La ring buffer, la relativa mappa e la helper di submit sono state rimosse.
Questa scelta mantiene compatibilita' con Rocky Linux 4.18 e riduce la
frammentazione tra hook storici, syscall enter/exit e networking.

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
