# Arricchimento di `sched_process_exec`

## Obiettivo

L'evento `sched_process_exec` e' stato esteso per descrivere l'immagine
effettivamente installata dopo una exec riuscita. I nuovi argomenti restano
parte dello stesso evento pubblico e non introducono un adapter alternativo.

## Payload aggiornato

Oltre a `fileName`, l'evento espone:

- `pathname`, `dev`, `inode` e `ctime` per il file risolto;
- `argc` e `argv` per gli argomenti della nuova immagine;
- `envc` e `envp` per l'ambiente installato;
- `real_uid` per distinguere il real UID dall'effective UID del contesto;
- `process_unique_id`, `thread_unique_id` e `parent_unique_id` come hash locali
  di identificatore host e start time.

`argv` ed `envp` vengono letti dagli intervalli del nuovo `mm_struct` e usano
il formato compatto NUL-delimited gia' supportato dal decoder. Ciascun array e'
limitato a 8 KiB e a 1.024 valori rappresentati, mentre `argc` ed `envc`
mantengono il conteggio originale.

## Verifica

La compilazione eBPF e i test di registry, decoder e detector sono stati
completati con successo. Un test manuale sul kernel Rocky Linux target ha
eseguito `/bin/sleep 2` con ambiente controllato e ha confermato:

- distinzione tra `fileName=/bin/sleep` e `pathname=/usr/bin/sleep`;
- identita' device--inode valorizzata;
- `argc=2` con `argv=["/bin/sleep", "2"]`;
- `envc=2` con le due variabili di test;
- real UID e tre identificatori locali valorizzati;
- assenza di errori runtime durante la prova.

## Considerazione di sicurezza

Una prova iniziale ha mostrato che l'ambiente di processi non appartenenti al
test puo' includere token di sessione e dettagli SSH. `envp` viene attualmente
raccolto senza redazione quando `sched_process_exec` e' selezionato. Gli output
devono quindi essere trattati come sensibili. Rendere questa raccolta opt-in o
applicare una strategia di redazione resta una modifica raccomandata.
