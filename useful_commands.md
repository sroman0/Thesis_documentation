- Benchmark userspace ripetibile con soglia del 5%:

```bash
cd demo_project
DURATION_SECONDS=120 WARMUP_SECONDS=10 CPU_THRESHOLD=5.0 make benchmark-userspace
```

- Profili consigliati per confrontare il costo del tool:

```bash
# Terminale 1: avviare uno dei profili standard.
make benchmark-profile PROFILE=raw
make benchmark-profile PROFILE=point
make benchmark-profile PROFILE=collective
KERNEL_FILTER_UID=1000 make benchmark-profile PROFILE=kernel-filter-uid

# Terminale 2: misurare il processo project.
DURATION_SECONDS=120 WARMUP_SECONDS=10 CPU_THRESHOLD=5.0 make benchmark-userspace
```

Per confrontare correttamente i profili, usare lo stesso workload e completare
la durata configurata del benchmark. I primi secondi includono attach e
inizializzazione, quindi vanno letti come warm-up.

- Suite automatica con PID e workload controllati:

```bash
DURATION_SECONDS=120 \
WARMUP_SECONDS=10 \
CPU_THRESHOLD=5.0 \
make benchmark-suite
```

- Stress test con tutti gli eventi pubblici:

```bash
DURATION_SECONDS=120 \
WARMUP_SECONDS=10 \
CPU_THRESHOLD=5.0 \
make benchmark-all-events
```

Il profilo `all-events` non rappresenta la configurazione operativa consigliata
e puo' usare gran parte di un core. Serve a misurare capacita' e rumore; il
target `<5%` viene valutato sui profili policy-driven.

- Monitoring manuale rapido:

```bash
top -p "$(pgrep -n -x project)"
```

- Per osservare CPU/RSS/thread con `ps`:

```bash
watch -n 1 'ps -C project -o pid,%cpu,%mem,rss,nlwp,etime,cmd'
```

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

- Per testare policy, detector YAML, `--alerts-only` e dedup degli alert:

```bash
sudo ./dist/project \
  --policy rules/policies/demo-detectors.yaml \
  --detectors rules/detectors \
  --alerts-only \
  --alerts-output table \
  --log-level error
```

In un secondo terminale:

```bash
cat /etc/passwd
cat /etc/passwd
cat /etc/passwd
```

Output atteso: un solo alert `sensitive-file-open` nella finestra breve di
dedup. Dopo circa 5 secondi, lo stesso comando puo' generare un nuovo alert.

- Per testare il primo detector collective:

```bash
sudo ./dist/project \
  --policy rules/policies/collective-privilege-exec.yaml \
  --detectors rules/detectors/privilege_exec_chain.yaml \
  --alerts-only \
  --alerts-output table \
  --log-level error
```

In un secondo terminale:

```bash
sudo whoami
```

Output atteso:

```text
type=alert alert=Privilege change followed by process execution severity=high detector=privilege-exec-chain events=2 ... sequence=security_task_fix_setuid(...)->sched_process_exec(...)
```

- Per testare una catena collective con mapping MITRE visibile:

```bash
sudo ./dist/project \
  --policy rules/policies/collective-local-chains.yaml \
  --detectors rules/detectors/privilege_sensitive_file_chain.yaml \
  --alerts-only \
  --alerts-output table \
  --log-level error
```

In un secondo terminale:

```bash
sudo cat /etc/passwd
```

Output atteso: un alert `privilege-sensitive-file-chain` con `events=2`,
campo `sequence=...` e campo compatto `mitre=...`.

- Per testare la correlazione file `dev:inode` tra eventi:

```bash
sudo ./dist/project \
  --policy rules/policies/temp-script-write-exec.yaml \
  --detectors rules/detectors/temp_script_write_exec.yaml \
  --alerts-only \
  --alerts-output table \
  --log-level error
```

In un secondo terminale:

```bash
script="$(mktemp /tmp/vesuvius-resource-XXXXXX.sh)"
printf '#!/bin/sh\nid >/dev/null\n' > "$script"
chmod +x "$script"
"$script"
rm -f "$script"
```

Il detector usa i path soltanto come condizioni restrittive. La correlazione
richiede che `security_file_permission` e `security_bprm_check` espongano lo
stesso device e inode.
