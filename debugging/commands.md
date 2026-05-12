# Comandi utili

## Compilare eBPF

Da `demo_project`:

```bash
make bpf
```

Il target genera `dist/project.bpf.o`, che viene embedded nel binario Go.

## Eseguire userspace

```bash
make run
```

Formato table per debug manuale:

```bash
make run_table
```

## Eseguire solo alcuni eventi

Abilitare solo `sched_process_exec`:

```bash
sudo go run ./cmd/project --events sched_process_exec
```

Disabilitare `cap_capable`, che e' molto rumoroso:

```bash
sudo go run ./cmd/project --drop-events cap_capable
```

Combinare include ed exclude:

```bash
sudo go run ./cmd/project \
  --events cap_capable,sched_process_exec \
  --drop-events cap_capable
```

## Build binario

```bash
make build
sudo ./dist/project
```

`make build` compila prima l'oggetto eBPF e poi costruisce il binario Go con
`dist/project.bpf.o` embedded.

## Help CLI

Con `libbpfgo`, il comando diretto `go run` richiede i flag CGO corretti. Usare
il target dedicato:

```bash
make help
```

Oppure buildare il binario:

```bash
make build
sudo ./dist/project --help
```

## Test Go

```bash
make test
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

## Testare `security_settime64`

Terminale 1:

```bash
make run ARGS="--events security_settime64 --output table"
```

Terminale 2:

```bash
sudo date -s "$(date '+%Y-%m-%d %H:%M:%S')"
```

Nota: questo evento richiede privilegi per modificare l'orologio di sistema
(`CAP_SYS_TIME`). Su macchine condivise va usato con cautela.

## Testare `security_socket_connect`

Target rapido:

```bash
make filtered
```

Generare traffico locale:

```bash
python3 -m http.server 18080 --bind 127.0.0.1
curl http://127.0.0.1:18080
```

Nota: gli hook networking importati usano ancora in gran parte perf buffer,
mentre il runtime Go legge la ring buffer. Se non compare output, non significa
automaticamente che il kprobe non venga raggiunto: puo' mancare il reader del
canale corretto.

## Installare il binario come comando locale

```bash
make build
sudo cp ./dist/project /usr/local/bin/project
```

Se `sudo project --help` non trova il comando, verificare il `secure_path` di
sudo oppure usare il path assoluto:

```bash
sudo /usr/local/bin/project --help
```

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

Gli `args` sono il payload specifico dell'evento kernel. Alcuni sono gia'
leggibili, ad esempio `filename=/usr/bin/ls` o `exit_code=0`. Altri restano
numerici perche' rappresentano costanti o puntatori kernel/userspace, ad esempio
gli argomenti di `security_task_prctl`.

## Collegamenti

- [Timeline](../timeline.md)
- [Debugging verifier](ebpf-verifier.md)
