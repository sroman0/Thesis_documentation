- Monitoring: top -p $(pgrep -n -x project)
- Per misurare la media di prestazioni in ps dall'avvio del tool: watch -n 1 'ps -C project -o pid,%cpu,%mem,rss,nlwp,etime,cmd'

- Per misurare il consumo istantaneo: sudo top -d 1 -p "$(pgrep -n project)"

- Per elencare gli eventi disponibili:

```bash
./dist/project --list-events
```

- Per testare insieme gli eventi namespace/filesystem:

```bash
sudo ./dist/project \
  --events switch_task_ns,security_sb_mount,security_sb_umount,security_inode_unlink \
  --output table
```

- Trigger controllati:

```bash
sudo unshare -m true
sudo mkdir -p /tmp/project-mount-test
sudo mount -t tmpfs -o nosuid,nodev tmpfs /tmp/project-mount-test
sudo umount /tmp/project-mount-test
sudo rmdir /tmp/project-mount-test
touch /tmp/project-unlink-test
rm /tmp/project-unlink-test
```

- Per testare un profilo process/security più aggiornato:

```bash
sudo ./dist/project \
  --events execve,execveat,process_execute_failed,open,chmod,chown,security_bprm_check \
  --output table \
  --log-level error
```

- Per osservare eventi di hardening kernel:

```bash
sudo ./dist/project \
  --events security_bpf,security_bpf_map,security_bpf_prog,module_load,module_free,do_init_module,proc_create,register_kprobe,kallsyms_lookup_name \
  --output table \
  --log-level error
```

Nota: `kallsyms_lookup_name` stampa solo se codice kernel o un modulo chiama
realmente quella funzione.

- Per avviare un profilo file/memoria ad alto volume, usare sempre un set
ristretto e, se possibile, `--comms`:

```bash
sudo ./dist/project \
  --events security_file_open,security_file_permission,mmap,mprotect,process_vm_readv,process_vm_writev \
  --comms bash,cat,gcc,cc1 \
  --output table \
  --log-level error
```
