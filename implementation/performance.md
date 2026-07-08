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

1. Avviare il tool con un set preciso di eventi.
2. Attendere la fine della fase di startup.
3. Raccogliere uno snapshot userspace ed eBPF.
4. Eseguire per almeno 60 secondi un workload ripetibile.
5. Raccogliere il secondo snapshot.
6. Ripetere lo stesso workload senza tracer.
7. Confrontare media, picco, RSS e tempo totale del workload.
