# Protocollo eventi e buffer eBPF

## Scopo

Il buffer eventi serve a trasportare dati dal kernel allo userspace.

Ogni evento contiene:

```text
event_context_t
argnum
args buffer
```

Nel runtime userspace il record puo' arrivare da due canali:

- ring buffer `events_ringbuf`;
- perf buffer `events`.

Entrambi trasportano lo stesso payload logico. Il decoder Go interpreta i byte
raw con questo layout:

```text
[event_context_t:128][argnum:u8][args...]
```

## `save_to_submit_buf`

Serializza argomenti scalari come:

- `u8`;
- `u16`;
- `u32`;
- `u64`.

Per rendere il codice accettabile dal verifier, usa slot fissi:

```text
slot scalare = 16 byte
```

Layout concettuale:

```text
[type_tag][value][padding...]
```

Il decoder legge questi slot con dimensione costante:

```text
offset arg N = N * 16
```

## `save_str_to_buf`

Serializza argomenti stringa usando slot fissi:

```text
slot stringa = 1 byte tag + 4 byte length + 512 byte payload
```

La lunghezza reale viene salvata nel campo `length`.

Il decoder legge questi slot con dimensione costante:

```text
offset arg N = N * 517
```

## Perche' slot fissi?

La serializzazione compatta originale usava `buf->offset` come offset dinamico.

Il verifier vedeva `offset` come potenzialmente fino a `65535`, generando errori:

```text
invalid access to map value, value_size=4280 off=65664
```

Gli slot fissi rendono gli offset dimostrabili:

```text
arg 0 -> offset costante
arg 1 -> offset costante
arg 2 -> offset costante
```

## Trade-off

Vantaggi:

- piu' semplice per il verifier;
- consente di proseguire con il load eBPF;
- adatto al debugging MVP.

Svantaggi:

- formato meno compatto;
- decoder Go deve conoscere gli slot;
- in futuro si potra' tornare a un formato compatto piu' raffinato.

## Decoder userspace

Il package `demo_project/pkg/bufferdecoder` implementa la parte userspace del
protocollo:

- `protocol.go`: layout Go e schema degli eventi;
- `decoder.go`: primitive binarie e `DecodeContext`;
- `eventsreader.go`: decoding completo degli eventi e argomenti.

Il context e' di 128 byte. Questa e' una differenza voluta rispetto a Tracee,
che usa 136 byte per includere campi di policy non presenti nell'MVP.

## Stato verificato

E' stato osservato output JSON da eventi reali, ad esempio `cap_capable`:

```json
{"event_name":"cap_capable","args":[{"name":"cap","type":1,"value":2}]}
```

Questo conferma che la pipeline kernel -> buffer eventi -> decoder Go -> JSON
e' operativa.

## Canali kernel/userspace

La versione attuale mantiene due mappe eventi:

```c
struct events_ringbuf {
    __uint(type, BPF_MAP_TYPE_RINGBUF);
} events_ringbuf SEC(".maps");

struct events {
    __uint(type, BPF_MAP_TYPE_PERF_EVENT_ARRAY);
} events SEC(".maps");
```

Lato C sono disponibili due funzioni di submit:

- `events_ringbuf_submit`: usa `bpf_ringbuf_output`;
- `events_perf_submit`: usa `bpf_perf_event_output`.

Lato Go, `Project.Init()` apre entrambe:

```go
module.InitRingBuf("events_ringbuf", ...)
module.InitPerfBuf("events", ..., 1024)
```

Entrambi i canali finiscono in `handleRawEvent(raw []byte)`, che esegue:

1. decode del payload;
2. filtro evento;
3. filtro `comm`;
4. output.

Questa architettura resta utile anche dopo la migrazione degli hook correnti al
perf buffer: il percorso principale usa `events_perf_submit`, mentre
`events_ringbuf_submit` e la mappa `events_ringbuf` restano disponibili per
esperimenti, fallback o confronti futuri senza dover ricostruire il runtime.

## Collegamenti

- [Timeline](../timeline.md)
- [Debugging verifier](../debugging/ebpf-verifier.md)
- [Overview implementazione](overview.md)
- [Decoder Go](decoder.md)
