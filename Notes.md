- Per la comunicazione tra userspace e kernel space, conviene utilizzare un
  ring buffer oppure un perf buffer? Risolto definitivamente il 24/07/2026: il
  perf buffer e' l'unico transport operativo; ring buffer, reader e helper non
  usati sono stati rimossi per ridurre complessita' e mantenere compatibilita'
  con Rocky Linux 4.18.
- Chiedere se è necessario implementare un file .yaml su .github per le github actions, cosi che ad ogni push viene effettuato un test sul codice.
- Idea: preparare dei template ai customers per filtrare gli eventi in base ad una macroarea. Template per il processing, template per il networking, template per security, template per attacchi specifici...
- Idea: provare ad implementare un sistema di monitoring non solo per eventi singoli, ma anche per pattern di eventi.
- Idea: implementare profili/policy di eventi, per esempio `process`, `filesystem`, `memory`, `kernel-hardening`, `network`, invece di lanciare sempre tutti gli hook.
- Idea: introdurre kernel-side filtering minimo per UID, PID, namespace, cgroup/container e `comm`, cosi' da ridurre rumore e costo prima del perf buffer.
- controllare la prestazione del tool, idealmente sotto il 5% di un core in steady-state con profili realistici, non necessariamente con tutti gli hook abilitati insieme.

## Nota rollback: flag detector esplicita

Configurazione CLI inizialmente prevista per policy/detector:

```bash
--policy policy.yaml
--detectors detectors/
--enable-detectors
--alerts
--alerts-output json
```

Questa versione separava il path dei detector (`--detectors`) dall'interruttore
che abilita il motore (`--enable-detectors`). La scelta corrente e' piu'
semplice per l'utente: se `--detectors` riceve almeno un file o una directory,
il tool abilita automaticamente i detector in fase di normalizzazione della
configurazione.

Se in futuro serve distinguere "carica i detector" da "esegui i detector", si
puo' ripristinare `--enable-detectors` come flag esplicita.
