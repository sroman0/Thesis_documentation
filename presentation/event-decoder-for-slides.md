# Event model, decoder e normalizzazione

Questo documento descrive come un evento kernel diventa un evento leggibile in
userspace.

## Perche' serve un event model

Gli eventi prodotti dal kernel non sono immediatamente leggibili.

Il programma eBPF scrive un record binario composto da:

- contesto comune dell'evento;
- numero di argomenti;
- payload degli argomenti;
- eventuali stringhe o array.

Il runtime Go deve sapere come interpretare quei byte. Per questo esiste un
event model condiviso tra lato eBPF e lato userspace.

## Contesto comune

Ogni evento contiene informazioni comuni:

- ID evento;
- timestamp;
- PID;
- TID;
- UID;
- command name (`comm`);
- informazioni di processo utili al runtime.

Questo permette di stampare righe del tipo:

```text
event=security_file_open pid=... tid=... uid=... comm=cat args=...
```

## Registry eventi

Il registry eventi associa:

- ID numerico;
- nome evento;
- schema degli argomenti;
- tipo di ogni argomento.

Esempio concettuale:

```text
ID 732 -> security_file_open
args:
  pathname: string
  flags: int
  dev: uint
  inode: ulong
  ctime: ulong
  syscall_pathname: string
```

Questa mappatura e' fondamentale: se l'ID scritto da eBPF non corrisponde allo
schema Go, il decoder interpreta male il payload o segnala eventi sconosciuti.

## Argomenti tipizzati

Il decoder supporta diversi tipi:

- interi signed/unsigned;
- long/ulong;
- booleani;
- puntatori;
- stringhe;
- array di stringhe.

Il supporto agli array di stringhe e' importante per eventi come `execve` ed
`execveat`, dove serve rappresentare `argv`.

## Normalizzazione

La normalizzazione rende gli eventi piu' leggibili.

Esempi:

- capability numeriche possono essere mappate a nomi come `CAP_SYS_ADMIN`;
- return value negativi possono essere convertiti in errori leggibili come
  `ENOENT(-2): no such file or directory`;
- path e stringhe C-style vengono puliti dai terminatori nulli;
- gli array di stringhe vengono stampati in modo leggibile.

## Output eventi

Il tool supporta due formati:

- `table`, pensato per debug e demo;
- `json`, pensato per integrazioni automatiche.

Il formato `table` e' compatto e umano:

```text
event=execve pid=123 uid=1000 comm=bash args=pathname=/usr/bin/id,argv=["id"]
```

Il formato `json` e' piu' adatto a pipeline esterne.

## Eventi pubblici e dipendenze interne

Il registry distingue tra:

- eventi pubblici, selezionabili dall'utente;
- eventi interni, usati come dipendenze tecniche.

Questa distinzione serve per gli eventi costruiti con modello enter/exit.

Esempio:

- l'utente vuole osservare `open`;
- internamente il tool deve attaccare sia `sys_enter_openat` sia
  `sys_exit_openat`;
- l'evento pubblico resta `open`;
- gli eventi enter/exit sono dipendenze, non output finali.

Questo evita confusione nella CLI e impedisce che l'utente debba conoscere tutti
i dettagli tecnici necessari per costruire un evento ad alto livello.

## Problema risolto: mismatch degli ID

Durante lo sviluppo e' emerso un problema: alcuni eventi venivano ricevuti ma
non stampati correttamente per mismatch tra ID lato eBPF e registry lato Go.

Il sintomo era:

```text
unknown event id=...
```

Oppure eventi filtrati perche' decodificati con nome sbagliato.

La soluzione e' stata rendere il registry piu' coerente, separando:

- eventi pubblici;
- dipendenze interne;
- schema di decoding.

Questo punto e' utile da citare nelle slide come esempio di maturazione del
tool: non e' solo cattura eventi, ma anche correttezza del contratto binario tra
kernel e userspace.

