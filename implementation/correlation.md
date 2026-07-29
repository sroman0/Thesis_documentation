# Correlazione locale dei detector collective

## Scopo

Il motore collective correla sequenze ordinate di eventi sul singolo nodo.
Non ricostruisce anomalie contestuali di cluster e non mantiene uno storico
generale delle attivita'.

Due concetti sono indipendenti:

- `window` stabilisce per quanto tempo una sequenza incompleta resta valida;
- `group_by` stabilisce quali eventi possono appartenere alla stessa sequenza.

Una finestra temporale uguale non implica una correlazione: gli eventi devono
anche produrre la stessa chiave.

## Modello evento disponibile

Ogni `bufferdecoder.Event` normalizzato contiene nel context:

| Campo | Uso |
|---|---|
| `HostPid` + `StartTime` | Identita' stabile del processo, protetta dal riuso PID. |
| `HostPpid` + `ParentStartTime` | Identita' del parent immediato. |
| `Uid` | Identita' utente, non identita' di sessione. |
| `CgroupID` | Identita' cgroup locale disponibile su tutti i record normali. |
| `MntID`, `PidID` | Identita' dei namespace mount e PID; non sono ancora strategie `group_by`. |

Gli identificatori di file non sono nel context comune. Gli eventi file
compatibili espongono invece gli argomenti tipizzati `dev` e `inode`. Il parser
consulta `pkg/events/spec.go` e accetta `resource` solo quando ogni step della
sequenza dichiara entrambi.

`comm` e pathname restano utili come condizioni, ma non sono identita' stabili.

## Strategie implementate

### `process`

Chiave:

```text
host_pid:start_time
```

Correla soltanto lo stesso processo. La combinazione impedisce che un PID
riutilizzato completi una sequenza iniziata da un processo precedente.

Per compatibilita', le vecchie forme `pid`, `process.pid`, `context.pid`,
`host_pid`, `host.pid` e `context.host_pid` vengono normalizzate
esplicitamente a `process`.

### `process_tree`

Il primo evento salva lo stato con l'identita' del processo corrente. Gli step
successivi provano:

```text
self   = host_pid:start_time
parent = host_ppid:parent_start_time
```

Sono quindi supportati stesso processo e relazione parent-child immediata. Non
viene mantenuto un grafo persistente e non sono correlati discendenti arbitrari.

### `resource`

Per gli eventi file compatibili:

```text
dev:inode
```

Pathname non partecipa alla chiave e non esiste fallback silenzioso. Se `dev` o
`inode` manca dal record, l'evento non crea e non avanza lo stato. Di
conseguenza:

- stesso pathname con inode diverso non corrisponde;
- stesso inode su device diverso non corrisponde;
- processi diversi possono corrispondere se osservano lo stesso oggetto.

### `cgroup`

Chiave:

```text
event_context.cgroup_id
```

Permette la correlazione tra processi diversi nello stesso cgroup locale. Un
valore zero viene trattato come identita' non disponibile e non crea stato.
Non equivale a un container ID arricchito e non fornisce contesto Kubernetes.

### Chiavi composite

`group_by` e' una lista ordinata:

```yaml
group_by:
  - cgroup
  - resource
```

Tutti i componenti devono corrispondere. Nell'esempio, stesso file in un cgroup
diverso non completa la sequenza, e stesso cgroup con file diverso non la
completa.

## Strategia differita

`user_session` non e' implementata. Il context contiene `Uid`, ma UID identifica
un account e non una sessione di login. Il parser riconosce il nome della
strategia e restituisce un errore esplicito invece di usarlo come alias di UID.

Un follow-up richiederebbe:

1. scegliere una sorgente affidabile, per esempio l'audit session ID;
2. aggiungere il campo a `task_context_t`/`event_context_t` in
   `pkg/ebpf/c/types.h`;
3. popolarlo nel percorso comune in `pkg/ebpf/c/common/context.h`;
4. aggiornare `bufferdecoder.EventContext`, offset e `eventContextSize`;
5. aggiornare test binari, output e compatibilita' degli oggetti embedded.

Questa modifica allargherebbe ogni record evento e aggiungerebbe lavoro al
percorso comune eBPF; deve essere preceduta da una verifica sul kernel Rocky
Linux target e da benchmark.

## Validazione YAML

Il parser:

- rifiuta strategie sconosciute;
- rifiuta componenti duplicati anche quando sono alias della stessa strategia;
- rifiuta `resource` se uno step non espone `dev` e `inode`;
- rifiuta `user_session` finche' manca un identificatore stabile;
- include nome detector e campo `group_by` nel messaggio di errore;
- normalizza la sintassi legacy a `process`.

## Stato e prestazioni

Ogni detector conserva soltanto:

- indice del prossimo step;
- timestamp del primo evento;
- eventi che costituiscono l'evidenza della sequenza.

Restano validi:

- finestra predefinita `2s`;
- finestra massima `5s`;
- pruning periodico `min(window / 2, 1s)`;
- `Flush()` per la pulizia forzata.

La mappa e' inoltre limitata a 4096 sequenze incomplete per detector. Al
raggiungimento del limite viene espulso lo stato piu' vecchio. La scansione
necessaria all'espulsione avviene solo nel percorso di saturazione, non per ogni
evento.

Le strategie vengono compilate durante `NewDetector()`. La hot path non
reinterpreta i nomi YAML e non serializza mappe di argomenti per costruire le
chiavi. Il profilo benchmark collective del 29 luglio 2026 ha misurato CPU
media `3.40%` e P95 `4.53%`; il picco `5.64%` mostra che il target riguarda la
media operativa e non ogni singolo campione.

## Detector dimostrativo

`rules/detectors/temp_script_write_exec.yaml` correla:

```text
security_file_permission(mask=MAY_WRITE, /tmp/*.sh)
  -> security_bprm_check(/tmp/*.sh)
```

La chiave e' `resource`, quindi i due eventi possono provenire da PID diversi
ma devono riferirsi allo stesso `dev:inode`. I filtri sul pathname restringono
il segnale; non definiscono l'identita' della risorsa.

Il mapping `TA0002` / `T1059.004` indica una sequenza coerente con esecuzione
tramite Unix shell. Il detector segnala comportamento osservabile, non prova da
solo che lo script sia malevolo.

Avvio:

```bash
sudo ./dist/project \
  --policy rules/policies/temp-script-write-exec.yaml \
  --detectors rules/detectors/temp_script_write_exec.yaml \
  --alerts-only \
  --alerts-output table \
  --log-level error
```

Esca, in un secondo terminale:

```bash
script="$(mktemp /tmp/vesuvius-resource-XXXXXX.sh)"
printf '#!/bin/sh\nid >/dev/null\n' > "$script"
chmod +x "$script"
"$script"
rm -f "$script"
```

Output atteso: un alert `temp-script-write-exec` con `events=2` e la sequenza
`security_file_permission -> security_bprm_check`.

Il test sul kernel target ha prodotto l'alert previsto. I due eventi avevano
PID differenti ma la stessa coppia `dev:inode`, confermando che il match
cross-process usa l'identita' della risorsa e non il pathname.

Il test e' stato ripetuto con pathname superiori a 120 caratteri dopo la
correzione dell'offset in `save_to_submit_buf()`. Il record
`security_bprm_check` e la successiva correlazione sono rimasti completi anche
oltre 256 byte complessivi di argomenti.

## Rischi residui

- device e inode possono essere riutilizzati nel tempo; la finestra massima di
  cinque secondi riduce, ma non elimina, il rischio;
- filesystem overlay o eventi riferiti a layer diversi possono esporre
  identita' differenti per path apparentemente uguali;
- `process_tree` copre soltanto una relazione parent-child immediata;
- `cgroup` e' locale al nodo e non sostituisce pod UID o container ID;
- il limite di stato puo' scartare sequenze legittime durante burst estremi;
- le condizioni demo dipendono dagli hook e dai mask osservati sul kernel
  target e devono essere validate dal vivo.
