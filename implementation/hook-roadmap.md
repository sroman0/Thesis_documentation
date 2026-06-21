# Roadmap hook process e security

Questa pagina confronta gli eventi process/security disponibili nel progetto
di riferimento con quelli gia' esposti dal tool. Non include networking,
syscall puramente informative o eventi interni usati solo per il funzionamento
del tracer.

## Completati di recente

- `process_vm_readv`: lettura diretta della memoria di un processo;
- `process_vm_writev`: scrittura diretta nella memoria di un processo;
- `setns`: ingresso in un namespace esistente;
- `unshare`: separazione di risorse e creazione di namespace;
- `commit_creds`: credenziali realmente applicate al task;
- `switch_task_ns`: cambio effettivo dei namespace del task;
- `security_sb_mount` e `security_sb_umount`: mount e smontaggio risolti dal kernel;
- `security_inode_unlink`: cancellazione di file con path e inode risolti;
- `security_bpf`, `security_bpf_map` e `security_bpf_prog`: richieste e oggetti eBPF;
- `security_kernel_read_file` e `security_kernel_post_read_file`: file letti dal kernel;
- `security_file_ioctl`: controlli security su ioctl;
- `security_file_permission`: controlli di permesso su file gia' aperti;
- `cgroup_attach_task`: migrazione di task tra cgroup;
- `cgroup_mkdir` e `cgroup_rmdir`: creazione e rimozione di cgroup;
- `call_usermodehelper`: esecuzione di helper userspace avviata dal kernel;
- `do_sigaction`: cambio dei signal handler del processo;
- `module_load`, `module_free` e `do_init_module`: ciclo di vita dei moduli kernel;
- `process_execute_failed`: tentativi falliti di `execve`/`execveat`;
- `proc_create`: creazione di entry procfs da parte del kernel o di moduli;
- `register_kprobe`: registrazione dinamica di kprobe;
- `kallsyms_lookup_name`: lookup runtime di simboli kernel interni.

## Priorita' alta

1. `bpf_attach`: aggancio effettivo di programmi eBPF a hook o target runtime.
2. `debugfs_create_dir` e `__debugfs_create_file`: superfici debugfs esposte da
   moduli e componenti kernel.
3. `security_mmap_addr`: controllo security su indirizzi mmap sensibili.
4. `do_mmap`: dettaglio kernel-side dei mapping creati dal processo.
5. `load_elf_phdrs`: lettura degli header ELF durante il percorso di exec.

## Priorita' media

- `security_path_notify`;
- `set_fs_pwd`;
- syscall `capset`, `setgroups` e `seccomp`;
- `pidfd_send_signal`;

## Hardening kernel

Gia' coperti:

- `proc_create`;
- `register_kprobe`;
- `kallsyms_lookup_name`.

Restano da valutare:

- `bpf_attach`;
- `debugfs_create_dir`;
- `__debugfs_create_file`;
- logica di hidden module detection.

## Nota sul kernel target

Il simbolo separato `security_task_fix_setgid` non risulta disponibile sul
kernel Rocky Linux `4.18.0-553.109.1.el8_10.x86_64`. I cambi GID possono essere
osservati attraverso le syscall `setgid`, `setregid`, `setresgid` e `setfsgid`,
oppure tramite `commit_creds`, che mostra lo stato finale realmente applicato.

Prima di ogni implementazione devono essere verificati:

1. presenza del simbolo o del tracepoint sul kernel target;
2. layout effettivo del context;
3. necessita' del return value;
4. volume atteso dell'evento;
5. metadati necessari per future policy.
