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
make run ARGS="--events sched_process_exec"
```

Disabilitare `cap_capable`, che e' molto rumoroso:

```bash
make run ARGS="--drop-events cap_capable"
```

Combinare include ed exclude:

```bash
make run ARGS="--events cap_capable,sched_process_exec --drop-events cap_capable"
```

Filtrare per nome comando (`comm`) dopo la decodifica:

```bash
make run ARGS="--events task_rename,sched_process_exec,sched_process_exit --comms ls,whoami --output table"
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

Il Makefile corrente non espone ancora un target `test`. Eseguire i package Go
direttamente, passando i flag CGO quando servono package che importano
`libbpfgo`.

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
mentre alcuni hook process/security usano ring buffer. La versione attuale del
runtime legge entrambi i canali (`events_ringbuf` e `events`), quindi questo
target puo' essere usato per verificare la pipeline networking.

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

## Verificare il reader duale

La versione attuale apre sia ring buffer sia perf buffer:

```text
events_ringbuf -> InitRingBuf
events         -> InitPerfBuf
```

Per verificare un evento su ring buffer:

```bash
make run ARGS="--events sched_process_exec --output table --comms ls"
```

Poi, da un altro terminale:

```bash
ls
```

Per verificare un evento su perf buffer/networking:

```bash
make filtered
```

Poi generare una connessione con `curl` come indicato nella sezione
`security_socket_connect`.

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
