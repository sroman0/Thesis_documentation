# Output layer

## Obiettivo

Il package `demo_project/pkg/output` separa la presentazione degli eventi dal
runtime eBPF. `Project.Run()` legge i record raw dai canali eBPF aperti dal
runtime, oggi principalmente perf buffer, decodifica e filtra; il package output
decide come trasformare l'evento in una riga stampabile.

Questa scelta riprende in forma ridotta il ruolo del sink stage di Tracee: la
pipeline non mescola piu' lettura kernel, decoding e formattazione finale.
La versione corrente prepara anche un modello separato per gli alert dei
detector, ma questi alert non sono ancora collegati al runtime eBPF.

## File principali

- `printer.go`: definisce l'interfaccia `Printer` e la factory `NewPrinter`.
- `json.go`: stampa eventi JSON line-oriented.
- `table.go`: stampa eventi compatti per debug manuale.
- `event.go`: costruisce una vista normalizzata dell'evento e arricchisce alcuni
  argomenti.
- `alert.go`: costruisce una vista normalizzata degli alert prodotti dai
  detector.
- `json_test.go`, `table_test.go` e `alert_test.go`: testano i formati output.

## Formato JSON

Il formato JSON non serializza piu' direttamente `bufferdecoder.Event`. Prima
campi come `comm` e `uts_name` apparivano come array di byte. Ora `json.go`
serializza una vista normalizzata prodotta da `newEventRecord()`.

Esempio compatto:

```json
{"timestamp":4074609472044753,"event_name":"cap_capable","process":{"pid":1374462,"tid":1374462,"ppid":1348641,"uid":1000,"comm":"cpuUsage.sh","uts_name":"security-thesis"},"host":{"pid":1374462,"tid":1374462,"ppid":1348641},"kernel":{"syscall":56,"processor_id":0,"mnt_id":4026531840,"pid_id":4026531836},"args":[{"name":"cap","type":1,"value":21,"label":"CAP_SYS_ADMIN"}]}
```

Miglioramenti principali:

- `comm` e `uts_name` sono stringhe;
- i metadati sono divisi in blocchi `process`, `host` e `kernel`;
- l'argomento `cap` viene arricchito con `label`, ad esempio
  `CAP_SYS_ADMIN`.

## Formato table

Il formato `table` e' pensato per uso interattivo da terminale.

Esempio:

```text
event=cap_capable pid=1374462 tid=1374462 uid=1000 comm=cpuUsage.sh args=cap=CAP_SYS_ADMIN(21)
```

Per gli eventi di signal delivery il numero del segnale viene arricchito nello
stesso modo:

```text
event=security_task_kill ... args=target_comm=sleep,signal=SIGTERM(15)
```

Anche le richieste `ptrace` vengono rese piu' leggibili:

```text
event=ptrace ... args=request=PTRACE_ATTACH(16),pid=1234,addr=0,data=0,returnValue=0
```

Per gli eventi file, il formato resta volutamente compatto:

```text
event=security_file_open ... args=pathname=/etc/hostname,flags=32768,dev=...,inode=...,ctime=...,syscall_pathname=/etc/hostname
```

Questo formato sacrifica alcuni metadati dettagliati per rendere l'output
leggibile mentre il tracer gira.

## Eventi di gestione memoria

Per `mmap`, `mprotect` e `pkey_mprotect`, il layer di output converte le
bitmask numeriche in nomi Linux:

```text
prot=PROT_READ|PROT_WRITE
flags=MAP_PRIVATE|MAP_ANONYMOUS
```

Il valore restituito da `mmap` viene mostrato come indirizzo esadecimale:

```text
returnValue=address:0x7f001000
```

Un errore mantiene sia il nome errno sia il valore kernel:

```text
returnValue=ENOMEM(-12): cannot allocate memory
```

Le syscall `mprotect` e `pkey_mprotect`, che restituiscono zero in caso di
successo, vengono mostrate come:

```text
returnValue=success
```

I bit non ancora riconosciuti non vengono persi: sono mantenuti in forma
esadecimale accanto alle flag note.

Per `process_vm_readv` e `process_vm_writev`, i puntatori agli array `iovec`
vengono mostrati in esadecimale e il risultato positivo indica il numero di
byte trasferiti:

```text
local_iov=0x7fa94e5dc3c8
remote_iov=0x7fa94e5dc450
returnValue=bytes:13
```

Un risultato negativo continua a usare il mapping errno comune, ad esempio
`EPERM(-1): operation not permitted`.

Per `commit_creds`, l'output table mantiene separate le vecchie e nuove
credenziali:

```text
old_cred={uid:1000,...,euid:1000,...,cap_effective:0x0}
new_cred={uid:1000,...,euid:0,...,cap_effective:0x1ffffffffff}
```

Nel formato JSON entrambi i valori sono oggetti strutturati con nomi
snake_case, quindi possono essere consumati direttamente da una futura policy
senza parsing testuale.

Per `setns`, il tipo numerico del namespace viene convertito nel corrispondente
nome Linux:

```text
nstype=CLONE_NEWNS
returnValue=success
```

Il valore zero viene mostrato come `AUTO`, per indicare che il kernel ricava il
tipo di namespace dal file descriptor. I valori non riconosciuti restano
visibili in esadecimale, mentre gli errori usano il mapping errno comune.

Per `unshare`, l'output interpreta la bitmask completa e puo' mostrare piu'
risorse nella stessa riga:

```text
flags=CLONE_NEWUSER|CLONE_NEWNS
returnValue=success
```

Sono riconosciuti sia i flag namespace `CLONE_NEW*` sia `CLONE_FS`,
`CLONE_FILES` e `CLONE_SYSVSEM`. I bit sconosciuti vengono mantenuti in
esadecimale.

## Namespace e filesystem

`switch_task_ns` non ripete tutti i namespace del processo. Mostra il PID del
task e soltanto gli ID realmente sostituiti:

```text
event=switch_task_ns ... args=pid=3550453,new_mnt=4026532233
```

Questo rende l'evento adatto alla correlazione con `setns` e `unshare`: i due
eventi syscall mostrano richiesta ed esito, mentre `switch_task_ns` descrive la
transizione effettiva applicata dal kernel.

Per mount e umount, le bitmask numeriche vengono convertite in nomi Linux:

```text
event=security_sb_mount ... args=dev_name=tmpfs,path=/tmp/project-mount-test,type=tmpfs,flags=MS_NOSUID|MS_NODEV
event=security_sb_umount ... args=mountpoint=/tmp/project-mount-test,type=tmpfs,flags=0
```

Sono riconosciute le principali flag `MS_*`, `MNT_*` e `UMOUNT_*`. Eventuali
bit non noti restano visibili in esadecimale.

`security_inode_unlink` non richiede arricchimenti simbolici, ma fornisce
direttamente metadati stabili del file risolto dal kernel:

```text
event=security_inode_unlink ... args=pathname=/tmp/project-unlink-test,inode=...,dev=...,ctime=...
```

## Hardening kernel

Gli eventi di hardening kernel espongono spesso indirizzi di funzioni o
strutture. Il formato `table` li mostra in esadecimale per evitare ambiguita'
con normali interi:

```text
event=proc_create ... args=name=example,proc_ops_addr=0xffffffffc0123450
event=kallsyms_lookup_name ... args=symbol_name=printk,address=0xffffffff810...
```

Per `register_kprobe`, il return value viene trattato come stato operativo:

```text
event=register_kprobe ... args=symbol_name=cap_capable,pre_handler=0xffffffffc...,post_handler=0x0,returnValue=success
```

Un errore negativo continua a usare il mapping errno comune, ad esempio:

```text
returnValue=ENOENT(-2): no such file or directory
```

Questi eventi non sono pensati per essere letti da soli: diventano piu'
informativi quando vengono correlati con caricamento moduli, creazione di
entry procfs, lookup di simboli kernel e futura logica su debugfs o hidden
modules.

## Output alert

Gli eventi raw e gli alert hanno significati diversi:

- un evento raw descrive un fatto osservato dal kernel;
- un alert descrive una decisione prodotta da un detector.

Per evitare di mescolare questi due livelli, e' stato introdotto `AlertRecord`
in `alert.go`. Il record contiene:

- `type=alert`;
- `detector_id` e `detector_name`;
- `policy_names`, per ora ricavati dai metadata dell'alert quando presenti;
- titolo, descrizione e severita';
- timestamp di creazione;
- numero di eventi correlati;
- eventi correlati normalizzati con lo stesso schema JSON degli eventi raw;
- metadata aggiuntivi.

La vista table e' intenzionalmente compatta:

```text
alert=Privilege change followed by exec severity=medium detector=setuid-exec-chain events=2 policies=local-collective
```

Questo step prepara il formato, ma non cambia ancora l'interfaccia `Printer`.
Il prossimo passaggio sara' aggiungere un metodo dedicato agli alert in
`JSONPrinter` e `TablePrinter`.

## Test

I test del package output verificano:

- che il JSON prodotto sia valido;
- che `comm` venga convertito da array C-style a stringa;
- che capability numeriche come `21` vengano convertite in `CAP_SYS_ADMIN`;
- che segnali numerici come `15` vengano convertiti in `SIGTERM`;
- che richieste ptrace come `16` vengano convertite in `PTRACE_ATTACH`;
- che i permessi memoria vengano convertiti in `PROT_*`;
- che le flag di mapping vengano convertite in `MAP_*`;
- che indirizzi, successi ed errno delle syscall di memoria siano leggibili;
- che `process_vm_writev` mostri puntatori esadecimali e byte scritti;
- che le flag mount e umount vengano convertite nei nomi simbolici;
- che puntatori kernel e indirizzi di callback vengano mostrati in
  esadecimale;
- che il return value degli eventi di registrazione kernel venga mostrato come
  successo o errno;
- che gli alert detector vengano convertiti in record separati dagli eventi raw;
- che la vista table degli alert mantenga detector, severita', policy e numero
  di eventi correlati;
- che il formato table contenga i campi essenziali;
- che formati non supportati vengano rifiutati dalla factory.

Comando:

```bash
GOCACHE=/tmp/go-build go test ./pkg/output -v
```

## Limiti attuali

Il mapping simbolico e' presente per Linux capabilities, segnali POSIX comuni,
richieste `ptrace`, errno, file descriptor, eventi eBPF, tipi `READING_*`,
permessi `MAY_*`, modalita' `UMH_*` e flag relative a file, memoria, namespace,
mount e umount. Altri valori numerici, come syscall, alcune opzioni `prctl`,
socket option o comandi driver-specific, sono ancora stampati come numeri e
potranno essere arricchiti nello stesso layer.

Esempio:

```text
event=security_task_prctl pid=... comm=ls args=option=23,arg2=32,arg3=0,arg4=...,arg5=...
```

In questo caso `option=23` e' una costante `prctl` e dovrebbe essere mostrata
come label simbolica, ad esempio `PR_SET_VMA`, quando verra' aggiunto il mapping
dedicato.

Per i test interattivi conviene ridurre il rumore selezionando solo gli eventi
necessari:

```bash
make run ARGS="--events task_rename,sched_process_exec,sched_process_exit --output table"
```

Oppure escludere eventi rumorosi:

```bash
make run ARGS="--drop-events cap_capable --output table"
```

La versione attuale supporta anche un filtro userspace sul campo `comm`:

```bash
make run ARGS="--events task_rename,sched_process_exec,sched_process_exit --comms ls,whoami --output table"
```

Il filtro viene applicato dopo il decode, quindi migliora la leggibilita'
dell'output ma non riduce ancora il numero di eventi prodotti lato kernel.

## Collegamenti

- [Userspace lifecycle](userspace-lifecycle.md)
- [Comandi utili](../debugging/commands.md)
- [Timeline](../timeline.md)
