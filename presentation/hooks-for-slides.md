# Hook process, security e kernel sviluppati

Questo documento riassume gli hook non-network sviluppati nel tool. E' pensato
per generare slide sulla parte di Simone senza allegare il codice sorgente.

## Strategia di implementazione degli hook

La scelta dell'hook dipende dal significato dell'evento:

- se serve sapere se una syscall e' riuscita o fallita, si usa un modello
  `enter/exit` per ottenere il valore di ritorno;
- se serve un oggetto kernel gia' risolto, si preferisce un hook semantico come
  kprobe o security hook;
- se il kernel espone un tracepoint stabile, lo si preferisce per eventi di
  lifecycle;
- se un hook e' troppo generico e produce troppo rumore, si valuta un punto di
  osservazione piu' specifico.

Il progetto cerca di seguire pattern simili a Tracee quando ha senso, ma con
adattamenti per Rocky Linux 4.18.

## Process lifecycle

Questi hook descrivono nascita, esecuzione e terminazione dei processi.

| Evento | Scopo |
| --- | --- |
| `sched_process_fork` | Osserva creazione di processi/thread con informazioni su parent e child. |
| `sched_process_exec` | Osserva exec riuscite, quindi il momento in cui il processo cambia immagine. |
| `sched_process_exit` | Osserva terminazione di processi/thread e relativo exit code. |
| `task_rename` | Osserva cambi del nome task, utile per capire transizioni del campo `comm`. |
| `fork` | Osserva syscall `fork` con modello enter/exit e risultato finale. |
| `vfork` | Osserva syscall `vfork` con modello enter/exit e risultato finale. |
| `clone` | Osserva syscall `clone` e flag di creazione del nuovo task. |

## Exec monitoring

Questi eventi osservano tentativi e risultati di esecuzione di programmi.

| Evento | Scopo |
| --- | --- |
| `execve` | Intercetta tentativi di esecuzione tramite `execve`, con path e argv. |
| `execveat` | Intercetta esecuzioni fd-relative o con flag dedicate, con `dirfd`, path, flags e argv. |
| `process_execute_failed` | Emette evento solo quando `execve` o `execveat` falliscono. |
| `security_bprm_check` | Osserva la fase security del caricamento binario. |
| `security_bprm_creds_for_exec` | Osserva preparazione credenziali durante il percorso exec. |

Messaggio chiave per le slide:

- `execve` e `execveat` osservano il tentativo;
- `sched_process_exec` osserva l'exec riuscita;
- `process_execute_failed` permette di non perdere i tentativi falliti.

## Privilege and identity hooks

Questi hook osservano cambiamenti di identita' e privilegi.

| Evento | Scopo |
| --- | --- |
| `security_task_fix_setuid` | Osserva transizioni UID/EUID/SUID/FSUID nel percorso security. |
| `security_task_fix_setgid` | Osserva transizioni GID/EGID/SGID/FSGID. |
| `setuid` | Osserva syscall setuid con esito. |
| `setgid` | Osserva syscall setgid con esito. |
| `setreuid` | Osserva modifica separata di real/effective UID. |
| `setregid` | Osserva modifica separata di real/effective GID. |
| `setresuid` | Osserva modifica atomica di real/effective/saved UID. |
| `setresgid` | Osserva modifica atomica di real/effective/saved GID. |
| `setfsuid` | Osserva modifica del filesystem UID. |
| `setfsgid` | Osserva modifica del filesystem GID. |
| `commit_creds` | Osserva applicazione finale di nuove credenziali nel kernel. |

Rilevanza security:

- escalation di privilegi;
- transizioni root/non-root;
- processi che cambiano identita';
- programmi privilegiati come `sudo` o `sshd`.

## Capability and process control hooks

Questi eventi monitorano controlli di permesso e manipolazioni dei processi.

| Evento | Scopo |
| --- | --- |
| `cap_capable` | Osserva controlli di Linux capability. |
| `security_task_prctl` | Osserva operazioni `prctl`, incluse opzioni sensibili. |
| `security_task_setrlimit` | Osserva modifiche ai limiti di risorse del processo. |
| `prlimit64` | Osserva lettura/modifica dei limiti di processo con esito. |
| `security_task_kill` | Osserva invio segnali a processi target. |
| `ptrace` | Osserva tentativi di tracing/manipolazione di altri processi. |
| `do_sigaction` | Osserva registrazione o modifica degli handler dei segnali. |

Rilevanza security:

- abuso di capability;
- manipolazione di processi;
- uso di `ptrace`;
- segnali verso processi critici;
- modifica comportamento segnali.

## File security hooks

Questi eventi osservano accesso, permessi e operazioni su file.

| Evento | Scopo |
| --- | --- |
| `security_file_open` | Osserva apertura file dal percorso security, con path risolto e metadata. |
| `open` | Osserva syscall `open/openat` con valore di ritorno. |
| `security_file_permission` | Osserva controlli di permission su file. |
| `security_file_ioctl` | Osserva ioctl su file descriptor. |
| `security_inode_unlink` | Osserva cancellazione di file. |
| `security_inode_rename` | Osserva rename atomici. |
| `security_inode_symlink` | Osserva creazione di symlink. |
| `security_inode_mknod` | Osserva creazione di file speciali/device node. |

Rilevanza security:

- accesso a file sensibili;
- modifiche a filesystem;
- cancellazioni sospette;
- creazione di symlink o device node.

## Permission and ownership hooks

Questi eventi osservano modifiche a permessi e ownership.

| Evento | Scopo |
| --- | --- |
| `chmod` | Osserva modifica permessi via path. |
| `fchmod` | Osserva modifica permessi via file descriptor. |
| `fchmodat` | Osserva modifica permessi fd-relative. |
| `chown` | Osserva modifica owner/group via path. |
| `fchown` | Osserva modifica owner/group via file descriptor. |
| `fchownat` | Osserva modifica owner/group fd-relative. |
| `lchown` | Osserva modifica owner/group su link simbolico. |

Questi eventi sono importanti perche' permessi e ownership possono essere usati
per persistenza, privilege escalation o evasione.

## Memory and process injection hooks

Questi eventi osservano operazioni legate a memoria e possibili tecniche di
injection.

| Evento | Scopo |
| --- | --- |
| `mmap` | Osserva mapping di memoria. |
| `mprotect` | Osserva cambi di protezione memoria. |
| `pkey_mprotect` | Osserva cambi protezione con protection keys. |
| `security_mmap_file` | Osserva mapping file-backed dal percorso security. |
| `security_file_mprotect` | Osserva transizioni di protezione su aree mappate. |
| `process_vm_writev` | Osserva scritture dirette nella memoria di un altro processo. |
| `process_vm_readv` | Osserva letture dirette dalla memoria di un altro processo. |
| `memfd_create` | Osserva creazione di file anonimi in memoria. |

Rilevanza security:

- fileless execution;
- process injection;
- memoria eseguibile;
- abuso di memfd.

## Namespace and cgroup hooks

Questi eventi osservano isolamento e spostamenti tra contesti.

| Evento | Scopo |
| --- | --- |
| `setns` | Osserva ingresso in namespace esistente. |
| `unshare` | Osserva separazione da namespace condivisi. |
| `switch_task_ns` | Osserva sostituzione effettiva dei namespace nel kernel. |
| `cgroup_attach_task` | Osserva spostamento di task in cgroup. |
| `cgroup_mkdir` | Osserva creazione cgroup. |
| `cgroup_rmdir` | Osserva rimozione cgroup. |

Rilevanza security:

- isolamento container;
- escape o movimenti anomali tra namespace;
- manipolazione cgroup.

## Mount and filesystem hooks

| Evento | Scopo |
| --- | --- |
| `security_sb_mount` | Osserva mount con target, filesystem type e flag. |
| `security_sb_umount` | Osserva unmount. |

Rilevanza security:

- mount sospetti;
- filesystem temporanei o bind mount;
- preparazione di ambienti isolati o evasivi.

## Kernel, LKM and tampering hooks

Questi eventi osservano attivita' kernel-sensitive e moduli caricabili.

| Evento | Scopo |
| --- | --- |
| `do_init_module` | Osserva inizializzazione modulo kernel e return value. |
| `module_load` | Osserva caricamento modulo. |
| `module_free` | Osserva rimozione modulo. |
| `security_kernel_read_file` | Osserva file letti dal kernel, per esempio firmware o moduli. |
| `security_kernel_post_read_file` | Osserva fase successiva alla lettura file da parte del kernel. |
| `kallsyms_lookup_name` | Osserva lookup di simboli kernel. |
| `register_kprobe` | Osserva registrazione dinamica di kprobe. |
| `proc_create` | Osserva creazione di entry procfs. |
| `call_usermodehelper` | Osserva esecuzione di helper userspace invocati dal kernel. |

Rilevanza security:

- caricamento moduli;
- rootkit;
- kernel tampering;
- interfacce procfs sospette;
- uso di simboli kernel interni.

## eBPF security hooks

| Evento | Scopo |
| --- | --- |
| `security_bpf` | Osserva operazioni generali della syscall bpf. |
| `security_bpf_map` | Osserva accesso o uso di mappe eBPF. |
| `security_bpf_prog` | Osserva programmi eBPF. |

Rilevanza security:

- programmi eBPF potenzialmente abusati;
- caricamento o uso di mappe;
- superficie eBPF come vettore di osservazione o evasione.

## Messaggio conclusivo per le slide

La parte non-network del tool copre gia' molte aree importanti:

- process lifecycle;
- exec monitoring;
- privilegi e credenziali;
- file security;
- permessi e ownership;
- memoria e injection;
- namespace/cgroup;
- mount;
- moduli kernel e tampering;
- superficie eBPF.

La copertura non e' ancora equivalente a Tracee, ma e' sufficiente per mostrare
un monitoring generale e dettagliato della superficie process/security/kernel.

