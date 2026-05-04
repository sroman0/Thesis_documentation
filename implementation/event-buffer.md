# Protocollo eventi e buffer eBPF

## Scopo

Il buffer eventi serve a trasportare dati dal kernel allo userspace.

Ogni evento contiene:

```text
event_context_t
argnum
args buffer
```

Nel runtime userspace il record viene letto come `ringbuf.Record.RawSample`.
Il decoder Go interpreta il buffer con questo layout:

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

Questo conferma che la pipeline kernel -> ring buffer -> decoder Go -> JSON e'
operativa.

## Collegamenti

- [Timeline](../timeline.md)
- [Debugging verifier](../debugging/ebpf-verifier.md)
- [Overview implementazione](overview.md)
- [Decoder Go](decoder.md)
