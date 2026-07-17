# 2026-07-16 - Primo filtro UID kernel-side

## Contesto

Dopo l'audit del registry eventi, il passo successivo lato eBPF e' ridurre il
rumore prima che gli eventi arrivino in userspace. Il benchmark userspace ha
mostrato che alcuni profili possono stare sopra il target del 5% di un core.

La prima ottimizzazione scelta e' un filtro UID minimale nel kernel.

## Implementazione

Sono stati aggiunti due campi in `config_entry_t`:

```text
kernel_uid_filter_enabled
kernel_uid_filter
```

Il runtime Go aggiorna `config_map` dopo `BPFLoadObject` e prima dell'attach dei
programmi. Per evitare di duplicare tutta la struct C lato Go, modifica solo i
byte relativi ai due campi del filtro e preserva il resto della configurazione.

Il controllo viene applicato in `init_program_data()` subito dopo il lookup di
`config_map`:

```text
config_map lookup
  -> uid filter
  -> context/process/task init
  -> arg capture
  -> perf submit
```

Se il filtro e' abilitato e l'UID corrente e' diverso da quello configurato,
l'hook ritorna senza costruire l'evento.

## CLI

Esempio:

```bash
sudo ./dist/project \
  --events security_file_open,sched_process_exec \
  --kernel-filter-uid-enabled \
  --kernel-filter-uid 1000 \
  --output table \
  --log-level error
```

Per filtrare root:

```bash
sudo ./dist/project \
  --events sched_process_exec \
  --kernel-filter-uid-enabled \
  --kernel-filter-uid 0 \
  --output table \
  --log-level error
```

## Verifica

Sono stati eseguiti:

```bash
GOCACHE=/tmp/go-build go test ./pkg/config
GOCACHE=/tmp/go-build make test-registry
GOCACHE=/tmp/go-build make build
```

La build eBPF e il binario Go sono stati generati correttamente.

## Primo benchmark esplorativo

Dopo la modifica sono stati confrontati tre profili manuali:

1. policy/detector collective con `--alerts-only`;
2. eventi rumorosi senza filtro UID;
3. stessi eventi rumorosi con filtro UID kernel-side.

I risultati osservati sono:

| Caso | CPU osservata | RSS osservato | Nota |
|---|---:|---:|---|
| Policy/detector collective | spesso `6.8% - 8.5%` | circa `82 MiB` | sopra target |
| Eventi senza filtro UID | spesso `2% - 3.7%` | circa `81 MiB` | quasi sempre sotto target |
| Eventi con filtro UID | spesso `2% - 3.7%`, picco `4.59%` | circa `48 MiB` | sotto target nei campioni |

Il dato non e' ancora benchmark finale perche' le run sono state interrotte
prima dei 120 secondi. Tuttavia indica che il filtro UID riduce traffico e RSS,
mentre il profilo piu' costoso resta quello con policy e detector collective
attivi.

Per rendere le prossime misure piu' difendibili, lo script
`scripts/benchmark_userspace.sh` ora distingue warm-up e misurazione reale. La
summary include `avg_cpu`, `p95_cpu`, `peak_cpu`, RSS massimo e thread massimi.
I campioni di warm-up restano visibili ma non entrano nella media e nella
valutazione della soglia.

E' stato aggiunto anche `scripts/benchmark_profiles.sh`, richiamabile dal
Makefile con:

```bash
make benchmark-profile PROFILE=raw
make benchmark-profile PROFILE=point
make benchmark-profile PROFILE=collective
KERNEL_FILTER_UID=1000 make benchmark-profile PROFILE=kernel-filter-uid
```

Questa separazione rende piu' chiaro il metodo di misura:

- un terminale avvia il profilo del tool;
- un secondo terminale misura CPU/RSS/thread con `make benchmark-userspace`;
- lo stesso workload puo' essere ripetuto su profili diversi.

E' stato poi verificato il detector dispatcher. Il routing per evento era gia'
presente, quindi non e' stato introdotto un nuovo modello. La modifica ha
consolidato la hot path:

- meno allocazioni quando un dispatch non produce alert o errori;
- nuove metriche `DetectorMatched`, `DetectorInvoked` e `DetectorSkipped`;
- test che verificano esplicitamente che i detector non interessati non vengano
  invocati.

Il benchmark ripetuto sul profilo `collective` resta sopra il target, con
campioni `measure` spesso tra circa `6%` e `7.7%` CPU e RSS intorno a `84 MiB`.
Questo indica che la causa principale non e' il routing del dispatcher, ma la
selettivita' di policy/detector, il volume di eventi e il costo di alert/output.

E' stato quindi preparato un file di discussione per chiarire i dubbi
implementativi prima di ulteriori modifiche:

```text
documentation/next-steps/open-implementation-questions-2026-07-16.md
```

Nota operativa: se si filtra UID `1000` e si usano detector orientati a eventi
root, come `root-exec`, e' normale non vedere alert. Il filtro scarta proprio
gli eventi UID `0` necessari a quei detector.

## Limiti

Il filtro e' intenzionalmente semplice:

- un solo UID alla volta;
- nessun range;
- nessuna lista UID;
- nessun filtro `comm`;
- nessuna semantica policy YAML nel kernel.

Questo evita di spostare prematuramente il policy engine in eBPF. L'obiettivo e'
misurare se un filtro economico riduce abbastanza traffico e CPU.
