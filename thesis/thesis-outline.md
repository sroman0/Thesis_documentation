# Mappa capitoli tesi

Questa e' una bozza viva della struttura della tesi. Va aggiornata man mano che il progetto matura.
## 1. Introduzione

- Motivazione: runtime security e bisogno di visibilita' kernel-level.
- Limiti di monitoraggio puramente userspace.
- Obiettivo della tesi.
- Collaborazione con Rakuten.

## 2. Background

- Linux kernel observability.
- eBPF: programmi, mappe, verifier, helper.
- CO-RE e BTF.
- Ring buffer.
- Tracepoint, raw tracepoint, kprobe, LSM hook.
## 3. Stato dell'arte

- Tracee come riferimento principale.
- Altri strumenti runtime security.
- Differenze tra tool production-grade e MVP di ricerca.
## 4. Requisiti e ambiente target

- Rocky Linux 8.10.
- Kernel RHEL-compatible con backport.
- Requisiti funzionali.
- Requisiti non funzionali.

## 5. Design del sistema

- Architettura generale.
- Kernel space component.
- Userspace component.
- Formato eventi.
- Scelte architetturali rispetto a Tracee.

## 6. Implementazione

- Hook process lifecycle.
- Hook security-related.
- Mappe eBPF.
- Loader Go.
- Attach programmi.
- Ring buffer.
- Debug verifier.

## 7. Output e detection

- Decoder eventi.
- Output JSON.
- Regole di detection.
- Mapping MITRE ATT&CK.

## 8. Valutazione

- Correttezza funzionale.
- Overhead.
- Stabilita' su Rocky Linux.
- Confronto qualitativo con Tracee.

## 9. Conclusioni

- Risultati raggiunti.
- Limiti.
- Lavori futuri.

## Collegamenti

- [Timeline](../timeline.md)
- [Spunti scrittura](writing-notes.md)
