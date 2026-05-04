# Comandi utili

## Compilare eBPF

Da `demo_project`:

```bash
clang -g -O2 -target bpf -D__TARGET_ARCH_x86 -I pkg/ebpf/c \
  -c pkg/ebpf/c/project.bpf.c -o pkg/ebpf/c/project.bpf.o
```

## Eseguire userspace

```bash
sudo go run ./cmd/project --bpf-object pkg/ebpf/c/project.bpf.o
```

## Build binario

```bash
go build -o /tmp/project ./cmd/project
sudo /tmp/project --bpf-object pkg/ebpf/c/project.bpf.o
```

## Test Go

```bash
GOCACHE=/tmp/go-build go test ./...
```

## Test decoder

```bash
GOCACHE=/tmp/go-build go test ./pkg/bufferdecoder
```

## Generare eventi manuali

Mentre il programma gira:

```bash
ls
whoami
sleep 1
echo test
```

Questi comandi dovrebbero generare almeno eventi `sched_process_exec`.

## Output atteso

Il programma stampa una riga JSON per ogni evento decodificato. Esempio:

```json
{"event_name":"cap_capable","args":[{"name":"cap","type":1,"value":2}]}
```

`cap_capable` puo' produrre molte righe ravvicinate. Questo non indica
necessariamente un errore: il security hook viene invocato spesso dal kernel.

## Collegamenti

- [Timeline](../timeline.md)
- [Debugging verifier](ebpf-verifier.md)
