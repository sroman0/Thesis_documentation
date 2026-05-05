# Decoder Go degli eventi eBPF

## Scopo

Il decoder trasforma i record raw letti dalla ring buffer in strutture Go
leggibili e poi in JSON.

Il flusso attuale e':

```text
ringbuf.Record.RawSample
  -> bufferdecoder.DecodeEvent()
  -> EventContext + argomenti tipizzati
  -> json.Marshal()
  -> stdout
```

## File principali

Percorso:

```text
demo_project/pkg/bufferdecoder/
```

File:

- `protocol.go`: definisce protocollo Go, ID eventi, tipi argomento e schema argomenti;
- `decoder.go`: primitive di lettura binaria (`u8`, `u16`, `u32`, `u64`, bytes, context);
- `eventsreader.go`: decodifica evento completo e argomenti;
- `eventsreader_test.go`: test minimi per eventi scalari e stringa.

## Responsabilita' dei file

### `protocol.go`

Definisce il contratto semantico tra eBPF e Go:

- dimensioni degli slot;
- ID eventi;
- tipi argomento;
- `EventContext`;
- schema statico `EventID -> nome evento + argomenti`.

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
- usa lo schema evento da `protocol.go`;
- decodifica slot scalari o stringa;
- produce un `Event` completo.

## `EventContext`

Il context Go e' allineato al `event_context_t` eBPF del progetto, non a quello
completo di Tracee.

Dimensione:

```text
128 byte
```

Layout:

```text
0   - 7    ts
8   - 111  task_context_t
112 - 115  event_id
116 - 119  syscall
120 - 123  stack_id
124 - 125  processor_id
126 - 127  pad
```

Differenza importante rispetto a Tracee:

- Tracee usa un context da `136` byte;
- Tracee include `policies_version` e `matched_policies`;
- il progetto li omette per MVP;
- quindi il decoder non puo' copiare pari pari `tracee/pkg/bufferdecoder/protocol.go`.

## Argomenti

Dopo il context, il formato inviato sulla ring buffer e':

```text
[event_context_t:128][argnum:u8][args...]
```

Gli argomenti usano gli slot fissi introdotti per rendere il codice accettabile
dal verifier.

Scalari:

```text
slot = 16 byte
[type_tag:u8][value][padding...]
```

Stringhe:

```text
slot = 517 byte
[type_tag:u8][length:u32][payload:512]
```

`eventsreader.go` usa lo schema statico degli eventi per sapere come leggere gli
argomenti. Esempio:

- `cap_capable`: `cap` come `INT_T`;
- `sched_process_exec`: `filename` come `STR_T`;
- `sched_process_exit`: `exit_code` come `LONG_T`, `group_dead` come `U8_T`.

## Runtime

In `demo_project/pkg/ebpf/project.go`, il loop runtime:

1. legge un record dalla ring buffer;
2. chiama `bufferdecoder.DecodeEvent(record.RawSample)`;
3. serializza l'evento con `json.Marshal`;
4. stampa una riga su stdout.

Esempio osservato:

```json
{"event_name":"cap_capable","args":[{"name":"cap","type":1,"value":2}]}
```

Questo conferma la pipeline:

```text
kprobe/cap_capable
  -> evento eBPF
  -> ring buffer
  -> decoder Go
  -> JSON
```

## Limitazioni

- `Comm` e `UtsName` sono ancora serializzati come array di byte nel JSON.
- `cap` viene stampato come numero, non ancora come nome capability.
- `cap_capable` e' molto rumoroso e puo' generare molte righe.
- Il decoder supporta gli eventi attuali; nuovi hook richiedono una voce nello
  schema statico in `protocol.go`.
- Gli eventi con argomenti misti stringa + scalare richiedono attenzione per
  evitare sovrapposizioni tra slot di dimensione diversa.

## Verifiche

Test Go:

```bash
GOCACHE=/tmp/go-build go test ./pkg/bufferdecoder
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
