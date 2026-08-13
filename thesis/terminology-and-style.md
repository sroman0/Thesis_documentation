# Terminologia e regole editoriali

## Lingua e voce

- Il testo finale della tesi e' in inglese.
- Usare una voce accademica chiara e impersonale, evitando formule promozionali.
- Preferire frasi verificabili a espressioni come "innovative", "complete" o
  "production-ready" senza evidenza.
- Usare il presente per architettura e comportamento corrente, il passato per
  attivita' sperimentali concluse e il futuro solo per lavoro non implementato.

## Nomi canonici

| Concetto | Forma da usare |
|---|---|
| Tool della tesi | `the proposed tool`, `the prototype`, `the monitoring agent` o `the proposed system` |
| Tecnologia | `eBPF`, non `EBPF` o `Ebpf` |
| Sistema operativo target | `Rocky Linux 8.10` |
| Kernel target | `4.18.0-553.137.1.el8_10.x86_64` |
| Spazio kernel | `kernel space` |
| Spazio utente | `user space` come sostantivo, `user-space` come aggettivo |
| Runtime security | `runtime security monitoring` |
| Regole ATT&CK | `MITRE ATT&CK tactics and techniques` |
| Evento individuale | `point anomaly` |
| Sequenza correlata | `collective anomaly` |
| Contesto non disponibile localmente | `contextual anomaly` |
| Canale operativo corrente | `perf buffer` |
| Canale alternativo discusso nel background | `BPF ring buffer` |

## Distinzioni da preservare

### Event, policy, detector e alert

- Un `event` e' un fatto osservato e decodificato.
- Una `policy` seleziona o filtra gli eventi rilevanti per uno scenario.
- Un `detector` valuta uno o piu' eventi e puo' mantenere stato locale breve.
- Un `alert` e' il risultato prodotto da un detector quando una condizione e'
  soddisfatta.

### Hook ed evento logico

Un evento userspace puo' essere alimentato da piu' hook o syscall. Non usare
`hook` ed `event` come sinonimi.

### Implementazione e novelty

- `Implemented contribution`: componente o comportamento costruito nel
  progetto e verificabile nel codice.
- `Research novelty`: elemento originale rispetto allo stato dell'arte,
  supportato da confronto bibliografico e valutazione.

Fino al completamento del related work, le novelty restano candidate.

### Tracee come riferimento

Descrivere Tracee come riferimento architetturale production-grade. Evitare
formulazioni che suggeriscano una copia completa o una equivalenza funzionale.
Quando il progetto devia da Tracee, specificare se la scelta deriva da scope,
compatibilita' Rocky Linux o controllo dell'overhead.

## Citazioni e prove

- Le nozioni generali su eBPF, kernel e MITRE richiedono fonti esterne.
- Le descrizioni del tool proposto devono rimandare al codice o alla documentazione
  tecnica, senza usare il repository come unica prova di affermazioni generali.
- Preferire documentazione kernel, paper scientifici, MITRE ATT&CK e
  documentazione ufficiale dei tool.
- Non inventare entry BibTeX. Ogni riferimento deve essere verificabile.

## Vincoli di scope gia' concordati

- Il focus principale e' process and security monitoring.
- I detector locali producono point e collective anomalies.
- Le contextual anomalies che richiedono visione globale del cluster restano
  responsabilita' di un eventuale livello centralizzato.
- L'obiettivo prestazionale corrente e' restare sotto il 5% di un core nel
  profilo operativo da concordare; non presentarlo come risultato raggiunto
  finche' il benchmark definitivo non lo conferma.
- L'abstract non viene aggiornato prima del completamento della tesi.
- Il nome interno `Vesuvius` non deve comparire nel testo della tesi. Non esiste
  un nome accademico ufficiale del tool.
- Il Capitolo 1 usa solo `section`, senza `subsection`, per mantenere una
  introduzione narrativa e proporzionata.
- Il runtime corrente usa il perf buffer. Il ring buffer e' materiale di
  background o lavoro futuro e non deve essere descritto come implementato.
