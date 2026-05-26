# Output layer

## Obiettivo

Il package `demo_project/pkg/output` separa la presentazione degli eventi dal
runtime eBPF. `Project.Run()` legge dalla ring buffer, decodifica e filtra; il
package output decide come trasformare l'evento in una riga stampabile.

Questa scelta riprende in forma ridotta il ruolo del sink stage di Tracee: la
pipeline non mescola piu' lettura kernel, decoding e formattazione finale.

## File principali

- `printer.go`: definisce l'interfaccia `Printer` e la factory `NewPrinter`.
- `json.go`: stampa eventi JSON line-oriented.
- `table.go`: stampa eventi compatti per debug manuale.
- `event.go`: costruisce una vista normalizzata dell'evento e arricchisce alcuni
  argomenti.
- `json_test.go` e `table_test.go`: testano i formati output.

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

Questo formato sacrifica alcuni metadati dettagliati per rendere l'output
leggibile mentre il tracer gira.

## Test

I test del package output verificano:

- che il JSON prodotto sia valido;
- che `comm` venga convertito da array C-style a stringa;
- che capability numeriche come `21` vengano convertite in `CAP_SYS_ADMIN`;
- che segnali numerici come `15` vengano convertiti in `SIGTERM`;
- che il formato table contenga i campi essenziali;
- che formati non supportati vengano rifiutati dalla factory.

Comando:

```bash
GOCACHE=/tmp/go-build go test ./pkg/output -v
```

## Limiti attuali

Il mapping simbolico e' presente per Linux capabilities e segnali POSIX comuni.
Altri valori numerici, come syscall, resource limit o opzioni `prctl`, sono
ancora stampati come numeri e potranno essere arricchiti nello stesso layer.

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
