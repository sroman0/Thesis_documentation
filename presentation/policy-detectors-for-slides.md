# Policy e detector layer

Questo documento riassume il layer policy/detector in forma adatta alle slide.
La parte policy/detector va presentata come componente condivisa del tool, non
come parte esclusiva di Simone.

## Definizioni

### Policy

Una policy definisce quali eventi e quali detector devono essere abilitati per
uno scenario.

Nel tool attuale una policy puo':

- selezionare eventi da osservare;
- definire un intento, per esempio threat hunting;
- abilitare detector associati agli eventi richiesti.

La policy risponde alla domanda:

```text
Cosa vogliamo osservare in questa run?
```

### Detector

Un detector descrive una condizione sospetta su uno o piu' eventi.

Nel tool attuale i detector YAML lavorano principalmente su singolo evento.

Un detector risponde alla domanda:

```text
Quando un evento diventa rilevante dal punto di vista security?
```

### Alert

Un alert e' l'output prodotto da un detector quando la condizione viene
soddisfatta.

L'alert contiene:

- titolo;
- severita';
- ID detector;
- evento sorgente;
- contesto processo;
- argomenti principali;
- metadata MITRE ATT&CK, quando definiti.

## Workflow

Il workflow attuale e':

```text
policy YAML
  -> policy loader
  -> selected events
  -> detector YAML loader
  -> detector registry
  -> event runtime
  -> detector evaluation
  -> alert output
```

## Esempi di detector demo

| Detector | Evento sorgente | Scopo | Severita' |
| --- | --- | --- | --- |
| `root-exec` | `sched_process_exec` | Alert quando un processo UID 0 esegue un binario. | medium |
| `sensitive-file-open` | `security_file_open` | Alert su apertura di file critici di identita' o access control. | medium |
| `privileged-uid-change` | `security_task_fix_setuid` | Alert quando un processo cambia effective UID a root. | medium |
| `privilege-exec-chain` | `security_task_fix_setuid` + `sched_process_exec` | Alert collective quando un cambio privilegi e' seguito da exec root. | high |
| `temp-script-write-exec` | `security_file_permission` + `security_bprm_check` | Alert collective quando lo stesso `dev:inode` temporaneo viene scritto e poi preparato per l'esecuzione. | high |
| `kernel-module-activity` | `do_init_module` | Alert su inizializzazione riuscita di un modulo kernel. | high |

## Mapping MITRE ATT&CK

I detector includono metadata MITRE ATT&CK.

Esempi:

- `root-exec` -> Execution;
- `sensitive-file-open` -> Discovery;
- `privileged-uid-change` -> Privilege Escalation;
- `privilege-exec-chain` -> Privilege Escalation / Execution;
- `kernel-module-activity` -> Persistence / Defense Evasion.

Il mapping MITRE serve a rendere gli alert piu' difendibili e comprensibili in
un contesto professionale.

## Alert-only mode

Il tool supporta una modalita' `alerts-only`.

Questa modalita':

- continua a raccogliere e valutare eventi raw;
- non stampa tutti gli eventi raw;
- stampa solo gli alert generati dai detector.

Serve per demo e per ridurre rumore durante l'esecuzione.

## Riduzione del rumore

Durante i test, il detector su file sensibili generava troppi alert su file
legittimi come:

- `/etc/hosts`;
- `/etc/ld.so.cache`;
- file letti dal tool stesso durante attach dei probe.

La regola e' stata resa piu' stringente:

- match esatto su una lista di file realmente critici;
- operatore `in`;
- esclusione di path troppo generici.

La lista demo attuale include file come:

- `/etc/passwd`;
- `/etc/shadow`;
- `/etc/sudoers`;
- `/etc/ssh/sshd_config`;
- `/root/.ssh/authorized_keys`.

In aggiunta, il detector engine applica un dedup temporale breve:

- finestra default: 5 secondi;
- chiave basata su detector, evento sorgente, pid, uid, comm e argomenti;
- alert identici nello stesso intervallo vengono stampati una sola volta.

Questo non elimina eventi dalla pipeline: riduce solo la ripetizione degli
alert in output.

## Limiti attuali

Il layer detector e' funzionante, ma ancora iniziale.

Limiti:

- i detector YAML supportano point anomaly e una prima forma di collective
  anomaly ordinata;
- il dedup alert e' presente, ma per ora usa una finestra fissa di 5 secondi;
- la correlazione collective supporta `process`, `process_tree`, `resource`,
  `cgroup` e composizioni locali;
- `process_tree` non mantiene un grafo globale persistente;
- `user_session` e' differita perche' UID non identifica una sessione;
- non ci sono ancora baseline comportamentali o contesto Kubernetes globale;
- le anomalie contextual su intero cluster Kubernetes sono considerate fuori
  dal singolo agent e dovrebbero essere gestite da un componente centralizzato.

## Prossimi step

I prossimi step piu' importanti sono:

- rendere configurabile il dedup se i test reali lo richiedono;
- operatori YAML piu' espressivi;
- stato process-tree piu' robusto e misurato con benchmark;
- selezione detector/policy e report di copertura basati su MITRE ATT&CK;
- benchmark per verificare overhead;
- migliore separazione tra rumore operativo e alert realmente utili.
