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

Formato table per debug manuale:

```bash
sudo go run ./cmd/project --bpf-object pkg/ebpf/c/project.bpf.o --output table
```

## Eseguire solo alcuni eventi

Abilitare solo `sched_process_exec`:

```bash
sudo go run ./cmd/project --bpf-object pkg/ebpf/c/project.bpf.o --events sched_process_exec
```

Disabilitare `cap_capable`, che e' molto rumoroso:

```bash
sudo go run ./cmd/project --bpf-object pkg/ebpf/c/project.bpf.o --drop-events cap_capable
```

Combinare include ed exclude:

```bash
sudo go run ./cmd/project --bpf-object pkg/ebpf/c/project.bpf.o \
  --events cap_capable,sched_process_exec \
  --drop-events cap_capable
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

## Test selezione probe

```bash
GOCACHE=/tmp/go-build go test ./pkg/ebpf/probes
```

## Test output layer

```bash
GOCACHE=/tmp/go-build go test ./pkg/output
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

Con `--output json`, il programma stampa una riga JSON per ogni evento
decodificato. Esempio:

```json
{"timestamp":4074609472044753,"event_name":"cap_capable","process":{"pid":1374462,"tid":1374462,"ppid":1348641,"uid":1000,"comm":"cpuUsage.sh","uts_name":"security-thesis"},"host":{"pid":1374462,"tid":1374462,"ppid":1348641},"kernel":{"syscall":56,"processor_id":0,"mnt_id":4026531840,"pid_id":4026531836},"args":[{"name":"cap","type":1,"value":21,"label":"CAP_SYS_ADMIN"}]}
```

`cap_capable` puo' produrre molte righe ravvicinate. Questo non indica
necessariamente un errore: il security hook viene invocato spesso dal kernel.

Con `--output table`, l'output e' piu' compatto:

```text
event=cap_capable pid=1374462 tid=1374462 uid=1000 comm=cpuUsage.sh args=cap=CAP_SYS_ADMIN(21)
```

## Collegamenti

- [Timeline](../timeline.md)
- [Debugging verifier](ebpf-verifier.md)
