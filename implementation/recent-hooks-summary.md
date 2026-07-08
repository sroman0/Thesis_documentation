# Sintesi ultimi hook implementati

Questo documento raccoglie una sintesi compatta degli ultimi gruppi di hook
aggiunti al tool. La descrizione dettagliata resta in
[`hooks.md`](hooks.md), mentre i comandi di test sono in
[`commands.md`](../debugging/commands.md).

- `security_bpf_map`: osserva operazioni security su mappe eBPF già esistenti.
- `security_bpf_prog`: osserva operazioni security su programmi eBPF già caricati.
- `security_kernel_post_read_file`: osserva il completamento di letture file fatte dal kernel.
- `security_file_ioctl`: osserva richieste `ioctl` su file o device.
- `security_file_permission`: osserva controlli di permesso su file già aperti.
- `cgroup_attach_task`: osserva lo spostamento di task tra cgroup.
- `cgroup_mkdir`: osserva la creazione di un cgroup.
- `cgroup_rmdir`: osserva la rimozione di un cgroup.
- `call_usermodehelper`: osserva helper userspace avviati dal kernel.
- `do_sigaction`: osserva modifiche agli handler dei segnali.
- `module_load`: osserva il caricamento di moduli kernel.
- `module_free`: osserva la rimozione/liberazione di moduli kernel.
- `do_init_module`: osserva l’inizializzazione di un modulo kernel, incluso il return value.
- `process_execute_failed`: osserva tentativi falliti di `execve`/`execveat`.
- `proc_create`: intercetta la creazione di entry in /proc, utile per moduli che espongono interfacce di controllo.
- `register_kprobe`: intercetta la registrazione dinamica di kprobe, con symbol_name, handler e returnValue.
- `kallsyms_lookup_name`: intercetta lookup di simboli kernel, con simbolo cercato e indirizzo restituito.

Questa lista completa la copertura piu' recente lato BPF object, cgroup,
moduli, signal handling, exec fallite e hardening kernel. Per la lista completa
degli eventi supportati usare:

```bash
./dist/project --list-events
```
