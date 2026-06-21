# Hook implementati

Ecco gli hook principali implementati, con spiegazione breve:

- `sched_process_fork`: intercetta la creazione di processi o thread.
- `sched_process_exec`: intercetta una exec riuscita.
- `sched_process_exit`: intercetta la terminazione di un processo/thread.
- `task_rename`: intercetta il cambio nome del task.
- `execve`: intercetta tentativi di esecuzione tramite `execve`.
- `execveat`: intercetta tentativi di esecuzione tramite `execveat`.
- `process_execute_failed`: intercetta exec fallite, con errore restituito.

- `clone`: osserva la creazione di nuovi processi/thread.
- `fork`: osserva fork classiche.
- `vfork`: osserva vfork.
- `setns`: osserva ingresso in namespace esistenti.
- `unshare`: osserva creazione/separazione di namespace.
- `switch_task_ns`: osserva il cambio effettivo dei namespace del task.

- `security_bprm_check`: osserva la validazione security di un binario prima dell’exec.
- `security_bprm_creds_for_exec`: osserva la preparazione delle credenziali per il nuovo eseguibile.
- `security_task_fix_setuid`: osserva cambi UID reali/effective/saved/fsuid.
- `commit_creds`: osserva le credenziali realmente applicate al processo.
- `security_task_kill`: osserva invio di segnali tra processi.
- `security_task_prctl`: osserva operazioni `prctl`.
- `security_task_setrlimit`: osserva cambi ai limiti di risorsa.
- `security_settime64`: osserva tentativi di modifica dell’orario di sistema.
- `cap_capable`: osserva controlli di capability.

- `open`: osserva aperture file con return value.
- `chmod`: osserva cambi di permessi file.
- `chown`: osserva cambi owner/group file.
- `security_file_open`: osserva aperture file dopo risoluzione kernel.
- `security_file_permission`: osserva controlli di permesso su file già aperti.
- `security_file_ioctl`: osserva chiamate `ioctl` su file/device.
- `security_inode_unlink`: osserva cancellazioni file.
- `security_inode_rename`: osserva rename/sostituzioni file.
- `security_inode_symlink`: osserva creazione di symlink.
- `security_inode_mknod`: osserva creazione di file speciali/device node.
- `security_sb_mount`: osserva mount.
- `security_sb_umount`: osserva unmount.

- `mmap`: osserva memory mapping.
- `mprotect`: osserva cambi di protezione memoria.
- `pkey_mprotect`: osserva cambi protezione memoria con protection key.
- `security_mmap_file`: osserva mapping file lato security.
- `security_file_mprotect`: osserva transizioni di permessi memoria lato security.
- `memfd_create`: osserva creazione di file anonimi in memoria.
- `process_vm_readv`: osserva letture memoria cross-process.
- `process_vm_writev`: osserva scritture memoria cross-process.

- `security_bpf`: osserva richieste alla syscall `bpf`.
- `security_bpf_map`: osserva operazioni security su mappe eBPF.
- `security_bpf_prog`: osserva operazioni security su programmi eBPF.

- `security_kernel_read_file`: osserva file letti direttamente dal kernel.
- `security_kernel_post_read_file`: osserva completamento della lettura kernel.
- `module_load`: osserva caricamento di moduli kernel.
- `module_free`: osserva rimozione/liberazione di moduli kernel.
- `do_init_module`: osserva inizializzazione moduli kernel con return value.
- `call_usermodehelper`: osserva helper userspace avviati dal kernel.

- `cgroup_attach_task`: osserva spostamento di task tra cgroup.
- `cgroup_mkdir`: osserva creazione di cgroup.
- `cgroup_rmdir`: osserva rimozione di cgroup.

- `do_sigaction`: osserva modifiche agli handler dei segnali.

- `security_socket_create`: osserva creazione socket.
- `security_socket_listen`: osserva socket in ascolto.
- `security_socket_connect`: osserva connessioni socket.
- `security_socket_accept`: osserva accept su socket.
- `security_socket_bind`: osserva bind socket.
- `security_socket_setsockopt`: osserva opzioni socket.
- `security_socket_recvmsg`: osserva ricezione messaggi socket.
- `security_socket_sendmsg`: osserva invio messaggi socket.

La roadmap completa e' in
[implementation/hook-roadmap.md](implementation/hook-roadmap.md).
