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

### Decoder userspace

Il decoder userspace rappresenta il punto di incontro tra il formato binario
prodotto dai programmi eBPF e una rappresentazione Go utilizzabile dal resto
della pipeline. Il progetto usa un decoder custom per mantenere controllo sul
contratto locale tra `types.h`, `pkg/events/spec.go` e output. Il context
attuale e' di 136 byte e include anche `policies_version` e
`matched_policies`, per preparare il passaggio verso policy e detector.

### Output layer

La separazione del layer output consente di distinguere il problema del
trasporto eventi dal problema della presentazione. Il runtime eBPF produce
eventi decodificati, mentre `pkg/output` decide se stamparli come JSON
machine-readable o come righe table per debug manuale.

Il passaggio da JSON raw a JSON normalizzato rende l'evento piu' utile per
analisi successive: campi C-style come `comm` e `uts_name` diventano stringhe e
valori numerici come le Linux capabilities possono essere arricchiti con label
simboliche.

## Dettagli da approfondire

- Differenza tra sicurezza logica del codice C e dimostrabilita' per il verifier.
- Trade-off tra formato eventi compatto e formato verifier-friendly.
- Evoluzione da ring buffer only a reader duale ring buffer/perf buffer.
- Evoluzione da `cilium/ebpf` a `libbpfgo`.
- Differenza tra decoder completo Tracee e decoder MVP custom del progetto.
- Ruolo dello schema statico eventi in `protocol.go`.

## Collegamenti

- [Timeline](../timeline.md)
- [Mappa capitoli](thesis-outline.md)
