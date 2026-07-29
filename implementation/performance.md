# Misurazione delle prestazioni

## Obiettivo

Il target operativo iniziale e' mantenere il costo medio del tool sotto il
`5%` di un singolo core. La misura deve includere due componenti distinte:

```text
processo Go userspace + programmi eBPF eseguiti nel kernel
```

`ps` e `top` mostrano principalmente il costo userspace. Il tempo consumato
dai programmi eBPF deve essere letto separatamente dalle statistiche kernel.

## Monitoraggio userspace

Il progetto include uno script ripetibile per il benchmark userspace:

```bash
cd demo_project
make benchmark-userspace
```

Valori configurabili:

```bash
DURATION_SECONDS=120 WARMUP_SECONDS=10 CPU_THRESHOLD=5.0 make benchmark-userspace
```

Lo script cerca il processo piu' recente chiamato `project`, legge i tick CPU da
`/proc/<pid>/stat` e calcola la CPU come percentuale di un singolo core. Alla
fine stampa media, p95, picco, RSS massimo e thread massimi. Se la media supera
`CPU_THRESHOLD`, il comando termina con exit code `2`.

`WARMUP_SECONDS` permette di separare startup, attach e inizializzazione dalla
misurazione steady-state. I campioni di warm-up vengono stampati ma sono esclusi
da `avg_cpu`, `p95_cpu` e dalla valutazione della soglia.

Esempio di output:

```text
benchmark pid=1234 process=project duration=120s warmup=10s threshold=5.0%
time       phase     cpu%     rss_kib    nlwp     elapsed
10:00:01   warmup    8.01     59000      10       00:01
10:00:12   measure   2.01     59000      10       00:12

summary pid=1234 warmup_samples=10 measured_samples=110 avg_cpu=2.35% p95_cpu=4.80% peak_cpu=6.01% peak_rss_kib=59000 peak_nlwp=10 threshold=5.0%
```

Per osservare CPU, memoria, thread e tempo di esecuzione:

```bash
watch -n 1 'ps -C project -o pid,%cpu,%mem,rss,nlwp,etime,cmd'
```

Campi principali:

- `%CPU`: media CPU del processo;
- `RSS`: memoria residente in KiB;
- `NLWP`: numero di thread;
- `ETIME`: tempo trascorso dall'avvio.

Per un valore aggiornato ogni secondo:

```bash
sudo top -d 1 -p "$(pgrep -n project)"
```

I picchi iniziali includono caricamento BTF, verifier, attach dei programmi e
inizializzazione dei buffer. Il valore steady-state va osservato dopo almeno
uno o due minuti.

## Monitoraggio eBPF kernel-side

Il kernel target espone le statistiche runtime:

```bash
sudo sysctl kernel.bpf_stats_enabled
```

Il valore atteso e':

```text
kernel.bpf_stats_enabled = 1
```

Per elencare i programmi appartenenti al processo più recente:

```bash
pid="$(pgrep -n project)"
sudo bpftool -j prog show |
  jq --argjson pid "$pid" '
    [.[] | select(any(.pids[]?; .pid == $pid)) |
      {id, name, type, run_cnt, run_time_ns}]'
```

`run_time_ns` e' cumulativo. Per ottenere il costo medio bisogna confrontare
due snapshot separati da un intervallo noto:

```text
CPU eBPF % di un core =
  delta run_time_ns / durata intervallo in nanosecondi * 100
```

## Risultato del 10 giugno 2026

Configurazione:

- tutti gli eventi abilitati;
- 56 programmi eBPF caricati;
- VM con 2 CPU;
- kernel `4.18.0-553.109.1.el8_10.x86_64`.

Campione di 30 secondi:

| Componente | Media |
|---|---:|
| Processo Go | 2.20% di un core |
| Programmi eBPF | 1.19% di un core |
| Totale stimato | 3.39% di un core |
| RSS | circa 58 MiB |

Sono stati osservati picchi userspace brevi fino al `13%`. In un intervallo
precedente con maggiore attivita' di sistema, il costo eBPF ha superato
temporaneamente il `15%` di un core. Il dato medio va quindi accompagnato da
test sotto workload controllato.

## Risultato del 14 luglio 2026

Profilo misurato:

```bash
sudo ./dist/project \
  --events security_task_fix_setuid,sched_process_exec \
  --policy rules/policies/demo-detectors.yaml \
  --detectors rules/detectors/privilege_exec_chain.yaml \
  --alerts-only \
  --alerts-output table \
  --log-level error
```

Benchmark userspace:

```bash
DURATION_SECONDS=120 CPU_THRESHOLD=5.0 make benchmark-userspace
```

Campioni osservati:

| Campo | Valore osservato |
|---|---:|
| CPU steady-state | spesso tra `5.9%` e `7.7%` |
| Picco osservato | `16.92%` |
| RSS stabilizzato | circa `45 MiB` |
| Thread | `8-9` |
| Target | `<5%` medio di un core |

La riga finale `summary` non e' stata riportata, quindi il dato non viene
considerato benchmark completo. Tuttavia i campioni mostrano che, in questo
profilo, la CPU userspace resta sopra il target per lunghi tratti. Il profilo
va quindi considerato non conforme finche' non viene ottimizzato o rimisurato
con una policy piu' stretta.

Aspetto importante: questo benchmark misura solo il processo Go userspace. Il
costo eBPF kernel-side va ancora stimato separatamente con `bpftool`.

## Interpretazione

Il target medio del `5%` e' raggiungibile, ma non e' garantito quando vengono
abilitati tutti gli hook. I programmi piu' frequenti dipendono dal workload:

- `cap_capable` puo' essere molto rumoroso su sistemi attivi;
- `security_file_open` e `security_file_permission` crescono rapidamente con
  build, shell interattive e processi che aprono molti file;
- syscall enter/exit come `open`, `mmap`, `mprotect`, `clone` e famiglia
  `set*id` aggiungono costo in base al numero di chiamate;
- hook di hardening kernel come `register_kprobe`, `proc_create` e
  `kallsyms_lookup_name` sono invece piu' rari ma ad alto valore semantico.

Le ottimizzazioni prioritarie sono:

1. abilitare solo gli eventi richiesti da una policy;
2. evitare raw syscall globali quando esiste un tracepoint dedicato;
3. introdurre filtri kernel-side per UID, PID, namespace e `comm`;
4. misurare eventi persi e throughput del perf buffer;
5. confrontare sempre la stessa attività con tool spento e acceso.

## Filtro UID kernel-side

Il primo filtro kernel-side implementato e' volutamente minimale: permette di
accettare solo eventi generati da un UID specifico prima che l'hook costruisca
il payload completo e lo invii al perf buffer.

Uso:

```bash
sudo ./dist/project \
  --events security_file_open,sched_process_exec \
  --kernel-filter-uid-enabled \
  --kernel-filter-uid 1000 \
  --output table \
  --log-level error
```

Implementazione:

- la CLI popola `cfg.Kernel`;
- `pkg/ebpf/project.go` aggiorna `config_map` dopo `BPFLoadObject`;
- `config_entry_t` contiene `kernel_uid_filter_enabled` e
  `kernel_uid_filter`;
- `init_program_data()` applica il controllo subito dopo il lookup di
  `config_map`;
- se l'UID corrente non coincide, l'hook ritorna prima di inizializzare context,
  process map, argomenti e submit.

Questo filtro non sostituisce policy e detector. Serve solo a ridurre traffico
kernel-to-userspace per benchmark e run mirate. La policy userspace continua a
decidere quali eventi sono semanticamente rilevanti.

Limite attuale: il filtro e' una allowlist singola. Non supporta ancora range,
liste multiple, namespace o `comm`.

## Benchmark parziale del 16 luglio 2026

Dopo l'introduzione del filtro UID kernel-side sono stati confrontati tre
profili manuali. I comandi sono stati interrotti prima dei 120 secondi, quindi i
numeri vanno letti come campioni esplorativi e non come benchmark finale.

| Caso | Profilo | CPU osservata | RSS osservato | Esito |
|---|---|---:|---:|---|
| 1 | Policy/detector collective con alert-only | spesso `6.8% - 8.5%` | circa `82 MiB` | sopra target |
| 2 | Eventi rumorosi senza filtro UID | spesso `2% - 3.7%`, picco `9.15%` | circa `81 MiB` | quasi sotto target |
| 3 | Eventi rumorosi con filtro UID kernel-side | spesso `2% - 3.7%`, picco `4.59%` | circa `48 MiB` | sotto target nei campioni |

Interpretazione:

- il filtro UID e' efficace nel ridurre il traffico verso userspace e la
  memoria residente;
- il profilo con policy e detector resta il piu' costoso, quindi il collo di
  bottiglia principale non e' solo l'hook eBPF, ma anche decode, detector
  stateful, dedup, costruzione alert e output;
- `--alerts-only` riduce la stampa degli eventi raw, ma ogni evento necessario
  ai detector viene comunque letto, decodificato e valutato;
- il filtro UID va scelto in base al detector: UID `1000` e' utile per run
  mirate sull'utente, mentre detector root-oriented richiedono UID `0` o nessun
  filtro UID.

Limite del test: il benchmark e' stato interrotto dopo circa 30 secondi. Per
una misura difendibile vanno completati i 120 secondi e usata la summary dello
script aggiornata, che riporta media, picco e p95 escludendo il warm-up.

Prossima ottimizzazione: isolare il costo del detector engine con profili
separati, ora che lo script produce statistiche aggregate (`avg`, `max`,
`p95`) in modo ripetibile.

## Ottimizzazione dispatcher detector

L'audit del detector engine ha mostrato che il dispatcher era gia' impostato
nel modo corretto: non invia ogni evento a tutti i detector, ma costruisce un
indice:

```text
event_name -> detector interessati
```

Questa scelta e' fondamentale per la scalabilita'. Se arrivano molti eventi
`security_file_open`, vengono valutati solo i detector che dichiarano di
consumare `security_file_open`, piu' eventuali detector wildcard.

La modifica del 16 luglio consolida questa architettura con due interventi:

1. riduzione di allocazioni inutili nella hot path del dispatcher, evitando
   slice preallocate quando non ci sono alert o errori;
2. metriche piu' esplicite:

```text
DetectorMatched
DetectorInvoked
DetectorSkipped
```

`DetectorSkipped` e' la metrica chiave per capire quanto l'indice stia
risparmiando lavoro. Se il valore cresce, significa che il dispatcher sta
evitando valutazioni inutili. Se resta basso in profili rumorosi, bisogna
controllare se troppi detector sono wildcard o dichiarano eventi troppo larghi.

Il benchmark post-modifica sul profilo `collective` mostra ancora campioni
stabili tra circa `6%` e `7.7%` CPU, con RSS intorno a `84 MiB`. Questo conferma
che il routing detector non e' il collo di bottiglia principale: il costo piu'
probabile deriva da volume eventi, detector troppo larghi come `root-exec`,
decode completo, costruzione alert e output.

La prossima mitigazione non deve quindi concentrarsi sul dispatcher, ma sulla
selettivita' delle policy/detector usate nei profili realistici.

## Compilazione delle condizioni e pruning periodico

Il 24 luglio il percorso runtime dei detector YAML e' stato alleggerito in due
passaggi.

Le condizioni YAML vengono ora compilate durante `NewDetector()`. I nomi degli
operatori sono trasformati in costanti interne, le liste dell'operatore `in`
sono separate una sola volta e i valori attesi di `gt` e `lt` sono convertiti
in numero al caricamento. Il formato YAML e il comportamento degli alert non
cambiano, ma il runtime non deve ripetere questo lavoro per ogni evento.

Per i detector collective, la pulizia dello stato non attraversa piu' tutta la
mappa a ogni evento. Ogni detector memorizza:

```text
stateWindow
statePruneInterval
nextStatePrune
```

L'intervallo e' pari a meta' della finestra del detector, con un massimo di un
secondo:

```text
prune_interval = min(window / 2, 1s)
```

Il controllo della sequenza direttamente interessata dall'evento rimane
puntuale. Le altre sequenze scadute vengono eliminate alla scadenza periodica,
mentre `Flush()` forza sempre una pulizia completa. Questa separazione conserva
la correttezza temporale ed evita scansioni lineari ripetute su stato non
correlato all'evento corrente.

Microbenchmark locale su AMD EPYC 7B12, cinque esecuzioni da due secondi:

| Stato aperto | Mediana indicativa | Memoria | Allocazioni |
| --- | ---: | ---: | ---: |
| 1 sequenza | circa `249 ns/op` | circa `64 B/op` | `5 allocs/op` |
| 1024 sequenze | circa `241 ns/op` | `64 B/op` | `5 allocs/op` |

Il risultato mostra che, tra due pruning, il numero di sequenze aperte non
introduce piu' una scansione completa per evento. Le oscillazioni osservate nei
tempi sono normali per un percorso cosi' breve; il segnale stabile e' che
memoria e allocazioni non crescono con la dimensione della mappa. I numeri sono
microbenchmark di sviluppo e non dimostrano da soli il rispetto del target
`<5%`: il profilo `collective` deve essere nuovamente misurato per 120 secondi,
senza interruzioni, usando lo stesso workload dei test precedenti.

Dal 29 luglio i nomi `group_by` vengono compilati una volta in un piano di
correlazione. La hot path costruisce soltanto i componenti necessari della
chiave corrente. La mappa delle sequenze incomplete e' inoltre limitata a 4096
entry per detector; oltre il limite viene espulsa la sequenza piu' vecchia.

L'introduzione di `resource`, `cgroup` e chiavi composite non dimostra da sola
il rispetto del target `<5%`. Il profilo `collective` deve essere rieseguito con
il nuovo detector `temp-script-write-exec` e con un workload ripetibile.

Per il risultato del 14 luglio, i punti critici piu' probabili lato userspace
sono:

1. Decode completo prima dei filtri: ogni record viene trasformato in evento
   completo, inclusi argomenti, prima che policy e detector decidano se serve.
2. Buffer non selettivo: se nel perf buffer arrivano eventi prodotti da hook
   non necessari, il costo di lettura e decode viene comunque pagato.
3. Detector stateful e dedup: sono feature utili, ma aggiungono lookup, tempo
   corrente, costruzione chiavi e stato in memoria per ogni evento consegnato al
   detector engine.
4. Output alert: `--alerts-only` evita gli eventi raw, ma ogni alert richiede
   comunque formattazione della sequenza e degli argomenti.
5. Selezione programmi eBPF: le allowlist evento esplicite delle policy vengono
   ora compilate nella selezione di autoload e attach. I programmi pubblici non
   richiesti non vengono caricati, evitando record inutili prima del decoder.

## Logging e overhead

Il logging runtime puo' influire sulle misure se viene usato in modo troppo
verboso. Per questo il piano di migrazione verso `zap` deve rispettare due
regole:

- `info` deve contenere solo lifecycle essenziale;
- `debug` deve essere considerato modalita' diagnostica, non profilo normale.

Nella hot path:

```text
raw event -> decode -> filters -> output -> detectors
```

non bisogna costruire stringhe pesanti quando il livello debug e' disabilitato.
I campi diagnostici devono essere passati come campi strutturati del logger,
per esempio `event_id`, `event_name`, `comm` e `drop_reason`.

Durante i benchmark vanno confrontate almeno tre configurazioni:

```bash
sudo ./dist/project --events execve,security_file_open --output table --log-level error
sudo ./dist/project --events execve,security_file_open --output table --log-level info
sudo ./dist/project --events execve,security_file_open --output table --log-level debug
```

Il risultato atteso e' che `error` e `info` abbiano overhead simile nelle run
normali, mentre `debug` puo' costare di piu' ed e' accettabile solo per
diagnostica mirata.

Nota sui detector stateful: la prima versione del contratto permette finestre
temporali brevi per correlare sequenze locali di eventi. La finestra di default
e' `2s`, con limite massimo `5s`. Questa scelta serve a coprire collective
anomalies locali, non contextual anomalies globali. Se i benchmark superano
stabilmente il target del `5%`, questa feature non deve essere considerata
essenziale: va ridotta, abilitata solo per detector selezionati o disabilitata
temporaneamente.

Per demo o benchmark manuali e' consigliato partire sempre da un set ristretto:

```bash
sudo ./dist/project --events execve,open,security_bprm_check --output table
```

e abilitare eventi ad alto volume solo quando servono davvero:

```bash
sudo ./dist/project --events security_file_open,security_file_permission --comms cat,bash --output table
```

## Metodo consigliato per benchmark

1. Compilare il tool:

```bash
cd demo_project
GOCACHE=/tmp/go-build make build
```

2. Avviare il tool con un profilo preciso. Il repository espone ora profili
standard tramite:

```bash
make benchmark-profile PROFILE=<profilo>
```

Profili disponibili:

| Profilo | Scopo |
|---|---|
| `raw` | Misura eventi raw ad alto volume senza policy/detector. |
| `point` | Misura detector puntuali e alert semplici. |
| `collective` | Misura detector stateful/collective e correlazione locale. |
| `kernel-filter-uid` | Misura gli stessi eventi raw con filtro UID kernel-side. |

## Suite automatica multi-scenario

La suite completa elimina la necessita' di coordinare manualmente due
terminali:

```bash
DURATION_SECONDS=120 \
WARMUP_SECONDS=10 \
CPU_THRESHOLD=5.0 \
make benchmark-suite
```

`scripts/benchmark_suite.sh` esegue in sequenza i profili `raw`, `point`,
`collective` e `kernel-filter-uid`. Per ogni profilo:

1. avvia una nuova istanza del tool;
2. individua il PID `project` discendente dal processo appena creato;
3. avvia `scripts/benchmark_workload.sh`, che genera esecuzioni, accessi a
   `/etc/passwd` e transizioni privilegiate a frequenza controllata;
4. passa quel PID esplicitamente a `benchmark_userspace.sh`;
5. termina l'istanza con un'attesa limitata e conserva workload log e misure;
6. marca il profilo `FAIL` quando `avg_cpu` supera `CPU_THRESHOLD`.

I risultati vengono salvati per default in:

```text
tmp/benchmarks/<timestamp>/
  raw.runtime.log
  raw.benchmark.log
  raw.workload.log
  ...
  summary.tsv
```

Per default lo stream degli eventi viene inviato a `/dev/null`: il tool
continua a decodificare, formattare e scrivere gli eventi, ma la suite evita di
salvare file molto grandi e di includere il costo del filesystem nel confronto.
`runtime.log` conserva soltanto lo standard error. Per analizzare anche l'output
completo si puo' impostare `CAPTURE_RUNTIME_OUTPUT=1`.

La chiusura prova prima `SIGINT`, poi `SIGTERM` e infine, soltanto per il
processo figlio creato dal benchmark, `SIGKILL`. Ogni fase ha un timeout
configurabile con `SHUTDOWN_TIMEOUT_SECONDS` (default: 5 secondi), quindi un
profilo non puo' bloccare indefinitamente quelli successivi.

La media CPU e' il criterio di accettazione richiesto. `p95_cpu`, `peak_cpu`,
RSS e numero di thread restano metriche diagnostiche e non fanno fallire da
sole la suite. Per limitare l'esecuzione ad alcuni scenari:

```bash
PROFILES="point collective" make benchmark-suite
```

`benchmark_userspace.sh` accetta ora `TARGET_PID`. Questo evita che `pgrep`
selezioni per errore un'altra istanza del tool gia' in esecuzione sulla VM.

## Risultati controllati del 24 luglio 2026

La suite completa in `tmp/benchmarks/20260724T070129Z` ha usato:

- durata `120s`;
- warm-up `10s`;
- soglia sulla CPU media userspace `5%`;
- stesso workload process/file/privilege per ogni profilo;
- output eventi su `/dev/null`;
- PID esplicito del processo creato dalla suite.

| Profilo | CPU media | P95 | Picco | RSS massimo | Thread massimi | Esito |
|---|---:|---:|---:|---:|---:|---|
| `raw` | `4.36%` | `7.14%` | `9.25%` | `46100 KiB` | 8 | PASS |
| `point` | `0.33%` | `1.79%` | `1.80%` | `40724 KiB` | 7 | PASS |
| `collective` | `3.52%` | `5.19%` | `8.49%` | `44920 KiB` | 8 | PASS |
| `kernel-filter-uid` | `2.92%` | `4.32%` | `5.15%` | `46560 KiB` | 7 | PASS |

Il confronto diretto `raw`/`kernel-filter-uid` mostra una riduzione indicativa
della CPU userspace di circa il 33%. La run dimostra che questi quattro profili
specifici rispettano la soglia media; non dimostra che il tool resti sempre
sotto il 5% con qualsiasi combinazione di eventi.

Il profilo `all-events` abilita tutti gli eventi pubblici e le dipendenze
interne. Una run successiva ha mantenuto circa `77-90%` di un core dopo il
warm-up ed e' stata interrotta prima dei 120 secondi. RSS e thread erano
stabili, quindi il primo sospetto e' il volume di record generato dagli hook ad
alta frequenza, non una crescita incontrollata di goroutine o memoria.

`all-events` va trattato come stress test. Per eseguirlo in modo automatico:

```bash
DURATION_SECONDS=120 \
WARMUP_SECONDS=10 \
CPU_THRESHOLD=5.0 \
make benchmark-all-events
```

I prossimi benchmark devono separare famiglie process, file, credential,
kernel e networking e aggiungere un conteggio del rate per evento. Il criterio
operativo `<5%` resta associato a profili policy-driven realistici.

Esempio:

```bash
make benchmark-profile PROFILE=collective
```

Per scegliere l'UID nel profilo kernel-side:

```bash
KERNEL_FILTER_UID=1000 make benchmark-profile PROFILE=kernel-filter-uid
```

Il benchmark va poi lanciato da un secondo terminale.

3. In alternativa, avviare manualmente il tool con un profilo preciso,
preferibilmente policy-driven.

Profilo collective detector:

```bash
sudo ./dist/project \
  --policy rules/policies/collective-privilege-exec.yaml \
  --detectors rules/detectors/privilege_exec_chain.yaml \
  --alerts-only \
  --alerts-output table \
  --log-level error
```

Profilo full demo:

```bash
sudo ./dist/project \
  --policy rules/policies/full-demo.yaml \
  --detectors rules/detectors \
  --alerts-only \
  --alerts-output table \
  --log-level error
```

4. Attendere 20-30 secondi per escludere startup, caricamento BTF, verifier e
attach probe.

5. Lanciare il benchmark userspace:

```bash
DURATION_SECONDS=120 WARMUP_SECONDS=10 CPU_THRESHOLD=5.0 make benchmark-userspace
```

6. Eseguire in parallelo un workload ripetibile.

Esempio leggero:

```bash
for i in $(seq 1 20); do sudo -n true 2>/dev/null || true; /usr/bin/id >/dev/null; sleep 1; done
```

Esempio file access:

```bash
for i in $(seq 1 60); do cat /etc/passwd >/dev/null; sleep 1; done
```

7. Ripetere lo stesso workload senza tracer per avere una baseline del sistema.

8. Confrontare:

- `avg_cpu`: deve restare sotto `5.0%` per il profilo considerato;
- `p95_cpu`: indica se la maggior parte dei campioni resta sotto controllo;
- `peak_cpu`: puo' superare brevemente il 5%, ma va riportato;
- `peak_rss_kib`: memoria massima osservata;
- `peak_nlwp`: numero massimo di thread;
- eventuali alert prodotti e volume eventi.

Se `avg_cpu` supera stabilmente il `5%`, mitigare in questo ordine:

1. ridurre gli eventi abilitati tramite policy piu' stretta;
2. usare `--alerts-only` per evitare stampa raw non necessaria;
3. disabilitare temporaneamente detector stateful o ridurre la finestra;
4. evitare eventi ad alto volume come `security_file_open` quando non necessari;
5. valutare kernel-side filtering solo dopo aver isolato il collo di bottiglia.

## Benchmark della correlazione generalizzata

Il 29 luglio 2026 e' stato eseguito il profilo `collective` dopo
l'introduzione delle strategie di correlazione `process`, `process_tree`,
`resource`, `cgroup` e delle chiavi composite:

```bash
DURATION_SECONDS=120 \
WARMUP_SECONDS=10 \
CPU_THRESHOLD=5.0 \
PROFILES=collective \
make benchmark-suite
```

Risultato:

| Profilo | CPU media | P95 | Picco | RSS massimo | Thread massimi | Esito |
|---|---:|---:|---:|---:|---:|---|
| `collective` | `3.40%` | `4.53%` | `5.64%` | `58772 KiB` | 7 | PASS |

La media e il P95 sono inferiori al target del 5% di un core. Un campione ha
raggiunto il `5.64%`, quindi il risultato conferma il target medio per questa
run e questo profilo, non un limite rigido applicabile a ogni istante o carico.
I risultati completi sono stati prodotti in
`demo_project/tmp/benchmarks/20260729T020649Z`.

## Stato del benchmark automatizzato

Il target `<5%` e' ora verificabile in modo ripetibile per la parte userspace
con `scripts/benchmark_userspace.sh`.

Resta da automatizzare la componente kernel-side eBPF con `bpftool`, usando due
snapshot di `run_time_ns` e `run_cnt`. Questa parte richiede privilegi e kernel
stats abilitate, quindi va mantenuta come fase separata del benchmark.
