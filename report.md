# Report tecnico - Loader eBPF userspace e compilazione `project.bpf.c`

## 1. Obiettivo del lavoro

L'obiettivo operativo e' stato portare il progetto al punto in cui:

1. il programma eBPF `pkg/ebpf/c/project.bpf.c` viene compilato in un oggetto CO-RE;
2. lo userspace Go legge l'oggetto compilato;
3. la collection eBPF viene caricata nel kernel;
4. i programmi eBPF vengono attaccati ai rispettivi hook;
5. ring buffer `events_ringbuf` e perf buffer `events` vengono drenati dallo
   userspace.

Il lavoro e' ispirato a Tracee, ma adattato a un MVP piu' piccolo. La prima
versione del runtime usava `github.com/cilium/ebpf`; dopo il merge con il
branch networking il runtime e' stato migrato verso
`github.com/aquasecurity/libbpfgo`, piu' vicino alla struttura di Tracee.

## 2. Contesto tecnico

Target principale:

- OS: Rocky Linux 8.10 / RHEL-compatible
- Kernel: `4.18.0-553.109.1.el8_10.x86_64`
- Feature eBPF disponibili/usate sul target: BTF/CO-RE, raw tracepoint, kprobe,
  ring buffer, perf event array
- BTF nativo atteso in: `/sys/kernel/btf/vmlinux`

Il progetto genera l'oggetto eBPF in `dist/project.bpf.o` e lo include nel
binario Go tramite `go:embed`. Il path esplicito resta disponibile per debug
tramite CLI o variabile d'ambiente.

## 3. Startup userspace

### 3.1 Configurazione

La struct `config.Config` contiene ora:

- `BPFObjPath`: path dell'oggetto eBPF compilato;
- `BTFObjPath`: path del file BTF;
- `BPFObjBytes`: contenuto dell'oggetto eBPF letto da disco;
- `Output`;
- `Events`, incluse selezione eventi e filtro `comm`;
- `LogLevel`.

La validazione statica controlla solo valori come output format e log level. La risoluzione dei path avviene in una fase successiva.

### 3.2 Inizializzazione dell'oggetto eBPF

E' stato introdotto `pkg/cmd/initialize/bpfobject.go`, che:

- legge `PROJECT_BPF_OBJECT`, se impostata;
- altrimenti usa il path CLI/config, se fornito;
- se non viene fornito nessun path, usa l'oggetto embedded;
- legge i byte dell'oggetto eBPF;
- risolve il BTF path usando `PROJECT_BTF_FILE`, configurazione o default `/sys/kernel/btf/vmlinux`.

Questo permette di lanciare il programma dalla root del repository con:

```bash
make run
```

oppure, dopo la build:

```bash
sudo ./dist/project
```

Il path esplicito resta supportato per debug.

## 4. Runtime eBPF Go

E' stato introdotto `pkg/ebpf/project.go`, che contiene il ciclo di vita eBPF:

1. `New(cfg)` valida che la config sia pronta;
2. `Init(ctx)` carica la collection e apre ring buffer e perf buffer;
3. `Run(ctx)` legge da entrambi i canali finche' il contesto viene cancellato;
4. `Close()` chiude link, ring buffer, perf buffer e modulo eBPF.

### 4.1 Rimozione del limite memlock

Prima del load della collection viene chiamato:

```go
rlimit.RemoveMemlock()
```

Questo evita errori come:

```text
map create: operation not permitted (MEMLOCK may be too low)
```

Nota: questa chiamata richiede privilegi adeguati. Nella sandbox Codex fallisce per mancanza di capability, mentre sulla VM va eseguita con `sudo`.

### 4.2 Attach esplicito dei programmi

Un punto fondamentale: caricare una collection eBPF non basta. I programmi devono essere attaccati ai rispettivi hook.

Sono stati aggiunti due gruppi:

- raw tracepoint:
  - `tracepoint__sched__sched_process_fork` -> `sched_process_fork`
  - `tracepoint__sched__sched_process_exec` -> `sched_process_exec`
  - `tracepoint__sched__sched_process_exit` -> `sched_process_exit`
  - `tracepoint__task__task_rename` -> `task_rename`

- syscall tracepoint:
  - `tracepoint__syscalls__sys_enter_execve` -> `execve`
  - `tracepoint__syscalls__sys_enter_execveat` -> `execveat`

- kprobe:
  - `trace_cap_capable` -> `cap_capable`
  - `trace_security_task_setrlimit` -> `security_task_setrlimit`
  - `trace_security_settime64` -> `security_settime64`
  - `trace_security_task_prctl` -> `security_task_prctl`

Nel codice Go gli attach passano ora dal registry in
`pkg/ebpf/probes/probes.go` e dalle API `libbpfgo`.

Nella prima versione gli attach usavano:

```go
link.AttachRawTracepoint(...)
link.Kprobe(...)
```

Ora il concetto resta lo stesso, ma gli handle sono gestiti tramite `libbpfgo`.

## 5. Confronto con Tracee

### 5.1 Pattern rispettati

Tracee definisce i probe in `pkg/ebpf/probes/probe_group.go` e poi li attacca in `pkg/ebpf/probes/trace.go`.

Pattern equivalente:

- Tracee registra coppie evento-programma;
- il nostro MVP registra coppie hook-programma in slice statiche;
- Tracee conserva gli handle dei link;
- il nostro MVP conserva `[]link.Link`;
- Tracee distrugge i link in detach;
- il nostro MVP chiude i link in `Close()`.

Esempi Tracee:

- `SchedProcessFork`: raw tracepoint `sched:sched_process_fork`
- `SchedProcessExec`: raw tracepoint `sched:sched_process_exec`
- `SchedProcessExit`: raw tracepoint `sched:sched_process_exit`
- `TaskRename`: raw tracepoint `task:task_rename`
- `CapCapable`: kprobe `cap_capable`
- `SecurityTaskSetrlimit`: kprobe `security_task_setrlimit`
- `SecuritySettime64`: kprobe `security_settime64`

Il nostro codice replica lo stesso concetto, ma senza introdurre ancora un sistema completo di `ProbeGroup`.
In piu', per il target Rocky/RHEL noto, usa tracepoint syscall dedicati per
`execve` ed `execveat`, evitando un secondo dispatcher generico su
`raw_tracepoint/sys_enter`.

### 5.2 Differenze importanti

La differenza iniziale era che Tracee usava `libbpfgo`, mentre il progetto
usava `cilium/ebpf`. Questa distanza e' stata ridotta con la migrazione del
runtime verso `libbpfgo`. Restano comunque differenze importanti:

- Tracee ha una pipeline eventi e policy piu' completa;
- Tracee gestisce i kprobe tramite symbol table e puo' attaccarsi per address se un simbolo ha piu' indirizzi;
- il progetto usa ancora un registry statico piu' semplice;
- il progetto deve ancora decidere come uniformare ring buffer e perf buffer.

Tracee usa inoltre un'infrastruttura molto piu' ampia:

- autoload per probe;
- compatibilita' kernel;
- tail calls;
- policy/filtering;
- metriche;
- gestione eventi completa;
- decoder userspace.

Il progetto mantiene solo il necessario per un MVP.

## 6. Compilazione di `project.bpf.c`

Comando usato:

```bash
clang -g -O2 -target bpf -D__TARGET_ARCH_x86 -I pkg/ebpf/c \
  -c pkg/ebpf/c/project.bpf.c -o pkg/ebpf/c/project.bpf.o
```

Output atteso:

```text
pkg/ebpf/c/project.bpf.o
```

Al momento l'oggetto viene generato correttamente.

## 7. Problemi incontrati in compilazione C

### 7.1 `bpf_task_pt_regs` non dichiarata

Errore iniziale:

```text
call to undeclared function 'bpf_task_pt_regs'
```

Tracee prova a usare `bpf_task_pt_regs()` quando disponibile, con fallback manuale. Sulla toolchain della VM, l'enum helper esiste nel BTF/header, ma la funzione non e' dichiarata dagli header libbpf installati.

Scelta fatta per MVP:

- rimuovere il path `bpf_task_pt_regs()`;
- usare sempre il fallback manuale basato su `task->stack + THREAD_SIZE`.

Questa scelta e' meno portabile di Tracee, ma adatta al target Rocky/RHEL del progetto.

### 7.2 Firma di `bpf_core_field_exists`

Errore iniziale:

```text
too many arguments provided to function-like macro invocation
```

Il codice usava:

```c
bpf_core_field_exists(struct task_struct, start_boottime)
```

La macro installata sulla VM accetta un solo argomento. Il codice e' stato adattato a:

```c
bpf_core_field_exists(task->start_boottime)
```

La semantica resta la stessa: controllare se il campo `start_boottime` esiste nel `task_struct` del kernel target.

### 7.3 `barrier()` non definita

Errore iniziale:

```text
call to undeclared function 'barrier'
```

E' stata aggiunta in `common/common.h`:

```c
#ifndef barrier
#define barrier() asm volatile("" ::: "memory")
#endif
```

## 8. Errori del verifier eBPF

Dopo la compilazione, il caricamento nel kernel ha evidenziato diversi errori del verifier.

### 8.1 Errore su `tracepoint__sched__sched_process_exec`

Errore:

```text
invalid access to map value, value_size=4280 off=65664 size=1
```

Interpretazione:

- il verifier vedeva `buf->offset` come `u16` potenzialmente fino a `65535`;
- l'accesso a `buf->args[buf->offset]` poteva quindi uscire dai limiti della scratch map;
- il valore `65664` deriva dall'offset base dentro `event_data_t` piu' un possibile `65535`.

Mitigazione:

- leggere `buf->offset` in una variabile locale `off`;
- controllare `off` contro `ARGS_BUF_SIZE`;
- usare `off` per indicizzare il buffer.

### 8.2 Errore su `trace_cap_capable`

Errore:

```text
invalid access to map value, value_size=4280 off=65664 size=4
```

Questo errore avveniva nel path `save_to_submit_buf`, usato per argomenti scalari.

Mitigazione:

- sostituire `bpf_probe_read` con copie a dimensione costante via `__builtin_memcpy`;
- gestire esplicitamente dimensioni supportate:
  - `u8`
  - `u16`
  - `u32`
  - `u64`

Questo aiuta il verifier perche' la dimensione della copia diventa costante in ogni branch.

### 8.3 Errore su `tracepoint__task__task_rename`

Errore:

```text
invalid access to map value, value_size=4280 off=65664 size=4096
```

Questo errore riguardava `save_str_to_buf`: il verifier vedeva una possibile scrittura di `4096` byte a offset variabile.

Mitigazione:

- introdotto `MAX_ARG_STRING_SIZE = 512`;
- limitata la dimensione effettiva passata a `bpf_probe_read_str`;
- mantenuti i controlli su `off` e sullo spazio residuo.

Trade-off:

- path/nomi lunghi possono essere troncati a 512 byte;
- per il MVP e' accettabile;
- in futuro si puo' reintrodurre un path buffer dedicato, come fa Tracee, oppure usare una strategia a chunk.

### 8.4 Errore su `tracepoint__sched__sched_process_exit`

Errore:

```text
invalid access to map value, value_size=4280 off=65664 size=1
```

Questo errore e' emerso dopo le prime mitigazioni sui writer degli argomenti. La causa piu' probabile non e' piu' solo la scrittura dentro `args`, ma anche la size passata a:

```c
bpf_ringbuf_output(&events, event, size, 0)
```

In `events_ringbuf_submit()` la size era calcolata a partire da:

```c
event->args_buf.offset
```

`offset` e' un `u16`, quindi il verifier lo considera potenzialmente fino a `65535`. Anche se a runtime il codice prova a mantenerlo entro `ARGS_BUF_SIZE`, il verifier deve poterlo dimostrare localmente nel punto della chiamata alla helper. In caso contrario vede una possibile copia dalla scratch map fino a offset `65664`, fuori da `value_size=4280`.

Mitigazione applicata:

- leggere `event->args_buf.offset` in `args_off`;
- controllare esplicitamente `args_off <= ARGS_BUF_SIZE`;
- calcolare `size` usando `args_off`;
- controllare esplicitamente `size <= MAX_EVENT_SIZE`;
- rimuovere il clamp `update_min(size, MAX_EVENT_SIZE)` in questo punto, preferendo branch espliciti piu' leggibili per il verifier.

Nuovo schema:

```c
u32 args_off = event->args_buf.offset;
if (args_off > ARGS_BUF_SIZE)
    return 0;

u32 size = sizeof(event_context_t) + sizeof(u8) + args_off;
if (size > MAX_EVENT_SIZE)
    return 0;
```

Questa modifica segue lo stesso principio delle mitigazioni precedenti: rendere i limiti locali e immediatamente visibili al verifier prima di ogni accesso/copia su memoria eBPF.

### 8.5 Secondo errore su `tracepoint__task__task_rename`

Errore:

```text
invalid access to map value, value_size=4280 off=65664 size=512
```

Anche dopo aver ridotto la size massima delle stringhe da 4096 a 512 byte, il verifier continuava a rifiutare `bpf_probe_read_str()`. Il problema residuo era la destinazione:

```c
&(buf->args[off + 1 + sizeof(u32)])
```

`off` derivava ancora dallo stato dinamico del buffer. Anche se il codice controllava `off`, il verifier non riusciva a dimostrare in modo stabile che la destinazione della helper restasse sempre dentro la `scratch_map`.

Mitigazione applicata:

- introdotti slot stringa a offset costante;
- ogni stringa occupa uno slot fisso:

```c
1 byte type tag + 4 byte length + 512 byte payload
```

- massimo iniziale: `MAX_STRING_ARGS = 4`;
- `save_str_to_buf()` sceglie lo slot usando `argnum`:
  - argomento 0 -> offset `0`;
  - argomento 1 -> offset `STRING_ARG_SLOT_SIZE`;
  - argomento 2 -> offset `STRING_ARG_SLOT_SIZE * 2`;
  - argomento 3 -> offset `STRING_ARG_SLOT_SIZE * 3`.

Trade-off:

- il wire format delle stringhe non e' piu' completamente compatto: ogni stringa riserva 512 byte anche se la stringa reale e' piu' corta;
- il campo length resta presente e indica quanti byte sono realmente significativi;
- per il MVP questo e' accettabile per sbloccare il verifier;
- quando verra' implementato il decoder Go, dovra' sapere che gli argomenti stringa usano slot fissi oppure il formato dovra' essere riportato a una serializzazione compatta con una strategia verifier-safe piu' raffinata.

### 8.6 Secondo errore su `tracepoint__sched__sched_process_exit`

Errore:

```text
invalid access to map value, value_size=4280 off=65664 size=1
```

Questo errore e' ricomparso su `sched_process_exit` dopo la modifica agli slot stringa. L'evento `sched_process_exit` serializza argomenti scalari:

- `exit_code` come `s64`;
- `group_dead` come `u8`.

Il rifiuto sul secondo argomento (`size=1`) indica che anche `save_to_submit_buf()` era ancora troppo dinamica per il verifier. Nonostante i controlli su `off`, l'offset effettivo continuava a derivare da `buf->offset`, campo residente nella scratch map e considerato potenzialmente fino a `65535`.

Mitigazione applicata:

- anche gli argomenti scalari vengono ora scritti in slot fissi;
- ogni slot scalare occupa `16` byte;
- massimo iniziale: `MAX_SCALAR_ARGS = 8`;
- lo slot viene scelto tramite `argnum`, con offset costanti:
  - argomento 0 -> `0`;
  - argomento 1 -> `16`;
  - argomento 2 -> `32`;
  - ecc.

Il writer continua a salvare:

- `type_tag` nel primo byte dello slot;
- valore scalare subito dopo il tag;
- padding inutilizzato nel resto dello slot.

Trade-off:

- il formato wire diventa meno compatto anche per gli scalari;
- ogni argomento scalare occupa sempre 16 byte, anche se il valore e' un `u8`;
- per il MVP questo e' accettabile per rendere gli offset dimostrabili dal verifier;
- il decoder Go dovra' conoscere questa forma a slot fissi oppure verra' riprogettato quando il protocollo sara' stabilizzato.

## 9. Stato attuale

Verifiche locali completate:

```bash
make bpf
```

Risultato: compilazione riuscita.

```bash
make build
```

Risultato: build Go riuscita con `libbpfgo` e oggetto eBPF in `dist/`.

Il comando da eseguire sulla VM e':

```bash
cd /home/simone/project/demo_project
make run
```

Ultimo esito osservato sulla VM:

```text
event=sched_process_exec pid=... tid=... uid=1000 comm=ls args=filename=/usr/bin/ls
```

Questo indica che il programma e' riuscito a superare le fasi precedenti che fallivano:

- lettura dell'oggetto eBPF;
- load della collection;
- apertura di ring buffer e perf buffer;
- attach dei programmi eBPF;
- ingresso nel loop runtime.

Il fatto che non vengano stampati alert non e' un errore in questa fase: gli
alert richiedono ancora detection logic. Dopo gli ultimi aggiornamenti, pero',
il runtime Go non scarta piu' i record: li riceve sia da ring buffer sia da perf
buffer, li decodifica, applica la selezione eventi, applica l'eventuale filtro
`comm` e li passa al layer `pkg/output`.

Loop aggiornato in `pkg/ebpf/project.go`:

```go
select {
case raw := <-p.ringBufChannel:
    p.handleRawEvent(raw)
case raw := <-p.perfBufChannel:
    p.handleRawEvent(raw)
}
```

Quindi, se un evento arriva da uno dei due canali, viene decodificato e viene
emessa una riga nel formato configurato (`json` o `table`).

Conclusione dello stato corrente:

- il loader eBPF e' arrivato a uno stato operativo;
- la pipeline userspace minimale degli eventi e' presente;
- gli hook producono eventi decodificati e stampabili;
- il runtime riceve sia eventi ring buffer sia eventi perf buffer;
- l'output e' separato dal runtime eBPF;
- per ottenere alert servono ancora enrichment piu' ampio e detection logic.

## 10. Limitazioni attuali

### 10.1 Decoder MVP

Il runtime ora decodifica gli eventi con il package:

```text
demo_project/pkg/bufferdecoder/
```

Il decoder implementa:

- parsing di `event_context_t` da 128 byte;
- parsing di `argnum`;
- parsing degli argomenti taggati a slot fissi (`INT_T`, `UINT_T`, `STR_T`, ecc.);
- produzione di `bufferdecoder.Event`, poi passato al layer `pkg/output`.

Esempio JSON normalizzato:

```json
{"timestamp":4074609472044753,"event_name":"cap_capable","process":{"pid":1374462,"tid":1374462,"ppid":1348641,"uid":1000,"comm":"cpuUsage.sh","uts_name":"security-thesis"},"host":{"pid":1374462,"tid":1374462,"ppid":1348641},"kernel":{"syscall":56,"processor_id":0,"mnt_id":4026531840,"pid_id":4026531836},"args":[{"name":"cap","type":1,"value":21,"label":"CAP_SYS_ADMIN"}]}
```

Limitazioni residue:

- syscall, resource limit e opzioni `prctl` sono ancora numeriche;
- nuovi hook richiedono aggiornamento dello schema statico in `protocol.go`.

### 10.2 Attach kprobe semplificato

Il progetto usa `link.Kprobe(symbol, prog, nil)`.

Tracee e' piu' robusto:

- legge la kernel symbol table;
- gestisce simboli multipli;
- puo' attaccare tramite address.

Miglioramento futuro:

- introdurre un piccolo `ProbeGroup`;
- opzionalmente leggere `/proc/kallsyms`;
- gestire fallback se un simbolo non esiste.

### 10.3 Stringhe troncate

Le stringhe serializzate sono limitate a 512 byte per evitare errori del verifier.

Per l'MVP e' accettabile, ma per exec path completi e argv/env sara' necessario un meccanismo migliore.

### 10.4 Selezione eventi ancora semplice

La CLI permette di usare `--events` e `--drop-events`, ma non esiste ancora un
policy manager completo.

Tracee invece abilita/disabilita probe in base agli eventi richiesti e alla configurazione.

### 10.5 Output arricchito ma non completo

Il runtime stampa eventi JSON o table decodificati. L'output e' piu' leggibile
rispetto al JSON raw iniziale, ma resta pensato per debugging:

- una riga per evento;
- filtri disponibili per include/exclude eventi e `comm`;
- enrichment presente per capability;
- mapping ancora mancante per `prctl`, socket family/type, resource limit e
  syscall.

## 11. Prossimi step consigliati

1. Decidere il canale eventi unico: ring buffer, perf buffer o entrambi.
2. Aggiungere lost channel e metriche per il perf buffer.
3. Decidere se migrare gli hook networking a `events_ringbuf_submit` oppure
   convergere tutto verso perf buffer in stile Tracee.
4. Aggiungere mapping human-readable per `prctl`, socket family/type, resource
   limit e syscall.
5. Aggiungere filtri per UID, PID o processo padre.
6. Aggiungere test Go per:
   - risoluzione path BPF/BTF;
   - validazione config;
   - decoder del wire format.

## 12. Comandi utili

Compilazione eBPF:

```bash
make bpf
```

Build/test Go:

```bash
make build
```

Run:

```bash
make run
```

Build binario:

```bash
sudo ./dist/project --events sched_process_exec --output table
```

## 13. Sintesi

Il progetto e' passato da uno userspace che restava semplicemente in attesa a un runtime che:

- prepara la config;
- legge l'oggetto eBPF embedded o da path esplicito;
- carica la collection;
- apre la ring buffer tramite `libbpfgo`;
- apre il perf buffer `events` tramite `libbpfgo`;
- attacca i programmi eBPF;
- gestisce cleanup ordinato.

Le modifiche sono coerenti con l'architettura di Tracee a livello di pattern, ma restano volutamente piu' semplici per l'MVP della tesi.

Il punto raggiunto e' importante: il problema non e' piu' solo il caricamento
dell'eBPF, ma l'allineamento tra canale kernel/userspace, decoder, output
human-readable e futuri alert.

## 14. Buffer.h

Sono stati limitati i possibili valori di offset sul buffer, in entrambe le funzioni, anche per quella di str. sono state cambiate le funzioni di save per rientrare nei limit del verifier.

## 15. common.h
Aggiunta la funzione barrier, che aiuta a mantenere più prevedibile l'ordine delle operazioni, è più una nota al compilatore per non fare ottimizzazioni troppo creative attraverso questo punto.
