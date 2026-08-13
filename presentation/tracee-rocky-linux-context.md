# Contesto progettuale: Tracee e kernel Rocky Linux

## Perche' Tracee e' il riferimento

Il tool sviluppato prende ispirazione da Tracee, un progetto open source basato
su eBPF per runtime security e forensics su Linux.

Tracee e' utile come riferimento per tre motivi principali:

- mostra come organizzare molti hook kernel in un unico runtime;
- propone un modello di eventi normalizzati e consumabili in userspace;
- include concetti di policy, detector e mapping verso threat model come MITRE
  ATT&CK.

Il nostro progetto non e' una replica completa di Tracee. L'obiettivo e'
costruire un tool piu' mirato, utile per la tesi e per il contesto aziendale,
con una superficie piu' controllabile.

## Differenza principale rispetto a Tracee

Tracee e' progettato per essere general purpose e supportare molti kernel,
molti eventi e molti scenari di deployment.

Il nostro tool invece e' progettato intorno a un target preciso:

```text
Rocky Linux / Red Hat Enterprise Linux 8
Kernel 4.18.0-553.137.1.el8_10.x86_64
```

Questa scelta influenza l'implementazione:

- alcuni helper eBPF moderni non sono disponibili;
- il verifier del kernel 4.18 e' piu' restrittivo rispetto a kernel recenti;
- alcune sezioni eBPF o alcuni hook possono avere nomi o disponibilita'
  diverse;
- il perf buffer e' stato preferito come trasporto principale per maggiore
  compatibilita' con il kernel target;
- alcune implementazioni sono state semplificate rispetto a Tracee per ridurre
  rischio di rigetto da parte del verifier.

## Messaggio da comunicare nelle slide

Il valore del progetto non e' "rifare Tracee", ma costruire una pipeline di
runtime monitoring eBPF:

- piu' piccola;
- piu' controllabile;
- adattata al kernel Rocky Linux 4.18;
- focalizzata sugli eventi utili alla tesi;
- estendibile verso policy e detector.

Questa impostazione permette di mostrare maturita' tecnica senza sostenere che
il tool abbia gia' la stessa copertura o robustezza di un progetto industriale
maturo come Tracee.

