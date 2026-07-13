# Ordine Di Implementazione Per Policy E Detector

Questa tabella propone un ordine pratico per implementare nel nostro tool un
primo sistema di policy e detector. L'idea e' partire da una versione
userspace-first, semplice da testare, e lasciare il kernel-side filtering come
evoluzione successiva.

Il modello di riferimento e':

```text
eventi eBPF decodificati
  -> policy userspace
  -> detector engine
  -> alert output
```

| Ordine | File | Scopo preciso | Funzionalita' MVP | Dipendenze | Comunica con |
|---:|---|---|---|---|---|
| 1 | `pkg/config/config.go` | Estendere la configurazione centrale | Completato: aggiunti `Policies.Paths`, `Detectors.Paths`, `Detectors.Enabled`, `Alerts.Enabled`, `Alerts.Format` | path config esistente | Letto da CLI, runtime e detector engine |
| 2 | `pkg/cmd/cobra/cobra.go` | Esporre le nuove flag CLI | Completato: aggiunte `--policy`, `--detectors`, `--alerts`, `--alerts-output`; `--detectors` abilita automaticamente i detector | `cobra`, `pkg/config` | Produce configurazione per `pkg/cmd/project.go` |
| 3 | `pkg/cmd/project.go` | Collegare config, runtime e detector layer | Completato come ponte applicativo: `ProjectRunner` prepara `RuntimeExtensions` con policy, detector e alert prima di avviare eBPF | `pkg/config`, futuro `pkg/policy`, futuro `pkg/detectors` | Coordina `pkg/ebpf/project.go` e output |
| 4 | `pkg/policy/policy.go` | Definire il modello interno di policy | Completato: aggiunti `Policy`, `Rule`, `Scope`, `EventSelector`, `EventInput`, `PolicyMode`, `PolicyIntent`, matching base su evento, comm e uid | nessuna dipendenza esterna | Usato da loader, manager e detector dispatcher |
| 5 | `pkg/policy/loader.go` | Caricare policy dichiarative | Completato: caricamento YAML da file o directory, supporto `mode`/`intent`, validazione e collegamento al runner | `os`, `path/filepath`, `gopkg.in/yaml.v3` | Produce `[]Policy` per runner e futuro manager |
| 6 | `pkg/policy/manager.go` | Gestire policy attive a runtime | Completato: `Manager`, `MatchResult`, `Match`, `IsEventSelected`, `SelectedEvents`; `suppress` vince su `monitor`/`detect` | `pkg/policy` | Usato dal runner, futuro runtime e detector dispatcher |
| 7 | `pkg/detectors/detector.go` | Definire il contratto dei detector | Completato: interfaccia `Detector`, alias `Event`, `Definition`, `Alert`, `Flush`; aggiunti `Consumes`, `Stateful`, `Window` con finestra corta per collective anomalies locali | `context`, `time`, `pkg/bufferdecoder` | Base per detector YAML e futuri detector Go |
| 8 | `pkg/detectors/definition.go` | Descrivere cosa produce e richiede un detector | Completato: `EventRequirement`, `DetectorOutput`, `ThreatMetadata`, `AttackTactic`, `AttackTechnique`, severita', helper eventi, finestra stateful e validazioni MITRE minime | `time`, `regexp`, `sort` | Usato da registry, policy e loader YAML |
| 9 | `pkg/detectors/yaml/schema.go` | Definire il formato YAML dei detector | Completato: schema esterno con `id`, `name`, `requires`, `consumes`, `conditions`, `steps`, `output`, `threat`, `stateful`, `window`, `group_by`, `tags` | `gopkg.in/yaml.v3` nei test | Letto dal parser detector |
| 10 | `pkg/detectors/yaml/parser.go` | Convertire YAML in detector definition | Completato: `ParseBytes`, `ParseFile`, `ParseFiles`, `ParseFileSchema`, mapping verso `detectors.Definition`, parsing `window`, normalizzazione severita'/MITRE, validazione opzionale eventi con `WithSupportedEvents` | `pkg/detectors`, `gopkg.in/yaml.v3` | Produce definizioni detector validate |
| 11 | `pkg/detectors/yaml/detector.go` | Implementare il detector YAML runtime | Completato: `NewDetector`, implementazione `detectors.Detector`, matching su condizioni globali e per-step, operatori `eq`, `neq`, `contains`, `in`, `prefix`, `suffix`, `exists`, `not_exists`, `gt`, `lt`, alert YAML | `pkg/detectors`, `pkg/bufferdecoder` | Pronto per registry/dispatcher |
| 12 | `pkg/detectors/registry.go` | Registrare detector disponibili | Completato: `Registry`, `NewRegistry`, `Register`, `Get`, `List`, `Len`, `EventNames`; controllo detector nil, ID duplicati, definition invalide e ordine stabile per ID | `pkg/detectors` | Base per dispatcher e engine |
| 13 | `pkg/detectors/dispatch.go` | Instradare eventi verso i detector giusti | Completato: `Dispatcher`, indice `eventName -> []Detector`, wildcard detector, `DispatchResult`, `DetectorError`, `Dispatch`, `Flush`, errori non fatali | `pkg/detectors` | Riceve eventi decodificati, produce alert |
| 14 | `pkg/detectors/engine.go` | Orchestrare registry e dispatcher | Completato: `Engine`, `NewEngine`, `Register`, `Init`, `ProcessEvent`, `Flush`, `Metrics`; metriche minime ed errori detector isolati | `pkg/detectors` | Punto unico per futuro runtime e output alert |
| 15 | `pkg/output/alert.go` | Separare output eventi da output alert | Completato: `AlertRecord`, conversione da `detectors.Alert`, eventi correlati normalizzati, metadata, policy names da metadata e formato table compatto | `pkg/output`, `pkg/detectors` | Base per printer e runtime |
| 16 | `pkg/output/printer.go` | Estendere il printer esistente | Completato: `Printer.PrintAlert`, implementazione JSON/table, test su output alert e compatibilita' interfaccia | `pkg/output`, `pkg/detectors` | Riceve eventi e alert dal runtime |
| 17 | `pkg/ebpf/project.go` | Inserire il detector engine nella pipeline eventi | Completato: policy manager e detector engine passati al runtime, detector YAML caricati dal runner, `ProcessEvent` dopo filtri base, alert stampati con `PrintAlert` | `pkg/bufferdecoder`, `pkg/output`, `pkg/detectors`, `pkg/policy` | Core runtime della pipeline |
| 18 | `pkg/events/spec.go` / `pkg/events/ids.go` | Validare eventi usati da policy/detector | Completato: helper `Exists(name)`, `ListNames()`, `IDByName(name)` e validazione early per policy/detector YAML | `pkg/events` | Usato dal runner per validare policy e detector prima dell'avvio eBPF |
| 19 | `pkg/bufferdecoder/eventsreader.go` | Garantire evento userspace completo | Completato: test end-to-end su context/args e helper `Event.Arg*` per accesso stabile agli argomenti | `pkg/bufferdecoder`, `pkg/output/event.go` | Produce input stabile per policy/detector |
| 20 | `rules/detectors/*.yaml` | Fornire detector demo | 2-3 detector iniziali: privilege change + execve, file sensibile + chmod/chown, module activity | Formato YAML definito | Caricati da `pkg/detectors/yaml` |
| 21 | `rules/policies/*.yaml` | Fornire policy demo | Policy demo che abilita eventi base e detector correlati | `pkg/policy` | Caricate da CLI |
| 22 | `README.md` | Documentare uso utente | Esempi `--policy`, `--detectors`, output alert, demo rapida | Tool compilato | Guida per test e presentazione |
| 23 | `documentation/implementation/*.md` | Aggiornare documentazione tecnica | Spiegare design, limiti userspace-first e futura estensione eBPF | Documentazione esistente | Collegata a timeline/report |
| 24 | `pkg/policy/ebpf.go` | Futura estensione kernel-side | Solo dopo MVP: tradurre parte dei filtri in mappe eBPF | `pkg/ebpf/c/maps.h`, `project.bpf.c` | Comunica con eBPF maps |
| 25 | `pkg/ebpf/c/maps.h` / `pkg/ebpf/c/project.bpf.c` | Futura integrazione kernel filtering | Mappe per event selection, uid/comm filter, policy bitmap minima | libbpf/eBPF verifier | Riduce rumore prima di userspace |

## Ordine Consigliato Per L'MVP

Per arrivare velocemente a una demo funzionante, non serve implementare subito
tutta la tabella. L'MVP dovrebbe fermarsi ai primi 18 punti, piu' qualche file
di esempio in `rules/`.

La sequenza minima e':

```text
config/CLI
  -> policy model + loader
  -> detector interface + YAML loader
  -> detector engine
  -> runtime dispatch
  -> alert output
```

Solo dopo aver dimostrato che gli alert funzionano ha senso spostare alcuni
filtri nel kernel.

## Prima Demo Attesa

Un primo comando realistico potrebbe essere:

```bash
sudo ./dist/project \
  --events sched_process_exec,security_task_fix_setuid,security_file_open,chmod,chown \
  --policy ./rules/policies/demo.yaml \
  --detectors ./rules/detectors \
  --output table
```

L'output desiderato dovrebbe restare separato:

```text
event=sched_process_exec ...
type=alert alert=suspicious_privilege_execution detector=privilege_change_then_exec ...
```

Questa separazione e' importante: gli eventi descrivono cosa e' successo, gli
alert spiegano perche' quella sequenza e' rilevante.
