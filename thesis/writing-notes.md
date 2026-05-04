# Spunti per testo tesi

## Frasi e idee da riutilizzare

### Motivazione

Il monitoraggio runtime a livello kernel permette di osservare eventi di sicurezza con maggiore completezza rispetto a soluzioni puramente userspace, riducendo il rischio che processi malevoli eludano la raccolta dati.

### Perche' eBPF

eBPF consente di eseguire programmi verificati nel kernel senza modificare o ricompilare il kernel stesso. Questo lo rende adatto a strumenti di osservabilita' e sicurezza runtime con overhead contenuto.

### Perche' Tracee come riferimento

Tracee rappresenta un riferimento production-grade per runtime security basata su eBPF. Il progetto di tesi ne riprende alcuni pattern fondamentali, semplificandoli per ottenere un'architettura piu' piccola e controllabile.

### Debug verifier

Una parte rilevante dell'implementazione e' stata guidata dal verifier eBPF. Alcune strutture dati logicamente corrette sono state adattate per rendere espliciti al verifier i limiti di memoria e le dimensioni delle copie.

## Dettagli da approfondire

- Differenza tra sicurezza logica del codice C e dimostrabilita' per il verifier.
- Trade-off tra formato eventi compatto e formato verifier-friendly.
- Scelta di ring buffer invece di perf event array.
- Scelta di `cilium/ebpf` rispetto a `libbpfgo`.

## Collegamenti

- [Timeline](../timeline.md)
- [Mappa capitoli](thesis-outline.md)
