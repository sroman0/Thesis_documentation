# Decoder Go degli eventi eBPF

## Scopo

Il decoder trasforma i record raw letti da ring buffer o perf buffer in
strutture Go leggibili e poi in output `json` o `table`.

Il flusso attuale e':

```text
raw event bytes
  -> bufferdecoder.DecodeEvent()
  -> EventContext + argomenti tipizzati
  -> output printer
```

## File principali

Percorso:

```text
demo_project/pkg/bufferdecoder/
```

File:

- `protocol.go`: definisce protocollo Go e layout del context evento;
- `decoder.go`: primitive di lettura binaria (`u8`, `u16`, `u32`, `u64`, bytes, context);
- `eventsreader.go`: decodifica evento completo e argomenti;
- `eventsreader_test.go`: test per scalari, stringhe, array di stringhe e
  payload strutturati.

## Responsabilita' dei file

### `protocol.go`

Definisce il contratto semantico tra eBPF e Go:

- dimensioni degli slot;
- `EventContext`;
- limiti di sicurezza per stringhe e array.

### `decoder.go`

E' il lettore binario di basso livello. Mantiene un cursore interno e legge
valori little-endian dal buffer:

- interi a 8/16/32/64 bit;
- byte array;
- `EventContext`.

Non decide quali argomenti appartengono a un evento: fornisce solo primitive di
lettura sicure.

### `eventsreader.go`

E' il livello semantico:

- crea un `EbpfDecoder`;
- legge il context;
- legge `argnum`;
- usa lo schema evento da `pkg/events/spec.go`;
- decodifica scalari, stringhe, sockaddr, puntatori, array di stringhe,
  argomenti NUL-delimited e credenziali strutturate;
- produce un `Event` completo.

## `EventContext`

Il context Go e' allineato al `event_context_t` eBPF del progetto, quindi
segue solo i campi realmente emessi dal runtime attuale.

Dimensione:

```text
136 byte
```

Layout:

```text
0   - 7    ts
8   - 111  task_context_t
112 - 115  event_id
116 - 119  syscall
120 - 123  stack_id
124 - 125  processor_id
126 - 127  policies_version
128 - 135  matched_policies
```

Nota importante: il context Go deve restare allineato al layout eBPF locale.
Ogni modifica a `event_context_t` o `task_context_t` richiede quindi un
aggiornamento esplicito del decoder.

### Debug recente: eventi corretti ma `args=-`

Durante l'integrazione policy/detector e' emerso un bug subdolo: gli eventi
arrivavano dal perf buffer, il nome evento era corretto, ma l'output mostrava
sempre:

```text
args=-
```

Il problema non era negli hook eBPF. La causa era nel contratto binario C/Go:
`protocol.go` dichiarava `eventContextSize = 128`, mentre il `event_context_t`
attuale e' lungo 136 byte perche' include anche `policies_version` e
`matched_policies`.

Con `eventContextSize` errato, `DecodeContext` leggeva correttamente i campi
fino a `matched_policies`, ma avanzava il cursore solo di 128 byte. Il byte
successivo, interpretato come `argnum`, veniva quindi letto dentro
`matched_policies` invece che subito dopo il context. Il risultato era
`argnum=0` e quindi nessun argomento decodificato.

Fix applicato:

- `demo_project/pkg/bufferdecoder/protocol.go`: `eventContextSize` portato da
  `128` a `136`;
- commenti del decoder aggiornati al layout reale;
- documentazione tecnica aggiornata per evitare nuovi disallineamenti.

Sintomo utile per riconoscere questo tipo di bug:

```text
event=<nome corretto> ... args=-
```

Se il nome evento e' corretto ma tutti gli argomenti spariscono, controllare
prima la dimensione del context e il punto in cui il decoder legge `argnum`.

## Argomenti

Dopo il context, il formato inviato attraverso il canale eventi e':

```text
[event_context_t:136][argnum:u8][args...]
```

Scalari:

```text
[index:u8][value]
```

Stringhe:

```text
[index:u8][length:u32][payload]
```

Array di stringhe (`StrArrT`):

```text
[index:u8][count:u8][size:int32][string bytes]...
```

Array di argomenti compatti (`ArgsArrT`):

```text
[index:u8][payload_len:int32][reported_count:int32][NUL-delimited strings]
```

`eventsreader.go` usa lo schema statico degli eventi per sapere come leggere gli
argomenti. Esempi:

- `cap_capable`: `cap` come `INT_T`;
- `sched_process_exec`: `filename` come `STR_T`;
- `execve`: `pathname` come `STR_T`, `argv` come `STR_ARR_T`;
- `execveat`: `dirfd`, `pathname`, `flags`, `argv`;
- `sched_process_exit`: `exit_code` come `LONG_T`, `group_dead` come `U8_T`;
- `commit_creds`: `old_cred` e `new_cred` come `CRED_T`;
- `security_socket_connect`: indirizzo remoto come `SOCKADDR_T`;
- `proc_create`, `register_kprobe` e `kallsyms_lookup_name`: puntatori kernel
  come `POINTER_T`.

Il limite massimo per le singole stringhe e' stato allineato al lato eBPF
(`MAX_STRING_SIZE`, 4096 byte). Questo evita che path o argomenti validi
scritti dal kernel vengano rifiutati artificialmente dal decoder Go.

## Runtime

In `demo_project/pkg/ebpf/project.go`, il loop runtime:

1. legge un record da ring buffer o perf buffer;
2. chiama `bufferdecoder.DecodeEvent(record.RawSample)`;
3. applica il filtro eventi runtime;
4. passa l'evento al printer configurato in `pkg/output`;
5. stampa una riga su stdout.

Esempio osservato:

```json
{"timestamp":4074609472044753,"event_name":"cap_capable","process":{"pid":1374462,"tid":1374462,"ppid":1348641,"uid":1000,"comm":"cpuUsage.sh","uts_name":"security-thesis"},"host":{"pid":1374462,"tid":1374462,"ppid":1348641},"kernel":{"syscall":56,"processor_id":0,"mnt_id":4026531840,"pid_id":4026531836},"args":[{"name":"cap","type":1,"value":21,"label":"CAP_SYS_ADMIN"}]}
```

Questo conferma la pipeline:

```text
kprobe/cap_capable
  -> evento eBPF
  -> perf buffer
  -> decoder Go
  -> output layer
```

## Array di stringhe

Il supporto agli array di stringhe serve soprattutto per argomenti come
`argv`. Il writer eBPF `save_str_arr_to_buf` legge una lista `char **` da
userspace, salva il numero di elementi e poi serializza ogni stringa con la sua
dimensione.

Il decoder Go:

- controlla che il numero di elementi non sia eccessivo;
- valida ogni dimensione prima di leggere il payload;
- rimuove il terminatore `NUL`;
- verifica UTF-8;
- restituisce un `[]string`.

Questo rende possibile stampare eventi come:

```text
event=execve ... args=pathname=/usr/bin/sh,argv=["sh","-c","id"]
```

Per ora non viene serializzato `envp`: e' spesso molto rumoroso e puo'
contenere dati sensibili. La scelta e' coerente con una demo piu' leggibile e
con un payload piu' controllato.

## Stringhe NUL-delimited

Il decoder contiene anche un parser per payload compatti in cui piu' stringhe
sono salvate in un solo blocco e separate dal byte `NUL`:

```text
sh\0-c\0id\0
```

La funzione `splitNullDelimitedStrings` scorre il payload, taglia una stringa
ogni volta che trova `NUL` e rispetta un limite massimo di elementi. Questo
limite evita che un valore `reported_count` errato produca parsing eccessivo.
Se l'ultima stringa non e' terminata da `NUL`, il suffisso restante viene
comunque decodificato.

Ogni elemento passa poi da `decodeStringPayload`, che:

- rimuove eventuali terminatori `NUL` finali;
- verifica che il risultato sia UTF-8 valido;
- restituisce una stringa Go pulita al livello di output.

Questa separazione rende il codice piu' semplice: una funzione si occupa di
separare il blocco in elementi, l'altra di normalizzare una singola stringa.

## Limitazioni

- Il decoder resta responsabile del formato binario, non della presentazione.
- `comm`, `uts_name` e capability vengono normalizzati nel package `output`.
- `cap_capable` e' molto rumoroso e puo' generare molte righe.
- Il decoder supporta gli eventi attuali; nuovi hook richiedono una voce in
  `pkg/events/spec.go`.
- `security_bprm_check` espone `argc`/`envc`, ma non ancora `argv`/`envp`:
  farlo bene richiede una decisione su dipendenze tra hook syscall e hook LSM.
- Gli eventi kernel-hardening possono esporre puntatori utili alla correlazione,
  ma il decoder non prova a risolverli in simboli: questa responsabilita'
  rimane all'output o a una futura fase di enrichment.

## Verifiche

Test Go:

```bash
GOCACHE=/tmp/go-build go test ./pkg/bufferdecoder
```

Test con package collegati:

```bash
PKG_CONFIG_PATH=./dist/libbpf/obj \
CGO_CFLAGS="-I/home/simone/project/demo_project/dist/libbpf/include -I/home/simone/project/demo_project/3rdparty/libbpfgo" \
CGO_LDFLAGS="-L/home/simone/project/demo_project/dist/libbpf/obj -lbpf" \
GOCACHE=/tmp/go-build \
go test ./pkg/bufferdecoder ./pkg/output ./pkg/detectors ./pkg/detectors/yaml ./pkg/policy ./pkg/ebpf ./pkg/cmd
```

Test completo:

```bash
make bpf
sudo go run ./cmd/project --bpf-object pkg/ebpf/c/project.bpf.o
```

Poi, in un altro terminale:

```bash
ls
whoami
sleep 1
echo test
```

## Collegamenti

- [Overview implementazione](overview.md)
- [Protocollo eventi e buffer eBPF](event-buffer.md)
- [Userspace Go e lifecycle eBPF](userspace-lifecycle.md)
- [Diario 2026-05-04](../daily/2026-05-04.md)
