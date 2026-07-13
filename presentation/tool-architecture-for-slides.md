# Architettura del tool per le slide

## Obiettivo macro

Il tool e' un sistema di runtime monitoring basato su eBPF.

Il suo compito e':

1. osservare eventi kernel rilevanti;
2. costruire un evento strutturato lato eBPF;
3. trasportare l'evento verso userspace;
4. decodificarlo in Go;
5. applicare filtri, policy e detector;
6. stampare eventi o alert in formato leggibile.

## Pipeline end-to-end

La pipeline puo' essere descritta cosi':

```text
CLI configuration
  -> load eBPF object
  -> attach selected probes
  -> kernel hook triggered
  -> event built in eBPF
  -> perf buffer transport
  -> userspace Go runtime
  -> decoder
  -> filters / policies / detectors
  -> event output or alert output
```

## Componenti principali

### Runtime Go

Il runtime Go:

- legge la configurazione da CLI;
- carica l'oggetto eBPF compilato;
- seleziona gli eventi richiesti;
- attacca i programmi eBPF agli hook corrispondenti;
- apre i buffer di comunicazione kernel-userspace;
- riceve record raw;
- passa i record al decoder;
- invia gli eventi decodificati a output e detector.

### Programmi eBPF

I programmi eBPF sono eseguiti nel kernel quando viene attivato un hook.

Esempi di hook:

- tracepoint di lifecycle processo;
- tracepoint syscall enter/exit;
- kprobe;
- kretprobe;
- LSM/security hook disponibili sul kernel target.

Il programma eBPF raccoglie informazioni essenziali come:

- PID/TID;
- UID;
- command name;
- nome evento;
- argomenti specifici dell'evento.

### Perf buffer

Il perf buffer e' il canale principale usato per trasportare gli eventi dal
kernel allo userspace.

La scelta e' pragmatica:

- e' compatibile con il kernel Rocky Linux 4.18;
- e' gia' supportato dal runtime libbpfgo;
- permette di mantenere un comportamento coerente tra hook diversi.

Le helper per ring buffer sono mantenute nel progetto per flessibilita', ma la
direzione attuale e' usare perf buffer come percorso standard.

### Decoder userspace

Il decoder trasforma byte raw in eventi leggibili.

Senza decoder, il runtime vedrebbe solo dati binari. Il decoder invece produce:

- nome evento;
- contesto processo;
- lista argomenti;
- tipi degli argomenti;
- valori normalizzati.

### Output

Il tool supporta due output principali:

- `table`: compatto, utile per demo e debug manuale;
- `json`: piu' adatto a integrazioni e consumo automatico.

Gli alert generati dai detector hanno un output separato, controllabile tramite
flag dedicate.

## Policy e detector nella pipeline

Il layer policy/detector si trova dopo il decoding.

Questo significa che:

- gli eventi vengono prima raccolti e normalizzati;
- poi vengono valutati da regole e detector userspace;
- se una condizione e' soddisfatta, viene prodotto un alert.

In questa fase il layer e' pensato per anomaly puntuali e prime forme di
correlazione collettiva locale. Le anomalie contestuali su tutto il cluster
Kubernetes sono considerate responsabilita' di un livello centralizzato esterno.

