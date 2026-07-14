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

## Listare gli eventi disponibili

La CLI puo' stampare tutti gli eventi supportati senza caricare eBPF e senza
richiedere `sudo`:

```bash
./dist/project --list-events
```

Questo comando legge il registry in `pkg/ebpf/probes/probes.go` e mostra i nomi
usabili con `--events` e `--drop-events`.

Nota: la lista contiene solo eventi pubblici con schema decoder. I probe interni
registrati per aggiornare stato kernel-side non vengono mostrati e non possono
essere selezionati direttamente.

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

## Testare policy, detector e alert

Il test minimo del layer detector usa i file demo:

```text
demo_project/rules/policies/demo-detectors.yaml
demo_project/rules/policies/full-demo.yaml
demo_project/rules/policies/collective-privilege-exec.yaml
demo_project/rules/policies/point-process-security.yaml
demo_project/rules/policies/sensitive-file-access.yaml
demo_project/rules/policies/kernel-module-activity.yaml
demo_project/rules/detectors/root_exec.yaml
demo_project/rules/detectors/sensitive_file_open.yaml
demo_project/rules/detectors/privileged_uid_change.yaml
demo_project/rules/detectors/privilege_exec_chain.yaml
demo_project/rules/detectors/kernel_module_activity.yaml
```

Le policy sono organizzate per scenario:

| Policy | Scenario |
| --- | --- |
| `demo-detectors.yaml` | Demo compatibile con il primo set di detector. |
| `full-demo.yaml` | Tutti i detector demo correnti. |
| `collective-privilege-exec.yaml` | Solo catena privilegi -> exec root. |
| `point-process-security.yaml` | Detector puntuali su processi e UID. |
| `sensitive-file-access.yaml` | Solo apertura file sensibili. |
| `kernel-module-activity.yaml` | Solo inizializzazione moduli kernel. |

Da `demo_project`, avviare il runtime con policy, detector e alert:

```bash
sudo ./dist/project \
  --policy rules/policies/demo-detectors.yaml \
  --detectors rules/detectors \
  --alerts-only \
  --alerts-output table \
  --output table \
  --log-level error
```

In un secondo terminale generare eventi semplici:

```bash
cat /etc/passwd
sudo whoami
sudo true
```

Output atteso per il detector `sensitive-file-open`:

```text
type=alert alert=Sensitive system file opened severity=medium detector=sensitive-file-open events=1 detector_name=Sensitive file open source_event=security_file_open source_pid=... source_uid=... source_comm=cat source_args=pathname=/etc/passwd,...
```

Questo output dimostra che il detector sta funzionando: l'evento
`security_file_open` viene decodificato, passa dal dispatcher, soddisfa la
condizione YAML su `args.pathname` e genera un alert.

Nota: il detector non usa piu' il prefisso generico `/etc/`, perche' produceva
troppo rumore su file benigni come `/etc/hosts` e `/etc/ld.so.cache`. Ora usa
`operator: in` su una lista di path critici.

I detector demo piu' rumorosi escludono inoltre gli helper legittimi `sudo` e
`unix_chkpwd` dove questo migliora la leggibilita' del segnale. In particolare
`root-exec`, `sensitive-file-open`, `privileged-uid-change`,
`privilege-exec-chain` e `privilege-sensitive-file-chain` sono pensati per demo
e threat hunting locale, non per modellare ogni dettaglio interno di PAM/sudo.

Nota sul rumore: il detector engine applica un dedup temporale di default pari
a 5 secondi. Se lo stesso processo genera piu' volte lo stesso alert, il tool
stampa il primo e sopprime i duplicati immediatamente successivi. Per verificarlo:

```bash
cat /etc/passwd
cat /etc/passwd
cat /etc/passwd
```

Nella finestra breve dovresti vedere un solo alert `sensitive-file-open`.
Aspettando piu' di 5 secondi e rilanciando `cat /etc/passwd`, l'alert puo'
essere emesso di nuovo.

Altri alert possibili con la stessa policy:

- `root-exec`: esecuzione di un processo come UID 0, esclusi helper legittimi
  noti come `sudo` e `unix_chkpwd`;
- `privileged-uid-change`: transizione di effective UID a 0 partendo da un real
  UID non root;
- `privilege-exec-chain`: transizione privilegiata seguita da exec root entro
  5 secondi;
- `privilege-sensitive-file-chain`: transizione privilegiata seguita da accesso
  root a file sensibili, escludendo helper legittimi di `sudo`;
- `memfd-exec-chain`: `memfd_create` riuscita seguita da exec da path memfd;
- `kernel-module-kprobe-chain`: inizializzazione modulo seguita da
  registrazione kprobe dinamica;
- `kernel-module-activity`: inizializzazione riuscita di un modulo kernel.

Per testare il detector collective `privilege-exec-chain`, avvia il tool con
lo stesso comando e poi lancia:

```bash
sudo whoami
```

Output atteso, se gli eventi arrivano nella finestra corretta:

```text
type=alert alert=Privilege change followed by process execution severity=high detector=privilege-exec-chain events=2 ... sequence=security_task_fix_setuid(...)->sched_process_exec(...)
```

Nota: questo detector collective usa ora `group_by: process_tree`. La prima
versione usava una chiave `global`, utile per rendere visibile la demo ma troppo
permissiva. `process_tree` riduce i falsi positivi usando chiavi locali derivate
da `host_pid`, `host_ppid`, `start_time` e `parent_start_time`.

Per isolare solo il detector collective durante il test:

```bash
sudo ./dist/project \
  --policy rules/policies/collective-privilege-exec.yaml \
  --detectors rules/detectors/privilege_exec_chain.yaml \
  --alerts-only \
  --alerts-output table \
  --log-level error
```

In questo caso `events=2` conferma che il detector ha correlato una sequenza,
non un singolo evento. Per gli alert collective il formato table usa `sequence`
come rappresentazione principale e non ripete i campi `source_*`, che restano
usati per gli alert generati da un singolo evento.

Per caricare tutti i detector collective correnti:

```bash
sudo ./dist/project \
  --policy rules/policies/collective-local-chains.yaml \
  --detectors rules/detectors \
  --alerts-only \
  --alerts-output table \
  --log-level error
```

Per testare la catena privilege -> sensitive file:

```bash
sudo cat /etc/passwd
```

Output atteso:

```text
type=alert alert=Privileged process accessed sensitive file ... events=2 mitre=TA0004|TA0007|T1548|T1083 ... sequence=security_task_fix_setuid(...)->security_file_open(...)
```

Questo detector e' volutamente piu' stretto del detector point
`sensitive-file-open`: richiede `args.old_uid != 0`, `args.new_euid == 0`,
apertura file come `uid=0` ed esclude `comm=sudo` e `comm=unix_chkpwd`. Inoltre
non include `/etc/sudoers` nella catena collective, per evitare alert sul normale
controllo policy interno di `sudo`.

Gli alert table includono anche una sintesi MITRE nel campo `mitre=...`. Per
avere il mapping completo con framework, tattiche, tecniche, data sources e data
components, usare l'output alert JSON:

```bash
sudo ./dist/project \
  --policy rules/policies/collective-local-chains.yaml \
  --detectors rules/detectors/privilege_sensitive_file_chain.yaml \
  --alerts-only \
  --alerts-output json \
  --log-level error
```

Nel JSON l'alert contiene un blocco `threat`, utile per SOC/SIEM e report:

```json
"threat": {
  "framework": "MITRE ATT&CK Enterprise",
  "tactics": [{"id": "TA0004", "name": "Privilege Escalation"}],
  "techniques": [{"id": "T1548", "name": "Abuse Elevation Control Mechanism"}]
}
```

`--alerts-only` sopprime la stampa degli eventi raw, ma non li elimina dalla
pipeline: gli eventi continuano a essere decodificati, filtrati e inviati ai
detector. Questo rende i test e le demo piu' leggibili.

Se invece vuoi vedere sia eventi raw sia alert, usa `--alerts`:

```bash
sudo ./dist/project \
  --policy rules/policies/demo-detectors.yaml \
  --detectors rules/detectors \
  --alerts \
  --alerts-output table \
  --output table \
  --log-level error
```

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

Dopo la migrazione a zap, anche i messaggi libbpf passano dal logger runtime.
Sono riconoscibili dai campi:

```text
source=libbpf
libbpf_level=<livello numerico libbpf>
```

La policy resta:

- `debug`: mostra anche i dettagli CO-RE/libbpf;
- `info`: nasconde la diagnostica verbose libbpf;
- `warn`/`error`: mostra solo warning libbpf.

## Logging runtime e zap

Il runtime usa `go.uber.org/zap` per i log applicativi e per i messaggi libbpf
ammessi dal filtro.
La distinzione da mantenere e':

```text
log runtime -> stderr
eventi      -> output printer
alert       -> alert printer
```

Quindi `--log-level` dovra' controllare il rumore applicativo senza cambiare la
semantica degli eventi stampati.

Comandi utili per confrontare i profili:

```bash
sudo ./dist/project --events execve,security_file_open --output table --log-level error
sudo ./dist/project --events execve,security_file_open --output table --log-level info
sudo ./dist/project --events execve,security_file_open --output table --log-level debug
```

`debug` deve essere usato solo per diagnostica mirata, perche' puo' stampare
attach probe, drop reason e dettagli di decode. Per benchmark e demo operative
usare `error` o `info`.

Per ambienti containerizzati o pipeline centralizzate, usare log runtime JSON:

```bash
sudo ./dist/project \
  --events execve,security_file_open \
  --output json \
  --log-level info \
  --log-format json
```

Nota: `--log-format` non cambia il formato degli eventi o degli alert. Per
quelli restano validi `--output` e `--alerts-output`.

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

## Debug eventi con `args=-`

Sintomo:

```text
event=security_file_open pid=... comm=git args=-
event=sched_process_exec pid=... comm=git args=-
```

Interpretazione:

- se il nome evento e' corretto, il trasporto perf/ring buffer funziona;
- se `args=-`, il decoder ha prodotto un evento con `argnum=0`;
- se l'hook dovrebbe sicuramente salvare argomenti, controllare subito il
  contratto binario tra `event_context_t` e `bufferdecoder.EventContext`.

Controlli utili:

```bash
rg -n "eventContextSize|MatchedPolicies|matched_policies" \
  pkg/bufferdecoder pkg/ebpf/c/types.h
```

Il formato corrente e':

```text
[event_context_t:136][argnum:u8][args...]
```

Quindi `eventContextSize` in `pkg/bufferdecoder/protocol.go` deve essere `136`.
Se resta a `128`, il decoder legge `argnum` dentro `matched_policies` e stampa
eventi senza argomenti.

Test consigliato dopo la correzione:

```bash
PKG_CONFIG_PATH=./dist/libbpf/obj \
CGO_CFLAGS="-I/home/simone/project/demo_project/dist/libbpf/include -I/home/simone/project/demo_project/3rdparty/libbpfgo" \
CGO_LDFLAGS="-L/home/simone/project/demo_project/dist/libbpf/obj -lbpf" \
GOCACHE=/tmp/go-build \
go test ./pkg/bufferdecoder ./pkg/output ./pkg/detectors ./pkg/detectors/yaml ./pkg/policy ./pkg/ebpf ./pkg/cmd
```

Poi ricompilare il binario:

```bash
PKG_CONFIG_PATH=./dist/libbpf/obj \
CGO_CFLAGS="-I/home/simone/project/demo_project/dist/libbpf/include -I/home/simone/project/demo_project/3rdparty/libbpfgo" \
CGO_LDFLAGS="-L/home/simone/project/demo_project/dist/libbpf/obj -lbpf" \
GOCACHE=/tmp/go-build \
go build -o ./dist/project ./cmd/project
```

E verificare runtime:

```bash
sudo ./dist/project \
  --events sched_process_exec,security_file_open \
  --output table \
  --log-level error
```

L'output deve contenere argomenti reali, per esempio `filename=...` o
`pathname=...,flags=...`.

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

## Testare `proc_create`

Terminale 1:

```bash
sudo ./dist/project --events proc_create --output table --log-level error
```

Terminale 2:

```bash
sudo modprobe nf_conntrack 2>/dev/null || true
```

Output atteso, se il modulo crea entry procfs nel percorso del kernel target:

```text
event=proc_create ... args=name=...,proc_ops_addr=0x...
```

Nota: se non compare output, non significa necessariamente che l'hook non
funzioni. Il modulo potrebbe essere gia' caricato, non creare entry procfs in
quel percorso, oppure attraversare una variante interna diversa. In quel caso
conviene provare con un modulo di test che chiama esplicitamente `proc_create`.

## Testare `register_kprobe`

Terminale 1:

```bash
sudo ./dist/project --events register_kprobe --output table --log-level error
```

Terminale 2:

```bash
sudo ./dist/project --events cap_capable --output table --log-level error
```

Il secondo comando carica un altro programma eBPF che registra kprobe, quindi
il primo tracer dovrebbe vedere una o piu' registrazioni:

```text
event=register_kprobe ... args=symbol_name=cap_capable,pre_handler=0x...,post_handler=0x0,returnValue=success
```

Chiudere il secondo tracer con `Ctrl-C` dopo aver osservato l'evento.

## Testare `kallsyms_lookup_name`

Terminale 1:

```bash
sudo ./dist/project --events kallsyms_lookup_name --output table --log-level error
```

Questo hook viene innescato solo quando codice kernel o un modulo chiama
realmente `kallsyms_lookup_name`. Da userspace non esiste un comando generico e
sempre affidabile per produrre l'evento.

Per un test forte serve un modulo di laboratorio che risolva esplicitamente un
simbolo, ad esempio `kallsyms_lookup_name("printk")`. Quando il percorso viene
raggiunto, l'output atteso e':

```text
event=kallsyms_lookup_name ... args=symbol_name=printk,address=0x...
```

Se il tracer parte senza errori ma non stampa nulla, significa semplicemente
che nessun lookup e' avvenuto durante la finestra di test.

Lettura del campo `flags`:

```text
1  LSM_SETID_ID   setuid/setgid
2  LSM_SETID_RE   setreuid/setregid
4  LSM_SETID_RES  setresuid/setresgid
8  LSM_SETID_FS   setfsuid/setfsgid
```

Descrizione rapida delle flag:

- `LSM_SETID_ID`: percorso `setuid`/`setgid`. Imposta l'ID utente o gruppo del
  processo. Se il chiamante e' privilegiato, puo' aggiornare real, effective e
  saved ID; se non lo e', puo' solo passare a un ID gia' consentito, per
  esempio real o saved ID.
- `LSM_SETID_RE`: percorso `setreuid`/`setregid`. Permette di gestire real ID
  ed effective ID separatamente, utile per cambi temporanei di privilegio.
- `LSM_SETID_RES`: percorso `setresuid`/`setresgid`. Permette di impostare in
  modo esplicito real, effective e saved ID; e' il caso piu' chiaro quando un
  programma vuole controllare l'intera transizione di privilegi.
- `LSM_SETID_FS`: percorso `setfsuid`/`setfsgid`. Modifica solo l'ID usato per
  i controlli di accesso al filesystem, senza cambiare direttamente effective
  UID/GID.

Se l'evento non viene emesso, controllare che il simbolo sia presente nel
kernel:

```bash
sudo grep security_task_fix_setuid /proc/kallsyms
```

## Testare `security_task_kill`

Terminale 1:

```bash
make run ARGS="--events security_task_kill --output table"
```

Terminale 2:

```bash
sleep 1000 &
target=$!
kill -TERM "$target"
```

Output atteso:

```text
event=security_task_kill ... args=target_host_pid=...,target_host_tid=...,target_uid=...,target_comm=sleep,signal=SIGTERM(15)
```

Nel formato `table`, il campo `signal` viene arricchito con il nome simbolico
del segnale. Per esempio `15` diventa `SIGTERM(15)`, mentre `0` viene mostrato
come `permission_check(0)` per indicare i controlli di permesso senza invio di
un segnale reale.

Per osservare altri segnali:

```bash
sleep 1000 &
target=$!
kill -STOP "$target"
kill -CONT "$target"
kill -TERM "$target"
```

Nota: questo evento viene emesso durante la validazione security dell'invio del
segnale. Non contiene il return value finale della syscall `kill`/`tgkill`.
Se serve sapere con certezza se la syscall e' riuscita, bisogna usare un evento
syscall con modello `sys_enter` + `sys_exit`.

## Testare `ptrace`

Terminale 1:

```bash
make run ARGS="--events ptrace --output table --comms strace,gdb"
```

Terminale 2:

```bash
sleep 1000 &
target=$!
strace -p "$target"
```

Output atteso:

```text
event=ptrace ... comm=strace args=request=PTRACE_SEIZE(16902),pid=...,addr=...,data=...,returnValue=...
```

Per testare un caso esplicitamente negato, si puo' provare ad agganciare un
processo non autorizzato o eseguire il comando senza privilegi sufficienti. In
questo caso `returnValue` sara' negativo, tipicamente `-1` con `EPERM` lato
userspace.

## Testare `security_file_open`

Terminale 1:

```bash
make run ARGS="--events security_file_open --output table --comms cat"
```

Terminale 2:

```bash
cat /etc/hostname
```

Output atteso:

```text
event=security_file_open ... comm=cat args=pathname=/etc/hostname,flags=...,dev=...,inode=...,ctime=...,syscall_pathname=/etc/hostname
```

Nota: `security_file_open` e' un hook molto rumoroso. Durante i test manuali e'
quasi sempre necessario usare `--events security_file_open` insieme a
`--comms <processo>`.

## Testare `security_file_ioctl`

Terminale 1:

```bash
sudo ./dist/project --events security_file_ioctl --output table --comms stty,python3
```

Terminale 2:

```bash
stty -a >/dev/null
```

Output atteso:

```text
event=security_file_ioctl ... args=pathname=...,cmd=0x...,arg=0x...,dev=...,inode=...,ctime=...
```

`ioctl` e' una famiglia molto ampia e driver-specific. Per questo il comando
viene mostrato in esadecimale quando il tool non conosce una label simbolica.

## Testare `security_file_permission`

Terminale 1:

```bash
sudo ./dist/project --events security_file_permission --output table --comms cat
```

Terminale 2:

```bash
cat /etc/hostname >/dev/null
```

Output atteso:

```text
event=security_file_permission ... comm=cat args=pathname=/etc/hostname,mask=MAY_READ|MAY_OPEN,dev=...,inode=...,ctime=...
```

Questo hook puo' generare molti eventi perche' osserva controlli successivi su
file gia' aperti. Per test manuali conviene abilitarlo da solo e filtrare con
`--comms`.

## Testare `do_sigaction`

Terminale 1:

```bash
sudo ./dist/project --events do_sigaction --output table --comms python3
```

Terminale 2:

```bash
python3 -c 'import signal, time; signal.signal(signal.SIGUSR1, lambda s, f: None); time.sleep(1)'
```

Output atteso:

```text
event=do_sigaction ... comm=python3 args=signal=SIGUSR1(10),new_action=true,new_sa_flags=...,new_sa_mask=...,new_handle_method=SIG_HND(2),new_sa_handler=0x...,old_action_requested=...,old_sa_flags=...,old_sa_mask=...,old_handle_method=SIG_DFL(0),old_sa_handler=0x0
```

Questo hook osserva il cambio di signal handler. Se il programma imposta il
segnale a default o ignored, `new_handle_method` diventa rispettivamente
`SIG_DFL(0)` o `SIG_IGN(1)`.

## Testare `call_usermodehelper`

Terminale 1:

```bash
sudo ./dist/project --events call_usermodehelper --output table --log-level error
```

Terminale 2:

```bash
sudo modprobe dummy || true
sudo modprobe -r dummy || true
```

Output possibile:

```text
event=call_usermodehelper ... args=path=/sbin/modprobe,argv=["modprobe",...],envp=[...],wait=UMH_WAIT_PROC(1)
```

Nota: questo evento dipende dal fatto che il kernel richiami davvero un helper
userspace. Se il modulo e' gia' caricato, non esiste o il kernel risolve la
richiesta senza helper, il test puo' non produrre output.

## Testare `cgroup_attach_task`

Terminale 1:

```bash
sudo ./dist/project --events cgroup_attach_task --output table --log-level error
```

Terminale 2:

```bash
sudo mkdir -p /sys/fs/cgroup/project-demo
sleep 30 &
pid=$!
echo "$pid" | sudo tee /sys/fs/cgroup/project-demo/cgroup.procs >/dev/null
wait "$pid" 2>/dev/null || true
sudo rmdir /sys/fs/cgroup/project-demo
```

Output atteso:

```text
event=cgroup_attach_task ... args=cgroup_id=...,hierarchy_id=...,cgroup_path=...,target_comm=sleep,target_host_pid=...,target_host_tid=...,threadgroup=...
```

Nota: il comando sopra assume cgroup v2 montato in `/sys/fs/cgroup`. Se la VM
usa layout diverso o permessi differenti, il test va adattato al mount cgroup
effettivo.

## Testare `cgroup_mkdir` e `cgroup_rmdir`

Terminale 1:

```bash
sudo ./dist/project --events cgroup_mkdir,cgroup_rmdir --output table --log-level error
```

Terminale 2:

```bash
sudo mkdir -p /sys/fs/cgroup/project-demo
sudo rmdir /sys/fs/cgroup/project-demo
```

Output atteso:

```text
event=cgroup_mkdir ... args=cgroup_id=...,path=/project-demo,hierarchy_id=...
event=cgroup_rmdir ... args=cgroup_id=...,path=/project-demo,hierarchy_id=...
```

Nota: il test assume cgroup v2 montato in `/sys/fs/cgroup`. Se la directory non
e' scrivibile o la VM usa un layout diverso, controlla il mount effettivo con:

```bash
mount | grep cgroup
```

## Testare `module_load` e `module_free`

Terminale 1:

```bash
sudo ./dist/project --events module_load,module_free --output table --log-level error
```

Terminale 2:

```bash
sudo modprobe -r dummy 2>/dev/null || true
sudo modprobe dummy
sudo modprobe -r dummy
```

Output atteso:

```text
event=module_load ... args=name=dummy,version=...,srcversion=...
event=module_free ... args=name=dummy,version=...,srcversion=...
```

Se non viene stampato nulla, il modulo potrebbe non esistere sulla VM oppure
potrebbe essere gia' gestito in modo diverso dal kernel. In quel caso prova con
un modulo innocuo disponibile sul sistema e controlla prima:

```bash
lsmod | head
modinfo dummy
```

## Testare `do_init_module`

Terminale 1:

```bash
sudo ./dist/project --events do_init_module --output table --log-level error
```

Terminale 2:

```bash
sudo modprobe -r dummy 2>/dev/null || true
sudo modprobe dummy || true
```

Output atteso:

```text
event=do_init_module ... args=name=dummy,version=...,srcversion=...,returnValue=success
```

Questo evento viene emesso quando il kernel attraversa davvero
`do_init_module`. Se il modulo e' gia' caricato, `modprobe dummy` puo' non
innescare una nuova inizializzazione.

## Testare `process_execute_failed`

Terminale 1:

```bash
sudo ./dist/project --events process_execute_failed --output table --log-level error --comms bash
```

Terminale 2:

```bash
/tmp/project-binary-does-not-exist 2>/dev/null || true
```

Output atteso:

```text
event=process_execute_failed ... comm=bash args=operation=execve(1),dirfd=-100,pathname=/tmp/project-binary-does-not-exist,flags=0,returnValue=ENOENT(-2): no such file or directory
```

Per testare esplicitamente il ramo `execveat`:

```bash
cat >/tmp/test_execveat_fail.c <<'EOF'
#define _GNU_SOURCE
#include <fcntl.h>
#include <unistd.h>
#include <sys/syscall.h>

int main(void) {
    char *argv[] = {"missing", NULL};
    char *envp[] = {NULL};
    return syscall(SYS_execveat, AT_FDCWD, "/tmp/does-not-exist", argv, envp, 0);
}
EOF
gcc /tmp/test_execveat_fail.c -o /tmp/test_execveat_fail
/tmp/test_execveat_fail || true
```

In questo caso l'output deve indicare `operation=execveat(2)`.

## Testare `open`

Terminale 1:

```bash
make run ARGS="--events open --output table --comms cat"
```

Terminale 2:

```bash
cat /etc/hostname
cat /etc/shadow 2>/dev/null || true
```

Output atteso:

```text
event=open ... comm=cat args=operation=openat(2),dirfd=-100,pathname=/etc/hostname,flags=...,mode=0000,returnValue=3
event=open ... comm=cat args=operation=openat(2),dirfd=-100,pathname=/etc/shadow,flags=...,mode=0000,returnValue=-13
```

Il valore positivo di `returnValue` e' il file descriptor restituito. Un valore
negativo rappresenta l'errore kernel, per esempio `-13` per `EACCES`.
Per osservare un errore di permesso, lascia il tool in esecuzione con `sudo`,
ma lancia il comando `cat` da un terminale non privilegiato. Se anche il comando
di test viene eseguito come root, `/etc/shadow` verra' aperto correttamente e
`returnValue` sara' un file descriptor positivo.

## Testare `chmod`

Terminale 1:

```bash
make run ARGS="--events chmod --output table --comms chmod"
```

Terminale 2:

```bash
touch /tmp/project-chmod-demo
chmod 600 /tmp/project-chmod-demo
chmod 755 /tmp/project-chmod-demo
```

Output atteso:

```text
event=chmod ... comm=chmod args=operation=fchmodat(3),dirfd=-100,pathname=/tmp/project-chmod-demo,mode=0755,returnValue=0
```

Per testare `fchmod` direttamente:

Terminale 1:

```bash
make run ARGS="--events chmod --output table --comms python3"
```

Terminale 2:

```bash
python3 -c 'import os; fd=os.open("/tmp/project-chmod-demo", os.O_RDONLY); os.fchmod(fd, 0o640); os.close(fd)'
```

In questo caso l'evento contiene `fd` invece di `pathname`.

## Testare `chown`

Terminale 1:

```bash
make run ARGS="--events chown --output table --comms chown"
```

Terminale 2:

```bash
touch /tmp/project-chown-demo
sudo chown root:root /tmp/project-chown-demo
sudo chown "$USER":"$USER" /tmp/project-chown-demo
```

Output atteso:

```text
event=chown ... comm=chown args=operation=fchownat(3),dirfd=-100,pathname=/tmp/project-chown-demo,owner=0,group=0,flags=...,returnValue=0
```

Per osservare un tentativo negato, lascia il tool in esecuzione con `sudo` ma
lancia il comando `chown` da un terminale non privilegiato:

```bash
chown root:root /tmp/project-chown-demo
```

In questo caso `returnValue` sara' negativo, tipicamente `-1` (`EPERM`).

Per testare `fchown` direttamente:

Terminale 1:

```bash
make run ARGS="--events chown --output table --comms python3"
```

Terminale 2:

```bash
python3 -c 'import os; fd=os.open("/tmp/project-chown-demo", os.O_RDONLY); os.fchown(fd, os.getuid(), os.getgid()); os.close(fd)'
```

In questo caso l'evento contiene `fd` invece di `pathname`.

## Testare `memfd_create`

Terminale 1:

```bash
make run ARGS="--events memfd_create --output table --comms python3"
```

Terminale 2:

```bash
python3 -c 'import ctypes, os; libc=ctypes.CDLL(None, use_errno=True); libc.memfd_create.argtypes=[ctypes.c_char_p, ctypes.c_uint]; libc.memfd_create.restype=ctypes.c_int; fd=libc.memfd_create(b"project-demo", 3); print(fd); os.close(fd)'
```

Output atteso:

```text
event=memfd_create ... comm=python3 args=name=project-demo,flags=MFD_CLOEXEC|MFD_ALLOW_SEALING,returnValue=fd:3
```

Il file descriptor preciso puo' cambiare. Un valore positivo indica che il
file anonimo e' stato creato; un valore negativo viene mostrato come errore
simbolico, ad esempio `EINVAL(-22): invalid argument`.

## Testare `mmap` e `mprotect`

Terminale 1:

```bash
make run ARGS="--events mmap,mprotect --output table --comms python3"
```

Terminale 2:

```bash
python3 -c 'import ctypes,mmap; region=mmap.mmap(-1,mmap.PAGESIZE,prot=mmap.PROT_READ|mmap.PROT_WRITE); addr=ctypes.addressof(ctypes.c_char.from_buffer(region)); libc=ctypes.CDLL(None,use_errno=True); print(libc.mprotect(ctypes.c_void_p(addr),ctypes.c_size_t(mmap.PAGESIZE),mmap.PROT_READ|mmap.PROT_EXEC)); region.close()'
```

Output rappresentativo:

```text
event=mmap ... comm=python3 args=addr=...,length=4096,prot=PROT_READ|PROT_WRITE,flags=MAP_PRIVATE|MAP_ANONYMOUS,fd=-1,offset=0,returnValue=address:0x...
event=mprotect ... comm=python3 args=addr=...,length=4096,prot=PROT_READ|PROT_EXEC,returnValue=success
```

Python e il loader dinamico generano altri mapping durante l'avvio. Il filtro
`--comms python3` limita i processi osservati, ma e' normale vedere piu' eventi
`mmap` rispetto alla singola regione creata dallo script.

## Testare `pkey_mprotect`

Terminale 1:

```bash
make run ARGS="--events pkey_mprotect --output table --comms python3"
```

Terminale 2:

```bash
python3 -c 'import ctypes,mmap; libc=ctypes.CDLL(None,use_errno=True); libc.syscall.restype=ctypes.c_long; region=mmap.mmap(-1,mmap.PAGESIZE,prot=mmap.PROT_READ|mmap.PROT_WRITE); addr=ctypes.addressof(ctypes.c_char.from_buffer(region)); result=libc.syscall(329,ctypes.c_void_p(addr),ctypes.c_size_t(mmap.PAGESIZE),ctypes.c_int(mmap.PROT_READ),ctypes.c_int(-1)); print("result",result,"errno",ctypes.get_errno()); region.close()'
```

Il numero `329` identifica `pkey_mprotect` su x86_64, cioe' l'architettura della
VM target. Il valore `pkey=-1` richiede il comportamento equivalente a
`mprotect` ma attraversa comunque la syscall `pkey_mprotect`, rendendo il test
affidabile anche quando `pkey_alloc` non riesce per assenza del supporto MPK
hardware. Un tentativo fallito resta osservabile e `returnValue` mostra
l'errno, ad esempio `EINVAL(-22)`.

## Testare `process_vm_writev`

Terminale 1:

```bash
sudo ./dist/project --events process_vm_writev --output table --comms python3
```

Terminale 2:

```bash
sudo python3 -c $'import ctypes, os\nclass IOVec(ctypes.Structure):\n    _fields_ = [("iov_base", ctypes.c_void_p), ("iov_len", ctypes.c_size_t)]\nlibc = ctypes.CDLL(None, use_errno=True)\nsource = ctypes.create_string_buffer(b"project-demo")\ntarget = ctypes.create_string_buffer(len(source))\nlocal = IOVec(ctypes.cast(source, ctypes.c_void_p), len(source))\nremote = IOVec(ctypes.cast(target, ctypes.c_void_p), len(target))\nlibc.process_vm_writev.argtypes = [ctypes.c_int, ctypes.POINTER(IOVec), ctypes.c_ulong, ctypes.POINTER(IOVec), ctypes.c_ulong, ctypes.c_ulong]\nlibc.process_vm_writev.restype = ctypes.c_ssize_t\nresult = libc.process_vm_writev(os.getpid(), ctypes.byref(local), 1, ctypes.byref(remote), 1, 0)\nprint("result", result, "target", target.value, "errno", ctypes.get_errno())'
```

Output rappresentativo:

```text
event=process_vm_writev ... comm=python3 args=pid=...,local_iov=0x...,liovcnt=1,remote_iov=0x...,riovcnt=1,flags=0,returnValue=bytes:13
```

## Testare `setns`

Terminale 1:

```bash
sudo ./dist/project --events setns --output table --comms python3
```

Terminale 2:

```bash
sudo python3 -c 'import ctypes, os; libc=ctypes.CDLL(None, use_errno=True); libc.setns.argtypes=[ctypes.c_int, ctypes.c_int]; libc.setns.restype=ctypes.c_int; fd=os.open("/proc/self/ns/mnt", os.O_RDONLY); rc=libc.setns(fd, 0x00020000); err=ctypes.get_errno(); print(f"setns rc={rc} errno={err}"); os.close(fd)'
```

Il comando prova a entrare nel mount namespace corrente. Non crea un nuovo
namespace e, se autorizzato, non cambia stabilmente l'ambiente della shell.

Output osservato sul kernel target:

```text
event=setns ... comm=python3 args=fd=3,nstype=CLONE_NEWNS,returnValue=success
```

Se la chiamata non e' autorizzata, l'evento viene comunque emesso con un valore
come `EPERM(-1): operation not permitted`.

## Testare `unshare`

Terminale 1:

```bash
sudo ./dist/project --events unshare --output table --comms python3
```

Terminale 2:

```bash
sudo python3 -c 'import ctypes; libc=ctypes.CDLL(None, use_errno=True); libc.unshare.argtypes=[ctypes.c_int]; libc.unshare.restype=ctypes.c_int; rc=libc.unshare(0x00000200); print(f"unshare rc={rc} errno={ctypes.get_errno()}")'
```

`0x00000200` corrisponde a `CLONE_FS`. La modifica resta confinata al breve
processo Python e non crea namespace di rete o PID.

Output osservato:

```text
event=unshare ... comm=python3 args=flags=CLONE_FS,returnValue=success
```

Per verificare il mapping di namespace multipli si puo' usare una combinazione
di flag, tenendo presente che la creazione di namespace richiede capability e
configurazioni kernel adeguate.

## Testare `process_vm_readv`

Terminale 1:

```bash
sudo ./dist/project --events process_vm_readv --output table --comms python3
```

Terminale 2:

```bash
sudo python3 -c $'import ctypes, os\nclass IOVec(ctypes.Structure):\n    _fields_ = [("iov_base", ctypes.c_void_p), ("iov_len", ctypes.c_size_t)]\nlibc = ctypes.CDLL(None, use_errno=True)\nsource = ctypes.create_string_buffer(b"project-read")\ntarget = ctypes.create_string_buffer(len(source))\nlocal = IOVec(ctypes.cast(target, ctypes.c_void_p), len(target))\nremote = IOVec(ctypes.cast(source, ctypes.c_void_p), len(source))\nlibc.process_vm_readv.argtypes = [ctypes.c_int, ctypes.POINTER(IOVec), ctypes.c_ulong, ctypes.POINTER(IOVec), ctypes.c_ulong, ctypes.c_ulong]\nlibc.process_vm_readv.restype = ctypes.c_ssize_t\nresult = libc.process_vm_readv(os.getpid(), ctypes.byref(local), 1, ctypes.byref(remote), 1, 0)\nprint("result", result, "target", target.value, "errno", ctypes.get_errno())'
```

Output osservato:

```text
event=process_vm_readv ... args=pid=...,local_iov=0x...,liovcnt=1,remote_iov=0x...,riovcnt=1,flags=0,returnValue=bytes:13
```

## Testare `commit_creds`

Terminale 1:

```bash
sudo ./dist/project --events commit_creds --output table --comms sudo
```

Terminale 2:

```bash
sudo -u nobody true
```

L'output contiene piu' transizioni perche' `sudo` modifica separatamente real,
effective, saved e filesystem ID, gruppi e capability. Un esempio rilevante e':

```text
event=commit_creds ... comm=sudo args=old_cred={uid:1000,...,euid:1000,...,cap_effective:0x0},new_cred={uid:1000,...,euid:0,...,cap_effective:0x1ffffffffff}
```

Il test scrive nel buffer dello stesso processo ed e' quindi innocuo. Serve a
verificare l'hook e il decoder. La condizione security piu' interessante sara'
una futura policy in cui il PID target e' diverso dal PID sorgente.

## Testare `security_bprm_creds_for_exec`

Terminale 1:

```bash
sudo ./dist/project --events security_bprm_creds_for_exec --output table --comms whoami,ls,bash
```

Terminale 2:

```bash
whoami
ls >/dev/null
```

Output atteso:

```text
event=security_bprm_creds_for_exec ... args=pathname=/usr/bin/whoami,dev=...,inode=...,filename=/usr/bin/whoami,argc=1,envc=...,ctime=...
```

Questo evento viene emesso durante la preparazione delle credenziali per il
nuovo programma. E' vicino al percorso di exec, ma non sostituisce
`sched_process_exec`: quest'ultimo conferma il cambio immagine riuscito.

## Testare `security_bpf`

Terminale 1:

```bash
sudo ./dist/project --events security_bpf --output table --comms bpftool,project
```

Terminale 2:

```bash
sudo bpftool prog show >/dev/null
sudo bpftool map show >/dev/null
```

Output atteso:

```text
event=security_bpf ... comm=bpftool args=cmd=BPF_PROG_GET_NEXT_ID(...),size=...
event=security_bpf ... comm=bpftool args=cmd=BPF_MAP_GET_NEXT_ID(...),size=...
```

Se `bpftool` non e' installato, l'evento puo' comunque comparire quando il
nostro tool o altri programmi caricano oggetti eBPF. In quel caso usare un
filtro `--comms project` mentre si avvia una seconda istanza o un altro tool
eBPF di test.

## Testare `security_bpf_map` e `security_bpf_prog`

Terminale 1:

```bash
sudo ./dist/project --events security_bpf_map,security_bpf_prog --output table --comms bpftool,project
```

Terminale 2:

```bash
sudo bpftool map show >/dev/null
sudo bpftool prog show >/dev/null
```

Output atteso:

```text
event=security_bpf_map ... args=map_id=...,map_name=...
event=security_bpf_prog ... args=prog_id=...,prog_name=...,prog_type=BPF_PROG_TYPE_TRACEPOINT(...)
```

Questi eventi osservano operazioni su oggetti eBPF gia' esistenti. Sono piu'
semantici di `security_bpf`, che invece descrive il comando generale passato a
`bpf(2)`.

## Testare `security_kernel_read_file`

Terminale 1:

```bash
sudo ./dist/project --events security_kernel_read_file --output table --log-level error
```

Terminale 2:

```bash
sudo modprobe dummy || true
sudo modprobe -r dummy || true
```

Output atteso, se il modulo e' disponibile sul sistema:

```text
event=security_kernel_read_file ... args=pathname=...,dev=...,inode=...,type=READING_MODULE(2),ctime=...
```

Questo hook si attiva quando il kernel legge file per proprie funzionalita',
per esempio moduli, firmware, policy o immagini kexec. Se `dummy` non e'
presente, provare con un modulo disponibile sulla VM, ad esempio:

```bash
find /lib/modules/$(uname -r) -type f -name '*.ko*' | head
```

e poi usare `modprobe` sul nome del modulo senza estensione.

## Testare `security_kernel_post_read_file`

Terminale 1:

```bash
sudo ./dist/project --events security_kernel_post_read_file --output table --log-level error
```

Terminale 2:

```bash
sudo modprobe dummy || true
sudo modprobe -r dummy || true
```

Output atteso, se il percorso kernel viene raggiunto:

```text
event=security_kernel_post_read_file ... args=pathname=...,size=...,type=READING_MODULE(2),dev=...,inode=...,ctime=...
```

Se non compare nulla, non e' necessariamente un errore del tool: il modulo
potrebbe non esistere, essere gia' cacheato, oppure il comando scelto potrebbe
non attraversare il percorso `kernel_read_file` sul kernel corrente.

## Testare `fork`, `vfork` e `clone`

Per `clone`:

Terminale 1:

```bash
sudo ./dist/project --events clone --output table --comms python3
```

Terminale 2:

```bash
python3 -c 'import os; pid=os.fork(); os._exit(0) if pid == 0 else os.waitpid(pid, 0)'
```

Output atteso:

```text
event=clone ... comm=python3 args=flags=...,stack=0x...,parent_tid=0x...,child_tid=0x...,tls=...,returnValue=child_pid:...
```

Su Linux/glibc, anche `fork()` puo' essere implementata internamente tramite
`clone`, quindi e' normale vedere `clone` invece di `fork`.

Per testare direttamente `vfork`, compilare un piccolo programma C:

```c
#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>

int main(void) {
    pid_t pid = vfork();
    if (pid == 0)
        _exit(0);
    waitpid(pid, 0, 0);
    return 0;
}
```

Poi:

```bash
gcc /tmp/test_vfork.c -o /tmp/test_vfork
sudo ./dist/project --events vfork --output table --comms test_vfork
/tmp/test_vfork
```

Output atteso:

```text
event=vfork ... comm=test_vfork args=returnValue=child_pid:...
```

## Testare le syscall `set*uid` e `set*gid`

Esempi rapidi, uno per evento. Tenere il tool aperto in un terminale:

```bash
sudo ./dist/project --events setuid,setgid,setreuid,setregid,setresuid,setresgid,setfsuid,setfsgid --output table --comms python3
```

Poi lanciare i comandi da un secondo terminale:

```bash
sudo python3 -c 'import os; os.setuid(65534); print(os.getuid(), os.geteuid())'
sudo python3 -c 'import os; os.setgid(65534); print(os.getgid(), os.getegid())'
sudo python3 -c 'import os; os.setreuid(0, 65534); print(os.getuid(), os.geteuid())'
sudo python3 -c 'import os; os.setregid(0, 65534); print(os.getgid(), os.getegid())'
sudo python3 -c 'import os; os.setresuid(0, 65534, 0); print(os.getresuid())'
sudo python3 -c 'import os; os.setresgid(0, 65534, 0); print(os.getresgid())'
sudo python3 -c 'import ctypes, os; libc=ctypes.CDLL(None, use_errno=True); print(libc.syscall(122, os.getuid()), ctypes.get_errno())'
sudo python3 -c 'import ctypes, os; libc=ctypes.CDLL(None, use_errno=True); print(libc.syscall(123, os.getgid()), ctypes.get_errno())'
```

Su x86_64, `122` e `123` sono rispettivamente `setfsuid` e `setfsgid`.

Output atteso:

```text
event=setuid ... args=uid=65534,returnValue=success
event=setgid ... args=gid=65534,returnValue=success
event=setresuid ... args=ruid=0,euid=65534,suid=0,returnValue=success
```

Questi eventi rappresentano il punto di vista syscall: argomenti richiesti e
risultato finale. Per vedere le credenziali effettivamente installate usare
anche `commit_creds`.

## Testare `prlimit64`

Terminale 1:

```bash
sudo ./dist/project --events prlimit64 --output table --comms prlimit
```

Terminale 2:

```bash
prlimit --nofile
prlimit --pid $$ --nofile=1024:4096
```

Output atteso:

```text
event=prlimit64 ... comm=prlimit args=pid=0,resource=RLIMIT_NOFILE(7),new_limit=0x0,old_limit=0x...,returnValue=success
event=prlimit64 ... comm=prlimit args=pid=...,resource=RLIMIT_NOFILE(7),new_limit=0x...,old_limit=0x0,returnValue=success
```

I campi `new_limit` e `old_limit` sono puntatori userspace. La prima versione
non dereferenzia il contenuto della struct `rlimit64`; questa scelta mantiene
l'evento leggero e vicino al modello Tracee-like entry/exit.

## Testare namespace, mount e unlink

Per osservare la transizione effettiva di namespace:

```bash
sudo ./dist/project --events switch_task_ns --output table --comms unshare
sudo unshare -m true
```

Output verificato:

```text
event=switch_task_ns ... comm=unshare args=pid=...,new_mnt=...
```

Per mount e umount usare un tmpfs temporaneo:

```bash
sudo ./dist/project --events security_sb_mount,security_sb_umount --output table
sudo mkdir -p /tmp/project-mount-test
sudo mount -t tmpfs -o nosuid,nodev tmpfs /tmp/project-mount-test
sudo umount /tmp/project-mount-test
sudo rmdir /tmp/project-mount-test
```

Le flag vengono mostrate come `MS_NOSUID|MS_NODEV`; per umount senza opzioni
il campo e' `flags=0`.

Per unlink:

```bash
sudo ./dist/project --events security_inode_unlink --output table --comms rm
touch /tmp/project-unlink-test
rm /tmp/project-unlink-test
```

L'evento contiene il path risolto, inode, device e ctime. Questi tre hook LSM
non portano il return value finale: descrivono il controllo security eseguito
dal kernel sull'operazione.

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

Nota: gli hook attuali inviano gli eventi tramite perf buffer. Il runtime apre
ancora anche `events_ringbuf`, ma il percorso operativo corrente usa la mappa
perf `events`.

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

La versione attuale apre ancora sia ring buffer sia perf buffer:

```text
events_ringbuf -> InitRingBuf
events         -> InitPerfBuf
```

La ring buffer resta disponibile per versatilita' e fallback futuri, mentre gli
hook correnti usano `events_perf_submit`.

Per verificare un evento sul percorso perf buffer:

```bash
make run ARGS="--events sched_process_exec --output table --comms ls"
```

Poi, da un altro terminale:

```bash
ls
```

Per verificare un evento networking:

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
