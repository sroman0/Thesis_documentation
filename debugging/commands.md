# Comandi utili

## Compilare eBPF

Da `demo_project`:

```bash
make bpf
```

Il target genera `dist/project.bpf.o`, che viene embedded nel binario Go.

Nota: il target attuale traccia direttamente `project.bpf.c`, ma non tutti gli
header `.h` inclusi. Dopo modifiche a `pkg/ebpf/c/common/*.h`, se `make bpf`
risponde con `Nothing to be done`, forzare una ricompilazione esplicita:

```bash
clang -O2 -g \
  -target bpf \
  -D__TARGET_ARCH_x86 \
  -D__x86_64__ \
  -I ./pkg/ebpf/c -I ./pkg/ebpf/c/common -I ./3rdparty/libbpfgo/libbpf/src \
  -c pkg/ebpf/c/project.bpf.c -o dist/project.bpf.o
```

## Eseguire userspace

```bash
make run
```

Formato table per debug manuale:

```bash
make run_table
```

Il target `run_table` e' utile anche per osservare `execve` ed `execveat`,
perche' usa eventi dedicati senza filtrare per `comm`.

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

Nota: per eventi presi all'ingresso della syscall, come `execve`, il campo
`comm` puo' ancora essere quello del processo chiamante. Per questo e'
preferibile testare `execve` filtrando per evento, non per `comm`.

## Log libbpf e relocation CO-RE

Per run normali, il default `--log-level info` nasconde i log verbose di
relocation CO-RE.

Per riabilitarli:

```bash
make run ARGS="--events execve,execveat --output table --log-level debug"
```

I messaggi `field_exists`, `byte_off` e `no matching targets found` non sono
necessariamente errori: spesso indicano che libbpf sta patchando il programma
in base ai campi disponibili nel BTF del kernel target.

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

Per generare eventi `execve` e `execveat`:

```bash
make run ARGS="--events execve,execveat --output table"
```

Poi, da un altro terminale:

```bash
ls
whoami
curl --version
```

La maggior parte dei comandi comuni genera `execve`; `execveat` e' meno comune
ma copre i casi fd-relative o con flag come `AT_EMPTY_PATH`.

Per forzare un evento `execveat`, compilare un piccolo programma C:

```c
#define _GNU_SOURCE
#include <fcntl.h>
#include <unistd.h>
#include <sys/syscall.h>

int main(void) {
    char *argv[] = {"whoami", NULL};
    char *envp[] = {NULL};

    return syscall(SYS_execveat, AT_FDCWD, "/usr/bin/whoami", argv, envp, 0);
}
```

Eseguirlo mentre il tool ascolta:

```bash
gcc /tmp/test_execveat.c -o /tmp/test_execveat
/tmp/test_execveat
```

Output atteso:

```text
event=execveat ... args=dirfd=-100,pathname=/usr/bin/whoami,flags=0
```

`-100` corrisponde a `AT_FDCWD`.

## Testare `security_bprm_check`

Terminale 1:

```bash
make run ARGS="--events security_bprm_check --output table"
```

Terminale 2:

```bash
whoami
ls
```

Output atteso:

```text
event=security_bprm_check ... args=pathname=/usr/bin/whoami,dev=...,inode=...,filename=/usr/bin/whoami,argc=1,envc=...
```

Questo evento non rappresenta solo la syscall richiesta da userspace: viene
emesso mentre il kernel valida il binario tramite `linux_binprm`. Per questo e'
utile accanto a `execve`, `execveat` e `sched_process_exec`.

Lettura dei campi principali:

- `pathname`: path del file eseguibile risolto dal kernel;
- `dev` e `inode`: identificano il file sul filesystem;
- `filename`: campo `linux_binprm->filename`;
- `argc`: numero di argomenti passati al nuovo programma;
- `envc`: numero di variabili d'ambiente.

Sequenza utile durante il debug:

```text
execve / execveat     -> richiesta userspace
security_bprm_check   -> validazione kernel del binario
sched_process_exec    -> exec riuscita
```

## Testare `security_task_fix_setuid`

Terminale 1:

```bash
make run ARGS="--events security_task_fix_setuid --output table"
```

Esca piu' pulita:

```bash
sudo python3 -c 'import os; os.setuid(65534); print(os.getuid(), os.geteuid())'
```

`65534` e' normalmente l'UID dell'utente `nobody`.

Output atteso:

```text
event=security_task_fix_setuid ... args=old_uid=0,new_uid=65534,old_euid=0,new_euid=65534,...
```

Esca alternativa:

```bash
sudo -u nobody id
```

Questa seconda esca genera piu' rumore perche' `sudo`, PAM e la sessione shell
possono alternare piu' volte UID effettivo e filesystem UID prima di eseguire
il comando finale.

Lettura del campo `flags`:

```text
1  LSM_SETID_ID   setuid/setgid
2  LSM_SETID_RE   setreuid/setregid
4  LSM_SETID_RES  setresuid/setresgid
8  LSM_SETID_FS   setfsuid/setfsgid
```

Se l'evento non viene emesso, controllare che il simbolo sia presente nel
kernel:

```bash
sudo grep security_task_fix_setuid /proc/kallsyms
```

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
