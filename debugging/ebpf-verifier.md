# Debugging verifier eBPF

## Problema principale incontrato

Durante il load della collection, il verifier ha rifiutato diversi programmi con errori simili:

```text
invalid access to map value, value_size=4280 off=65664 size=...
```

## Interpretazione

Il valore `65664` deriva da:

```text
offset base dentro event_data_t + possibile valore massimo di args_buf.offset
```

Poiche' `args_buf.offset` e' un `u16`, il verifier lo considera potenzialmente fino a:

```text
65535
```

Anche se il codice mantiene `offset` entro `ARGS_BUF_SIZE`, il verifier deve poterlo dimostrare localmente.

## Mitigazioni applicate

### Slot fissi scalari

Invece di usare offset dinamico:

```c
buf->args[buf->offset]
```

gli scalari usano slot fissi da 16 byte.

### Slot fissi stringhe

Le stringhe usano slot fissi:

```text
1 byte tag + 4 byte length + 512 byte payload
```

### Bound esplicito in submit

Prima di `bpf_ringbuf_output`:

```c
if (args_off > ARGS_BUF_SIZE)
    return 0;

if (size > MAX_EVENT_SIZE)
    return 0;
```

## Lezione appresa

In eBPF non basta che il codice sia logicamente sicuro: deve essere dimostrabile dal verifier. Spesso questo richiede:

- offset costanti;
- size costanti;
- controlli vicini all'uso;
- meno aritmetica dinamica su puntatori a map value.

## Collegamenti

- [Timeline](../timeline.md)
- [Event buffer](../implementation/event-buffer.md)
- [Report tecnico](../report.md)
