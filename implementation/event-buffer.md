# Protocollo eventi e buffer eBPF

## Scopo

Il buffer eventi serve a trasportare dati dal kernel allo userspace.

Ogni evento contiene:

```text
event_context_t
argnum
args buffer
```

Nel runtime corrente il record arriva dal perf buffer `events`. La precedente
mappa `events_ringbuf`, la helper `events_ringbuf_submit` e il relativo reader
Go sono stati rimossi: nessun hook attivo li usava e il kernel Rocky Linux 4.18
target rende il perf buffer la scelta piu' compatibile.

Il decoder Go interpreta i byte raw con questo layout:

```text
[event_context_t:136][argnum:u8][args...]
```

## `save_to_submit_buf`

Serializza argomenti scalari come:

- `u8`;
- `u16`;
- `u32`;
- `u64`.

Ogni record contiene prima l'indice logico dell'argomento e poi il payload:

```text
[index:u8][value]
```

Il tipo e la dimensione vengono risolti in userspace usando
`pkg/events/spec.go`.

## `save_str_to_buf`

Serializza argomenti stringa come record a dimensione variabile:

```text
[index:u8][length:int32][payload]
```

La lunghezza reale viene salvata nel campo `length`.

## Perche' il formato e' indicizzato?

Il writer eBPF usa `buf->offset` per appendere record al buffer degli
argomenti. Ogni record porta il proprio indice (`index`), quindi gli argomenti
possono essere decodificati e poi riordinati secondo lo schema Go.

Questo formato evita di dipendere dall'ordine fisico di scrittura e permette a
un hook di saltare un argomento opzionale senza rompere lo schema degli altri.

In passato il progetto ha sperimentato slot fissi per semplificare il lavoro
del verifier. La versione corrente usa invece record indicizzati compatti, con
controlli di bounds nelle helper eBPF.

```text
invalid access to map value, value_size=4280 off=65664
```

Errore tipico quando i bounds non sono dimostrabili:

```text
invalid access to map value, value_size=4280 off=65664
```

## Trade-off

Vantaggi:

- formato piu' compatto degli slot fissi;
- argomenti opzionali piu' semplici da gestire;
- contratto esplicito tramite `pkg/events/spec.go`.

Svantaggi:

- richiede massima parita' tra layout C e decoder Go;
- ogni nuovo tipo argomento deve essere supportato esplicitamente dal decoder;
- errori nella dimensione del context possono spostare la lettura di `argnum`.

## Decoder userspace

Il package `demo_project/pkg/bufferdecoder` implementa la parte userspace del
protocollo:

- `protocol.go`: layout Go e schema degli eventi;
- `decoder.go`: primitive binarie e `DecodeContext`;
- `eventsreader.go`: decoding completo degli eventi e argomenti.

Il context corrente e' di 136 byte e include:

- task context;
- event id;
- syscall id;
- stack id;
- processor id;
- `policies_version`;
- `matched_policies`.

Nota di debug: se il decoder usa una dimensione errata del context, puo'
leggere `argnum` dalla posizione sbagliata. Il sintomo osservato e' output con
nome evento corretto ma `args=-`.

## Stato verificato

E' stato osservato output JSON da eventi reali, ad esempio `cap_capable`:

```json
{"event_name":"cap_capable","args":[{"name":"cap","type":1,"value":2}]}
```

Questo conferma che la pipeline kernel -> buffer eventi -> decoder Go -> JSON
e' operativa.

## Canale kernel/userspace

La versione attuale mantiene una sola mappa eventi:

```c
struct events {
    __uint(type, BPF_MAP_TYPE_PERF_EVENT_ARRAY);
} events SEC(".maps");
```

Lato C `events_perf_submit` usa `bpf_perf_event_output`.

Lato Go, `Project.Init()` apre un solo transport:

```go
module.InitPerfBuf("events", ..., 1024)
```

Il canale perf termina in `handleRawEvent(raw []byte)`, che esegue:

1. decode del payload;
2. filtro evento;
3. filtro `comm`;
4. output.

Tutti gli hook pubblici usano `events_perf_submit`. Anche il canale degli eventi
persi resta associato al perf buffer, quindi il runtime mantiene un solo
percorso operativo e una sola semantica di backpressure.

## Correlazione syscall entry/exit

Gli eventi che richiedono il valore di ritorno non vengono inviati direttamente
dal tracepoint `sys_enter_*`. L'entry salva gli argomenti in `args_map`, usando
una chiave composta da:

```text
event ID + thread ID
```

Il tracepoint `sys_exit_*` usa la stessa chiave per:

1. recuperare gli argomenti;
2. eliminare la voce temporanea dalla mappa;
3. aggiungere il return value;
4. serializzare e inviare un solo evento completo.

Il pattern e' usato, tra gli altri, da:

- `open`;
- `chmod`;
- `chown`;
- `memfd_create`;
- `mmap`;
- `mprotect`;
- `pkey_mprotect`;
- `process_vm_readv`;
- `process_vm_writev`;
- `setns`;
- `unshare`.

La struttura condivisa `args_t` contiene sette slot `unsigned long`. Il settimo
slot e' necessario per le syscall con piu' metadati, in particolare `chown`.
Anche `load_args` copia tutti e sette gli slot dalla mappa alla struttura
locale.

Questo modello evita due eventi separati per la stessa syscall e consente al
decoder di ricevere direttamente una rappresentazione che include l'esito
finale.

## Eventi di gestione memoria

I nuovi eventi usano lo stesso protocollo degli altri record perf buffer:

```text
mmap:
  addr, length, prot, flags, fd, offset, returnValue

mprotect:
  addr, length, prot, returnValue

pkey_mprotect:
  addr, length, prot, pkey, returnValue

process_vm_writev:
  pid, local_iov, liovcnt, remote_iov, riovcnt, flags, returnValue

process_vm_readv:
  pid, local_iov, liovcnt, remote_iov, riovcnt, flags, returnValue

setns:
  fd, nstype, returnValue

unshare:
  flags, returnValue

switch_task_ns:
  pid, new_mnt?, new_pid?, new_uts?, new_ipc?, new_net?, new_cgroup?

security_sb_mount:
  dev_name?, path, type?, flags

security_sb_umount:
  mountpoint, type?, flags

security_inode_unlink:
  pathname, inode, dev, ctime
```

Gli indirizzi e i campi `unsigned long` vengono serializzati come valori a 64
bit sul target x86_64. `returnValue` resta signed per conservare gli errno
negativi restituiti dal kernel.

Il punto interrogativo indica un argomento opzionale. In particolare,
`switch_task_ns` serializza solo gli ID diversi dal vecchio `nsproxy`; il
decoder usa gli indici registrati nel record e mantiene comunque il corretto
nome del campo.

## Credenziali strutturate

`commit_creds` usa due record `CredT` da 80 byte:

```text
commit_creds:
  old_cred, new_cred
```

Ogni record contiene UID/GID, effective/saved/filesystem ID, user namespace,
securebits e cinque maschere capability. Il decoder Go legge esplicitamente il
layout little-endian di `slim_cred_t`, evitando cast dipendenti dalla memoria
del processo userspace.

## Collegamenti

- [Approfondimento: scelta tra perf buffer e ring buffer](perf-buffer-vs-ring-buffer.html)
- [Timeline](../timeline.md)
- [Debugging verifier](../debugging/ebpf-verifier.md)
- [Overview implementazione](overview.md)
- [Decoder Go](decoder.md)
