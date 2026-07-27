# Timeline della tesi

Questo file e' il punto di ingresso principale per tenere traccia del lavoro di tesi. Ogni giorno dovrebbe contenere:

- cosa e' stato fatto;
- quali file sono stati toccati;
- quali problemi sono emersi;
- quali decisioni sono state prese;
- quali link portano alle note tecniche piu' dettagliate.

L'obiettivo non e' scrivere un diario perfetto, ma accumulare materiale grezzo e ordinato per rendere piu' semplice la scrittura finale della tesi.

## Indice dei file collegati

### Diario giornaliero

- [2026-04-29 - Loader eBPF, verifier e struttura documentazione](daily/2026-04-29.md)
- [2026-05-04 - Decoder userspace e output JSON raw](daily/2026-05-04.md)
- [2026-05-05 - Selezione eventi e probe registry Tracee-light](daily/2026-05-05.md)
- [2026-05-06 - Oggetto eBPF embedded nel binario Go](daily/2026-05-06.md)
- [2026-05-12 - Merge libbpfgo, hook networking e output operativo](daily/2026-05-12.md)
- [2026-05-13 - Reader duale ring buffer/perf buffer e filtro per comm](daily/2026-05-13.md)
- [2026-05-18 - Prima demo, Docker workflow e documentazione di supporto](daily/2026-05-18.md)
- [2026-05-19 - Execve dedicata, log libbpf e direzione target-specific](daily/2026-05-19.md)
- [2026-05-20 - Security bprm check e decoder per array di stringhe](daily/2026-05-20.md)
- [2026-05-26 - security_task_fix_setuid e studio delle flag LSM_SETID](daily/2026-05-26.md)
- [2026-06-10 - Memory protection e process_vm_writev](daily/2026-06-10.md)
- [2026-06-16 - Hook process/security ad alta copertura](daily/2026-06-16.md)
- [2026-06-17 - Hook BPF object, ioctl, cgroup e signal handling](daily/2026-06-17.md)
- [2026-06-18 - Kernel tampering e superfici procfs/kprobe](daily/2026-06-18.md)
- [2026-07-02 - Policy engine, detector YAML e prossimi step](daily/2026-07-02.md)
- [2026-07-06 - Runner applicativo per policy e detector](daily/2026-07-06.md)
- [2026-07-07 - Schema e parser YAML dei detector](daily/2026-07-07.md)
- [2026-07-08 - Runtime detector YAML](daily/2026-07-08.md)
- [2026-07-10 - Piano logging strutturato con zap](daily/2026-07-10.md)
- [2026-07-13 - Validazione eventi per policy e detector](daily/2026-07-13.md)
- [2026-07-14 - Dedup temporale degli alert detector](daily/2026-07-14.md)
- [2026-07-16 - Audit registry eventi e decoder](daily/2026-07-16.md)
- [2026-07-16 - Primo filtro UID kernel-side](daily/2026-07-16-kernel-filter.md)
- [2026-07-24 - Ottimizzazione runtime e benchmark controllati](daily/2026-07-24.md)

### Implementazione

- [Overview implementazione](implementation/overview.md)
- [Userspace Go e lifecycle eBPF](implementation/userspace-lifecycle.md)
- [Protocollo eventi e buffer eBPF](implementation/event-buffer.md)
- [Decoder Go degli eventi eBPF](implementation/decoder.md)
- [Output layer e formati eventi](implementation/output.md)
- [Hook implementati](implementation/hooks.md)
- [Docker nel progetto](implementation/docker.md)
- [Misurazione prestazioni](implementation/performance.md)
- [Roadmap hook process e security](implementation/hook-roadmap.md)
- [Tracee Policy Engine](implementation/tracee-policies.md)
- [Prossimi step del tool](next-steps/README.md)
- [Detector YAML e alert correlati](next-steps/detectors-and-correlations.md)
- [Piano logging strutturato con zap](next-steps/zap-logging-plan.md)

### Debugging

- [Debugging verifier eBPF](debugging/ebpf-verifier.md)
- [Comandi utili](debugging/commands.md)

### Decisioni architetturali

- [Decision log](decisions/decision-log.md)

### Materiale tesi

- [Workspace documentale della tesi](thesis/README.md)
- [Indice canonico della tesi](thesis/definitive-outline.md)
- [Dossier preparatorio del Capitolo 1](thesis/chapters/chapter-01-introduction.md)
- [Sintesi editoriale del Capitolo 1](thesis/chapters/chapter-01-editorial-synthesis.md)
- [Workflow agenti del Capitolo 1](thesis/chapter-01-agent-workflow.md)
- [Terminologia e regole editoriali](thesis/terminology-and-style.md)
- [Mappa capitoli storica](thesis/thesis-outline.md)
- [Spunti per testo tesi](thesis/writing-notes.md)

### Riferimenti e note esistenti

- [Report tecnico corrente](report.md)
- [Note operative](Notes.md)
- [Note nuovi hook](04-NuoviHook.md)
- [Comandi rapidi](useful_commands.md)
- [Contesto operativo repository](CLAUDE.md)

## Timeline

### 2026-07-17 - Preparazione strutturale della tesi

E' stato svolto l'audit preliminare del repository `Thesis` prima della
scrittura assistita del Capitolo 1. La struttura storica a nove capitoli e'
stata sostituita da un indice canonico a sei capitoli, coerente con
l'introduzione LaTeX gia' presente.

Sono stati definiti i confini tra introduzione, background, design,
implementazione, valutazione e conclusioni. E' stato inoltre creato un dossier
del Capitolo 1 con fatti verificabili, fonti interne, research questions
candidate, contributi implementativi, novelty da verificare e decisioni ancora
aperte.

La prima ondata di analisi agentica ha poi prodotto tre report indipendenti:
ricerca bibliografica, audit tecnico del repository e revisione argomentativa.
I risultati sono stati consolidati in una sintesi editoriale vincolante che
fissa tre research questions, quattro contributi principali, scope, wording
della candidate novelty e claims policy per il futuro scrittore LaTeX.

L'abstract non e' stato modificato. La regola editoriale stabilita prevede che
ogni capitolo venga prima preparato nella documentazione e solo successivamente
integrato nel repository LaTeX da un unico agente editoriale.

**Note collegate:**

- [Workspace documentale della tesi](thesis/README.md)
- [Indice canonico](thesis/definitive-outline.md)
- [Dossier del Capitolo 1](thesis/chapters/chapter-01-introduction.md)
- [Sintesi editoriale del Capitolo 1](thesis/chapters/chapter-01-editorial-synthesis.md)
- [Workflow agenti del Capitolo 1](thesis/chapter-01-agent-workflow.md)
- [Terminologia e regole editoriali](thesis/terminology-and-style.md)

### 2026-07-16 - Audit registry eventi e decoder

E' stato rafforzato il contratto tra decoder, registry probe e CLI. I test ora
verificano entrambe le direzioni:

- ogni probe pubblico deve avere uno schema decoder;
- ogni evento presente nel decoder deve essere esposto da almeno un probe
  pubblico.

In `pkg/events/spec_test.go` e' stato aggiunto anche un round-trip su ID e nome
evento, cosi' un errore nel mapping `EventID -> name -> EventID` fallisce subito
in test.

Per rendere l'audit ripetibile e usare l'ambiente CGO corretto per `libbpfgo`,
il Makefile espone ora:

```bash
make test-registry
```

Il comando e' passato e conferma che, allo stato attuale, non ci sono eventi
decodificabili rimasti fuori da `--list-events`.

**Note collegate:**

- [Diario dettagliato del giorno](daily/2026-07-16.md)
- [Decoder Go degli eventi eBPF](implementation/decoder.md)
- [Comandi utili](debugging/commands.md)

### 2026-07-16 - Primo filtro UID kernel-side

E' stato implementato il primo filtro kernel-side minimale: allowlist su UID
corrente. Il filtro viene scritto dal runtime Go in `config_map` e letto dagli
hook in `init_program_data()` prima di costruire context, argomenti e submit.

La CLI espone:

```text
--kernel-filter-uid-enabled
--kernel-filter-uid <uid>
```

Questa feature non sostituisce policy e detector. Serve a ridurre traffico
kernel-to-userspace nelle run mirate e nei benchmark. La build eBPF e il binario
Go passano con il nuovo layout di `config_entry_t`.

Sono stati raccolti anche primi campioni di benchmark. Il profilo
policy/detector collective resta sopra il target del 5% in diversi campioni,
mentre il profilo con filtro UID kernel-side riduce sensibilmente RSS e resta
sotto il target nei campioni osservati. La misura non e' ancora conclusiva
perche' le run sono state interrotte prima dei 120 secondi.

**Note collegate:**

- [Diario dettagliato del filtro UID](daily/2026-07-16-kernel-filter.md)
- [Misurazione prestazioni](implementation/performance.md)
- [Comandi utili](debugging/commands.md)

### 2026-07-14 - Dedup temporale degli alert detector

E' stato aggiunto un primo meccanismo anti-rumore nel detector engine. Gli
alert identici prodotti dallo stesso detector sulla stessa sorgente evento
vengono emessi una sola volta nella finestra breve di default pari a 5 secondi.

La modifica e' centralizzata in `demo_project/pkg/detectors/engine.go`, quindi
non sporca i singoli detector YAML e non cambia il comportamento degli hook
eBPF. Gli eventi continuano a essere processati normalmente; viene soppressa
solo la stampa ripetuta degli alert gia' prodotti.

Le metriche del detector engine distinguono ora `AlertsEmitted` e
`AlertsSuppressed`, rendendo osservabile il dedup. Sono stati aggiunti test per
verificare soppressione dentro la finestra e riemissione dopo la scadenza.

Nella stessa fase e' stato aggiunto il primo detector collective MVP:
`privilege-exec-chain`. Il detector correla una transizione
`security_task_fix_setuid(new_euid=0)` con una successiva
`sched_process_exec(uid=0)` entro 5 secondi, producendo un alert con entrambi
gli eventi sorgente.

La prima versione usava una chiave `global`, utile per testare rapidamente
flussi come `sudo`, ma troppo permissiva. In quella fase il detector e' stato
ristretto a `group_by: host_pid`, correlando solo eventi appartenenti allo
stesso PID visto nel namespace host.

L'output table degli alert collective e' stato reso piu' leggibile con il campo
`sequence`, che mostra l'intera catena correlata senza duplicare i campi
`source_*`. Infine sono state aggiunte policy preset in `rules/policies` per
testare separatamente full demo, collective privilege-exec, point process
security, sensitive file access e kernel module activity.

Successivamente la chiave di correlazione e' stata evoluta da `host_pid` a
`process_tree`. Il nuovo `group_by` usa informazioni gia' presenti nel context
evento (`host_pid`, `host_ppid`, `start_time`, `parent_start_time`) per
correlare eventi dello stesso processo o tra parent e child, senza tornare a una
chiave globale.

Sono stati aggiunti tre detector collective ulteriori:

- `privilege-sensitive-file-chain`;
- `memfd-exec-chain`;
- `kernel-module-kprobe-chain`.

E' stata aggiunta anche la policy `collective-local-chains.yaml` per caricare
insieme tutte le catene collective locali.

I detector demo sono stati poi resi piu' stringenti per ridurre rumore da
flussi legittimi come `sudo` e `unix_chkpwd`. In particolare, le catene
privilege-based richiedono una transizione da utente non-root verso euid 0 e
filtrano i passaggi ausiliari meno utili per la demo.

Infine il mapping MITRE dichiarato nei detector YAML viene propagato negli
alert: il formato table mostra `mitre=...`, mentre il formato JSON conserva il
blocco `threat` strutturato. La prossima evoluzione non e' piu' "mostrare
MITRE", ma usarlo per selezione policy/detector e report di copertura ATT&CK.

**Note collegate:**

- [Diario dettagliato del giorno](daily/2026-07-14.md)
- [Detector YAML e alert correlati](next-steps/detectors-and-correlations.md)
- [Comandi utili](debugging/commands.md)

### 2026-07-13 - Validazione eventi per policy e detector

Sono stati completati gli step 18 e 19 del piano policy/detector. Il package
`demo_project/pkg/events` espone ora helper ufficiali per interrogare il
registry eventi del decoder: `ListNames()`, `Exists(name)` e `IDByName(name)`.

Il runner applicativo usa questi helper per validare i nomi evento dichiarati
da policy e detector YAML prima dell'avvio eBPF. Se un file dichiarativo
referenzia un evento non presente nel decoder, il tool fallisce subito durante
il bootstrap invece di partire con una configurazione silenziosamente inutile.

Sono stati aggiunti test per il registry eventi e per il rifiuto di policy e
detector con eventi inesistenti.

Inoltre e' stato rafforzato il contratto dell'evento userspace passato ai
detector. Il package `pkg/bufferdecoder` espone ora helper `Event.Arg*` per
leggere argomenti per nome, e il detector YAML usa `event.ArgString` invece di
scorrere manualmente `Event.Args`. Un test end-to-end verifica che
`DecodeEvent` preservi context e argomenti essenziali.

Infine sono stati aggiunti due detector demo locali con mapping MITRE:
`privileged-uid-change` su `security_task_fix_setuid` e
`kernel-module-activity` su `do_init_module`. La policy demo e' stata estesa
per includere gli eventi necessari. Il detector su `mprotect` e' stato rimandato
perche' richiede operatori bitmask espliciti nel runtime YAML.

**Note collegate:**

- [Diario dettagliato del giorno](daily/2026-07-13.md)
- [Ordine implementazione policy/detector](next-steps/policy-detector-implementation-order.md)
- [Userspace Go e lifecycle eBPF](implementation/userspace-lifecycle.md)

### 2026-07-10 - Piano logging strutturato con zap

E' stato definito il piano di migrazione del logging runtime verso
`go.uber.org/zap`. La scelta nasce dalla necessita' di separare in modo netto
tre flussi diversi: log applicativi, eventi raw e alert detector.

Il piano stabilisce che eventi e alert restano responsabilita' del layer
`pkg/output`, mentre i messaggi di lifecycle, debug, errore e diagnostica
devono passare da un logger strutturato.

La Fase 1 e' stata implementata con `demo_project/pkg/logging/logger.go` e
`demo_project/pkg/logging/logger_test.go`. Il package costruisce un
`*zap.Logger` con livelli `debug`, `info`, `warn`, `error`, output su `stderr`
e formato `console` o `json`.

La Fase 2 ha collegato il logger al runner applicativo in
`demo_project/pkg/cmd/project.go` e al runtime eBPF in
`demo_project/pkg/ebpf/project.go` tramite `projectebpf.WithLogger(logger)`.
Il runtime usa `zap.NewNop()` se nessun logger viene fornito.

La fase successiva ha migrato i log diagnostici interni di
`pkg/ebpf/project.go`: attach probe, primi eventi ricevuti e decodificati,
drop reason, eventi persi dal perf buffer, errori di decode/output, errori
detector e print alert. Eventi raw e alert restano sui rispettivi printer.

Il callback libbpf e' stato poi instradato nello stesso logger zap. I messaggi
CO-RE/libbpf restano filtrati da `--log-level`, ma ora sono distinguibili con
`source=libbpf` e `libbpf_level=<livello>`.

E' stata aggiunta anche la flag `--log-format console|json`, separata da
`--output` e `--alerts-output`. In questo modo il tool puo' produrre eventi e
alert nel formato scelto dal monitoraggio, ma tenere i log runtime in JSON per
ambienti containerizzati o pipeline centralizzate.

La fase zap e' stata chiusa con una decisione esplicita: zap non viene usato
per stampare eventi eBPF o alert detector. Gli eventi e gli alert restano nel
package `pkg/output`, perche' sono dati di monitoraggio e non diagnostica
applicativa.

E' stata poi aggiunta la modalita' `--alerts-only`: gli eventi raw continuano a
entrare nella pipeline e ad alimentare policy/detector, ma non vengono stampati
su stdout. Questo rende piu' leggibili demo e test dei detector, perche'
l'output mostra solo gli alert prodotti.

La verifica finale eseguita e':

```bash
PKG_CONFIG_PATH=./dist/libbpf/obj \
CGO_CFLAGS="$(PKG_CONFIG_PATH=./dist/libbpf/obj pkg-config --cflags libbpf 2>/dev/null) -I$(pwd)/3rdparty/libbpfgo" \
CGO_LDFLAGS="$(PKG_CONFIG_PATH=./dist/libbpf/obj pkg-config --libs libbpf 2>/dev/null)" \
GOCACHE=/tmp/go-build \
go test ./pkg/config ./pkg/cmd/cobra ./pkg/logging ./pkg/cmd ./pkg/ebpf ./cmd/project/cmd
```

La migrazione sara' progressiva: prima i log non-hot-path, poi la diagnostica
debug, infine eventuali benchmark per verificare che il logging non peggiori il
target prestazionale del 5% di un core nelle run normali.

**Note collegate:**

- [Diario dettagliato del giorno](daily/2026-07-10.md)
- [Piano logging strutturato con zap](next-steps/zap-logging-plan.md)
- [Userspace lifecycle](implementation/userspace-lifecycle.md)
- [Misurazione prestazioni](implementation/performance.md)

### 2026-07-08 - Runtime detector YAML

Sono stati completati gli step 11, 12, 13 e 14 del piano policy/detector con
`demo_project/pkg/detectors/yaml/detector.go` e
`demo_project/pkg/detectors/registry.go` e
`demo_project/pkg/detectors/dispatch.go` e
`demo_project/pkg/detectors/engine.go`.

Il nuovo runtime detector prende un `Parsed` prodotto dal parser YAML e lo
trasforma in un oggetto che implementa `detectors.Detector`. Il detector valuta
condizioni globali e condizioni per-step sul singolo evento decodificato, poi
produce `detectors.Alert` usando titolo, descrizione e severita' dichiarati nel
file YAML.

Sono supportati campi su evento, contesto processo e argomenti (`args.<nome>`)
con operatori semplici come `eq`, `neq`, `contains`, `in`, `prefix`, `suffix`,
`exists`, `not_exists`, `gt` e `lt`.

Nota aggiornata: in questa fase la correlazione stateful non era ancora
presente. Successivamente e' stato aggiunto un MVP collective nel runtime YAML
e la chiave `process_tree` per ridurre l'uso di correlazioni globali da demo.

E' stato poi aggiunto il registry dei detector. Il registry registra detector
validati, rifiuta ID duplicati, mantiene una lista stabile ordinata per ID ed
espone l'insieme degli eventi consumati globalmente. Questo prepara il dispatcher
che instradera' gli eventi decodificati verso i detector corretti.

Il dispatcher e' stato implementato come componente separato: costruisce un
indice `eventName -> []Detector`, chiama solo i detector interessati, raccoglie
alert e conserva gli errori dei singoli detector senza bloccare l'intero flusso.
Questo e' il primo punto concreto di collegamento tra eventi decodificati e
detector caricati.

Infine e' stato aggiunto l'engine detector. L'engine coordina registry e
dispatcher, inizializza i detector, processa eventi decodificati, supporta
`Flush` e mantiene metriche minime su eventi, detector invocati, alert ed
errori.

E' stato poi introdotto il primo modello di output per gli alert. `AlertRecord`
separa gli alert dei detector dagli eventi raw, conserva detector, severita',
metadata, policy names e gli eventi correlati normalizzati.

Il printer e' stato quindi esteso con `PrintAlert`. `JSONPrinter` e
`TablePrinter` possono ora stampare alert detector senza modificare la stampa
degli eventi raw.

Infine il detector engine e' stato inserito nella pipeline principale. Il runner
carica detector YAML da file o directory, costruisce l'engine e lo passa al
runtime eBPF. Dopo decode, event filter, comm filter e policy filter, ogni
evento viene inviato a `Engine.ProcessEvent`; gli alert prodotti vengono
stampati con `Printer.PrintAlert` quando `--alerts` e' attivo. Il prossimo
passaggio e' migliorare leggibilita' e rumore dell'output alert.

Il test runtime con `rules/policies/demo-detectors.yaml` e
`rules/detectors/sensitive_file_open.yaml` ha prodotto alert reali:

```text
type=alert alert=Sensitive system file opened severity=medium detector=sensitive-file-open events=1 detector_name=Sensitive file open source_event=security_file_open source_pid=... source_uid=... source_comm=cat source_args=pathname=/etc/passwd,...
event=security_file_open ... args=pathname=/etc/passwd,...
```

Il risultato conferma la pipeline completa policy/detector/alert. Il limite
emerso e' di leggibilita': eventi raw e alert condividono lo stesso stream, per
cui durante una demo e' utile usare `--alerts-only` oppure filtrare
`type=alert`.

In seguito il detector e' stato ristretto per ridurre rumore: la condizione su
`args.pathname` non usa piu' un prefisso generico `/etc/`, ma `operator: in` su
una lista esplicita di file sensibili. In questo modo aperture ricorrenti e
legittime come `/etc/hosts` e `/etc/ld.so.cache` non generano piu' alert.

Nella stessa giornata e' stato risolto un bug end-to-end della pipeline eventi:
gli hook producevano eventi con nome corretto ma senza argomenti (`args=-`).
Il debug ha evidenziato due cause operative. La prima era una ricompilazione
eBPF incompleta: il Makefile non tracciava gli header `.h`, quindi
`dist/project.bpf.o` poteva restare obsoleto dopo modifiche a `types.h`. La
seconda era un mismatch nel decoder: `eventContextSize` era ancora 128 byte,
mentre il context reale e' 136 byte dopo l'aggiunta di `policies_version` e
`matched_policies`. Il Makefile ora dipende dagli header eBPF e il decoder usa
il layout corretto.

**Note collegate:**

- [Diario dettagliato del giorno](daily/2026-07-08.md)
- [Piano di implementazione](next-steps/implementation-plan.md)
- [Ordine implementazione policy/detector](next-steps/policy-detector-implementation-order.md)

### 2026-07-07 - Schema e parser YAML dei detector

Sono stati completati gli step 9 e 10 del piano policy/detector con
`demo_project/pkg/detectors/yaml/schema.go` e
`demo_project/pkg/detectors/yaml/parser.go`.

Il nuovo package definisce il formato YAML esterno dei detector: identificativi,
eventi richiesti e consumati, modalita' stateful, finestra temporale,
`group_by`, condizioni, step, output alert, metadata MITRE ATT&CK e tag.

La scelta architetturale e' mantenere lo schema solo come rappresentazione del
file utente. La conversione verso il modello runtime interno
`detectors.Definition` e' stata implementata nel parser YAML.

Il parser espone `ParseBytes`, `ParseFile`, `ParseFiles` e `ParseFileSchema`,
normalizza severita', finestra temporale e metadata MITRE, conserva condizioni e
step per il futuro runtime YAML e supporta validazione opzionale dei nomi evento
tramite `WithSupportedEvents`.

Sono stati aggiunti test di unmarshalling e parsing con un detector YAML
completo, cosi' da fissare il formato prima di implementare il detector runtime.

**Note collegate:**

- [Diario dettagliato del giorno](daily/2026-07-07.md)
- [Piano di implementazione](next-steps/implementation-plan.md)
- [Ordine implementazione policy/detector](next-steps/policy-detector-implementation-order.md)

### 2026-07-06 - Runner applicativo e modello policy

Sono stati completati gli step 3 e 4 del piano policy/detector.

Il runner applicativo in `demo_project/pkg/cmd/project.go` prepara ora un blocco
`RuntimeExtensions` prima di avviare il runtime eBPF. Questo blocco raccoglie
path policy, path detector, stato detector, stato alert e formato alert.

E' stato inoltre creato `demo_project/pkg/policy/policy.go`, che definisce il
modello interno normalizzato delle policy: `Policy`, `Scope`, `Rule`,
`EventSelector` ed `EventInput`. Il modello supporta matching minimo su nome
evento, `comm` e `uid` ed e' indipendente dal futuro formato YAML.

Come elemento di novelty, la policy include anche `PolicyMode` e `PolicyIntent`:
il primo distingue monitoraggio, detection e soppressione del rumore, mentre il
secondo descrive lo scopo operativo, per esempio threat hunting, hardening,
compliance o noise reduction.

E' stato poi aggiunto `demo_project/pkg/policy/loader.go`, che carica policy
YAML da file o directory e le converte nel modello interno. Il runner
applicativo carica gia' queste policy in `RuntimeExtensions.Policies`, anche se
il matching runtime verra' introdotto nello step successivo con il manager.

Lo step manager e' stato completato con `demo_project/pkg/policy/manager.go`.
Il manager valuta gli eventi rispetto alle policy attive e produce un
`MatchResult` con policy matchate e segnali operativi `Monitor`, `Detect` e
`Suppressed`. La regola scelta e' che `suppress` vince su `monitor` e `detect`.

E' stato infine creato `demo_project/pkg/detectors/detector.go`, che definisce
il contratto dei detector: metadata, inizializzazione, gestione evento, flush
periodico e alert preliminare. La definizione del detector ora dichiara anche
gli eventi consumati (`Consumes`) e se il detector e' stateful. Per i detector
contestuali e' stata introdotta una finestra temporale corta: default `2s`,
massimo `5s`. Questa scelta abilita correlazioni semplici senza trattenere stato
per troppo tempo; se i benchmark peggiorano, la feature va mitigata o limitata.

Prima di procedere con `pkg/detectors/definition.go` e' stato preparato anche
un documento di proposta per allineare detector e policy a MITRE ATT&CK:
tattiche, tecniche, data source e futura vista di copertura.

Lo step `pkg/detectors/definition.go` e' stato poi completato. La definizione
dei detector e' stata separata dal contratto runtime e ora include requisiti
evento, output atteso, severita', metadati MITRE ATT&CK, helper per eventi
consumati e validazioni minime. I detector senza mapping MITRE restano ammessi,
ma possono essere trattati come `unmapped`.

**Note collegate:**

- [Diario dettagliato del giorno](daily/2026-07-06.md)
- [Piano di implementazione](next-steps/implementation-plan.md)
- [Ordine implementazione policy/detector](next-steps/policy-detector-implementation-order.md)
- [Proposte allineamento MITRE ATT&CK](next-steps/mitre-attack-alignment-proposals.md)

### 2026-04-29

**Tema principale:** passaggio da skeleton userspace a runtime eBPF caricabile, debug del verifier, introduzione di una documentazione strutturata.

**Attivita' svolte:**

- Compilazione di `demo_project/pkg/ebpf/c/project.bpf.c` in `project.bpf.o`.
- Introduzione del runtime eBPF in Go in `demo_project/pkg/ebpf/project.go`.
- Aggiunta del caricamento dell'oggetto eBPF da filesystem.
- Aggiunta dell'attach esplicito di raw tracepoint e kprobe.
- Debug di errori del verifier legati a `args_buf.offset`.
- Modifica di `buffer.h` per usare slot fissi per argomenti scalari e stringhe.
- Verifica che il programma arrivi al loop runtime senza errori di load.
- Chiarimento iniziale: il programma non stampava ancora alert perche' mancavano decoder/output/detection.
- Creazione della struttura documentale principale.

**File tecnici principali:**

- `demo_project/pkg/ebpf/project.go`
- `demo_project/pkg/ebpf/c/common/buffer.h`
- `demo_project/pkg/ebpf/c/common/common.h`
- `demo_project/pkg/ebpf/c/common/arch.h`
- `demo_project/pkg/ebpf/c/common/task.h`
- `demo_project/pkg/cmd/initialize/bpfobject.go`
- `demo_project/pkg/cmd/cobra/cobra.go`
- `demo_project/cmd/project/cmd/root.go`

**Note collegate:**

- [Diario dettagliato del giorno](daily/2026-04-29.md)
- [Debugging verifier eBPF](debugging/ebpf-verifier.md)
- [Protocollo eventi e buffer eBPF](implementation/event-buffer.md)
- [Userspace Go e lifecycle eBPF](implementation/userspace-lifecycle.md)
- [Decoder Go degli eventi eBPF](implementation/decoder.md)

### 2026-05-04

**Tema principale:** integrazione del decoder userspace e primo output JSON raw.

**Attivita' svolte:**

- Aggiunto package `demo_project/pkg/bufferdecoder`.
- Introdotto `protocol.go` con il primo layout Go di `EventContext`.
- Introdotto `decoder.go` con primitive di lettura binaria.
- Introdotto `eventsreader.go` per decodificare eventi completi e argomenti.
- Aggiunti test minimi per argomenti scalari e stringhe.
- Collegato `Project.Run()` al decoder e a output JSON su stdout.
- Verificato output reale da `cap_capable`, con argomento `cap` decodificato.

Nota successiva: il layout e' stato poi esteso a 136 byte con
`policies_version` e `matched_policies`.

**File tecnici aggiunti/toccati:**

- `demo_project/pkg/bufferdecoder/protocol.go`
- `demo_project/pkg/bufferdecoder/decoder.go`
- `demo_project/pkg/bufferdecoder/eventsreader.go`
- `demo_project/pkg/bufferdecoder/eventsreader_test.go`
- `demo_project/pkg/ebpf/project.go`

**Note collegate:**

- [Diario dettagliato del giorno](daily/2026-05-04.md)
- [Decoder Go degli eventi eBPF](implementation/decoder.md)
- [Userspace Go e lifecycle eBPF](implementation/userspace-lifecycle.md)
- [Protocollo eventi e buffer eBPF](implementation/event-buffer.md)

**Prossimo passo consigliato:**

Rendere l'output piu' leggibile:

- `comm` e `uts_name` come stringhe;
- capability numeriche convertite in nomi (`CAP_SYS_PTRACE`, ecc.);
- filtro o flag debug per hook rumorosi come `cap_capable`.

### 2026-05-05

**Tema principale:** introduzione di una selezione eventi/probe ispirata a Tracee, ma semplificata per l'MVP e per il kernel Rocky Linux 4.18.

**Attivita' svolte:**

- Aggiunta della configurazione `Events` in `demo_project/pkg/config/config.go`.
- Aggiunte le flag CLI `--events` e `--drop-events`.
- Spostata la lista statica degli hook in un registry unico in `demo_project/pkg/ebpf/probes/probes.go`.
- Introdotta la selezione dei probe da attaccare in base agli eventi richiesti.
- Aggiunto filtro userspace sugli eventi decodificati come controllo difensivo.
- Aggiunti test unitari per default, include/exclude, eventi non supportati e selezione vuota.
- Introdotto `demo_project/pkg/output` come layer separato per la stampa eventi.
- Aggiunti printer `json` e `table`, con test dedicati.
- Normalizzato il JSON di output: `comm` e `uts_name` sono stringhe, non array di byte.
- Aggiunto mapping delle Linux capabilities per eventi `cap_capable` (`21` -> `CAP_SYS_ADMIN`, ecc.).
- Segnalato il testo descrittivo della CLI ancora da rinominare da `Vesuvius` a `Project`.

**File tecnici aggiunti/toccati:**

- `demo_project/pkg/config/config.go`
- `demo_project/pkg/cmd/cobra/cobra.go`
- `demo_project/pkg/ebpf/project.go`
- `demo_project/pkg/ebpf/probes/probes.go`
- `demo_project/pkg/ebpf/probes/probes_test.go`
- `demo_project/pkg/output/printer.go`
- `demo_project/pkg/output/json.go`
- `demo_project/pkg/output/table.go`
- `demo_project/pkg/output/event.go`
- `demo_project/pkg/output/json_test.go`
- `demo_project/pkg/output/table_test.go`
- `demo_project/cmd/project/cmd/root.go`

**Decisione progettuale:**

Tracee usa un sistema piu' completo basato su definizioni evento, probe group e pipeline. Per questo progetto e' stato scelto un registry statico e selezionabile: riduce il rumore di hook come `cap_capable`, mantiene l'attach esplicito e resta piu' prevedibile sul kernel Rocky Linux 4.18.

**Note collegate:**

- [Diario dettagliato del giorno](daily/2026-05-05.md)
- [Userspace Go e lifecycle eBPF](implementation/userspace-lifecycle.md)
- [Output layer e formati eventi](implementation/output.md)
- [Decision log](decisions/decision-log.md)
- [Comandi utili](debugging/commands.md)

### 2026-05-06

**Tema principale:** embedding dell'oggetto eBPF nel binario Go, seguendo il
pattern usato da Tracee.

**Attivita' svolte:**

- Aggiunto `demo_project/embedded-ebpf.go` con `go:embed`.
- Spostato l'output eBPF di default a `demo_project/dist/project.bpf.o`.
- Aggiornato `Makefile`: `build`, `test` e `run` dipendono da `bpf`.
- Aggiornato `initialize.BPFObject()` per usare l'oggetto embedded quando non
  viene passato un path esplicito.
- Lasciati disponibili `--bpf-object` e `PROJECT_BPF_OBJECT` come override.
- Impostato `config.Default().BPFObjPath` a stringa vuota per indicare uso
  dell'embedded object.

**File tecnici aggiunti/toccati:**

- `demo_project/embedded-ebpf.go`
- `demo_project/Makefile`
- `demo_project/pkg/cmd/initialize/bpfobject.go`
- `demo_project/pkg/config/config.go`

**Verifiche:**

- `make bpf`
- `GOCACHE=/tmp/go-build go test ./...`
- `go run ./cmd/project --help`
- `go build -o /tmp/project ./cmd/project`

**Note collegate:**

- [Diario dettagliato del giorno](daily/2026-05-06.md)
- [Decision log](decisions/decision-log.md)
- [Comandi utili](debugging/commands.md)

### 2026-05-12

**Tema principale:** consolidamento post-merge con `libbpfgo`, integrazione
degli hook networking del branch collaboratore e chiarimento dei limiti attuali
tra ring buffer e perf buffer.

**Attivita' svolte:**

- Confermato `cmd/project` come entrypoint ufficiale del tool.
- Aggiornato il Makefile per buildare con `libbpfgo` e `libbpf` statico.
- Aggiunto target `filtered` per test rapidi di `security_socket_connect`.
- Esteso il registry probe con gli hook networking:
  `security_socket_create`, `security_socket_listen`,
  `security_socket_connect`, `security_socket_accept`,
  `security_socket_bind`, `security_socket_setsockopt`,
  `security_socket_recvmsg`, `security_socket_sendmsg`.
- Esteso `protocol.go` con gli schemi di decoding per gli eventi socket
  principali.
- Ripristinata `events_ringbuf_submit` per gli hook che inviano eventi sulla
  ring buffer.
- Corretto l'indice degli argomenti salvati dagli hook process/security, evitando
  errori come `argument index 1 out of range for event cap_capable`.
- Verificato output `table` leggibile per process lifecycle e security hooks.
- Chiarito come testare `security_settime64`, `security_task_prctl` e
  `security_socket_connect`.

**Decisione/nota tecnica:**

Al 2026-05-12 il runtime Go leggeva solo `events_ringbuf`, mentre diversi hook
networking importati usavano ancora `events_perf_submit`. Per questo motivo gli
hook potevano essere compilati e attaccati, ma gli eventi networking su perf
buffer richiedevano un prossimo intervento: aggiungere un perf-buffer reader o
migrare quegli hook a `events_ringbuf_submit`. Questa limitazione e' stata
superata il 2026-05-13 con l'aggiunta di `InitPerfBuf("events", ...)`.

**File tecnici principali:**

- `demo_project/Makefile`
- `demo_project/pkg/ebpf/project.go`
- `demo_project/pkg/ebpf/probes/probes.go`
- `demo_project/pkg/bufferdecoder/protocol.go`
- `demo_project/pkg/ebpf/c/project.bpf.c`
- `demo_project/pkg/ebpf/c/common/buffer.h`

**Note collegate:**

- [Diario dettagliato del giorno](daily/2026-05-12.md)
- [Hook implementati](implementation/hooks.md)
- [Userspace lifecycle](implementation/userspace-lifecycle.md)
- [Comandi utili](debugging/commands.md)
- [Decision log](decisions/decision-log.md)

### 2026-05-13

**Tema principale:** aggiornamento del runtime alla nuova versione con lettura
sia da ring buffer sia da perf buffer, piu' filtro userspace per `comm`.

**Attivita' svolte:**

- Analizzato `demo_project/pkg/ebpf/project.go` aggiornato.
- Verificato che `Project.Init()` apre:
  - `events_ringbuf` tramite `InitRingBuf`;
  - `events` tramite `InitPerfBuf`.
- Verificato che `Project.Run()` ascolta entrambi i canali e passa i record
  raw a `handleRawEvent`.
- Documentato il nuovo handler comune per decode, filtro eventi, filtro comm e
  output.
- Documentato il nuovo filtro CLI `--comms`, basato su
  `cfg.Events.FilterComms`.
- Aggiornata la nota precedente: gli eventi networking su perf buffer non sono
  piu' bloccati dalla mancanza di reader userspace.
- Registrato il target Makefile `filtered` per test veloci di
  `security_socket_connect`.

**Decisione/nota tecnica:**

Il progetto e' entrato in una fase di transizione con doppio canale:
ring buffer per gli hook gia' migrati a `events_ringbuf_submit` e perf buffer
per gli hook networking compatibili con il modello Tracee. A breve andra'
scelta una strategia stabile: mantenere entrambi, migrare tutto a perf buffer
o migrare tutto a ring buffer.

**File tecnici principali:**

- `demo_project/pkg/ebpf/project.go`
- `demo_project/pkg/config/config.go`
- `demo_project/pkg/cmd/cobra/cobra.go`
- `demo_project/pkg/ebpf/c/maps.h`
- `demo_project/pkg/ebpf/c/common/buffer.h`
- `demo_project/Makefile`

**Note collegate:**

- [Diario dettagliato del giorno](daily/2026-05-13.md)
- [Userspace lifecycle](implementation/userspace-lifecycle.md)
- [Protocollo eventi e buffer eBPF](implementation/event-buffer.md)
- [Comandi utili](debugging/commands.md)
- [Decision log](decisions/decision-log.md)

### 2026-05-18

**Tema principale:** preparazione della prima demo tecnica e introduzione di un
workflow Docker per rendere piu' riproducibile la build.

**Attivita' svolte:**

- Preparato materiale di supporto per la demo in `documentation/demo`.
- Raffinati i diagrammi LaTeX/PDF della pipeline.
- Preparato un breve discorso in italiano con riferimenti ai file principali.
- Aggiornato `demo_project/README.md` come guida piu' completa al progetto.
- Aggiunto `demo_project/Dockerfile`.
- Aggiunto `demo_project/.dockerignore`.
- Esteso `demo_project/Makefile` con:
  - `docker-image`;
  - `docker-build`;
  - `docker-shell`;
  - `docker-run`.
- Gestiti i problemi DNS nei container usando host networking per build e shell.
- Modificato il mount Docker per mantenere lo stesso path assoluto dell'host,
  evitando problemi con path generati da `pkg-config` e `libbpf.pc`.
- Aggiunta documentazione narrativa su Docker in
  `documentation/implementation/docker.md`.

**Decisione/nota tecnica:**

Docker viene usato come supporto alla build e allo sviluppo, non come
isolamento completo del runtime eBPF. L'esecuzione del tool in container usa
comunque il kernel host e richiede privilegi, BTF e accesso a `/sys/fs/bpf`.

**File tecnici principali:**

- `demo_project/Dockerfile`
- `demo_project/.dockerignore`
- `demo_project/Makefile`
- `demo_project/README.md`
- `documentation/implementation/docker.md`
- `documentation/demo/first_demo.md`
- `documentation/demo/first_demo.tex`
- `documentation/demo/speech.md`

**Note collegate:**

- [Diario dettagliato del giorno](daily/2026-05-18.md)
- [Docker nel progetto](implementation/docker.md)
- [Decision log](decisions/decision-log.md)

### 2026-05-19

**Tema principale:** ottimizzazione target-specific degli eventi `execve` e
`execveat`, pulizia dei log libbpf e consolidamento della CLI.

**Attivita' svolte:**

- Aggiunto/raffinato l'evento `execve` come tentativo di esecuzione.
- Aggiunto l'evento `execveat` come syscall separata, con payload dedicato:
  `dirfd`, `pathname`, `flags`.
- Chiarita la differenza tra:
  - `execve`: tentativo di eseguire un binario;
  - `sched_process_exec`: exec riuscita.
- Spostato `execve` da un secondo hook generico su
  `raw_tracepoint/sys_enter` al tracepoint dedicato
  `tracepoint/syscalls/sys_enter_execve`.
- Agganciato `execveat` al tracepoint dedicato
  `tracepoint/syscalls/sys_enter_execveat`.
- Esteso il registry probe Go con supporto a `AttachTracepoint`.
- Valutato e rimosso il path `_light`, mantenendo il path standard
  `init_program_data` + `events_perf_submit`.
- Collegato `--log-level` ai callback di logging libbpfgo.
- Filtrati di default i log verbose di relocation CO-RE.
- Aggiornato l'help CLI e il README per documentare `--log-level` e `--comms`.
- Aggiornato `run_table` per testare `execve`/`execveat` senza filtro `comm`.
- Verificato `execveat` con un piccolo programma C che invoca direttamente
  `SYS_execveat`.

**Decisione/nota tecnica:**

Il progetto inizia a seguire una direzione volutamente meno generalista di
Tracee: quando il kernel target e' noto, e' preferibile usare hook specifici
e piu' economici invece di dispatcher generici che girano per ogni evento del
sistema. Questa scelta e' una parte importante della novelty del lavoro.
`execveat` viene mantenuto distinto da `execve` per non perdere informazioni
su `dirfd` e `flags`, evitando pero' di duplicare lo stesso payload.

**File tecnici principali:**

- `demo_project/pkg/ebpf/c/project.bpf.c`
- `demo_project/pkg/ebpf/c/types.h`
- `demo_project/pkg/ebpf/probes/probes.go`
- `demo_project/pkg/events/ids.go`
- `demo_project/pkg/events/spec.go`
- `demo_project/pkg/ebpf/project.go`
- `demo_project/pkg/cmd/cobra/cobra.go`
- `demo_project/Makefile`
- `demo_project/README.md`

**Note collegate:**

- [Diario dettagliato del giorno](daily/2026-05-19.md)
- [Hook implementati](implementation/hooks.md)
- [Decision log](decisions/decision-log.md)
- [Comandi utili](debugging/commands.md)

### 2026-05-20

**Tema principale:** aggiunta di `security_bprm_check` come hook security nel
percorso di exec.

**Attivita' svolte:**

- Analizzato il pattern Tracee per `kprobe/security_bprm_check`.
- Implementato `trace_security_bprm_check` lato eBPF.
- Registrato l'evento nel registry Go dei probe.
- Aggiunta la specifica di decode userspace.
- Documentata la semantica dell'hook nel percorso
  `execve/execveat -> security_bprm_check -> sched_process_exec`.
- Documentato il test manuale con `whoami`.

**Decisione/nota tecnica:**

Tracee raccoglie anche `argv`/`envp` per questo evento. Nel progetto, per ora,
il payload e' limitato a `pathname`, `dev`, `inode`, `filename`, `argc`, `envc`.
Questa scelta riduce complessita' e rischio verifier sul target Rocky Linux
4.18, mantenendo comunque informazioni utili per correlare tentativo di exec,
validazione del binario ed exec riuscita.

**File tecnici principali:**

- `demo_project/pkg/ebpf/c/project.bpf.c`
- `demo_project/pkg/ebpf/probes/probes.go`
- `demo_project/pkg/events/ids.go`
- `demo_project/pkg/events/spec.go`

**Note collegate:**

- [Diario dettagliato del giorno](daily/2026-05-20.md)
- [Hook implementati](implementation/hooks.md)
- [Comandi utili](debugging/commands.md)

### 2026-05-26

**Tema principale:** aggiunta di hook semantici per transizioni UID e invio
segnali tra processi.

**Attivita' svolte:**

- Implementato `trace_security_task_fix_setuid` lato eBPF.
- Registrato l'evento nel registry probe Go.
- Aggiunta la specifica di decode per `old/new uid`, `old/new euid`,
  `old/new suid`, `old/new fsuid` e `flags`.
- Verificato che sul kernel target e' presente `security_task_fix_setuid` ma
  non `security_task_fix_setgid`.
- Documentato il significato delle flag `LSM_SETID_ID`, `LSM_SETID_RE`,
  `LSM_SETID_RES` e `LSM_SETID_FS`.
- Aggiunto un test manuale pulito basato su `sudo python3` e `os.setuid(65534)`.
- Implementato `trace_security_task_kill` per osservare relazioni
  `processo sorgente -> segnale -> processo target`.
- Documentato il test manuale con `sleep` e `kill`.
- Implementato `ptrace` con modello syscall enter/exit: argomenti salvati
  all'enter e `returnValue` aggiunto all'exit.
- Implementato `security_file_open` per osservare aperture file con path
  risolto e metadati stabili (`dev`, `inode`, `ctime`).
- Aggiunto mapping human-readable per segnali e richieste ptrace nello strato
  di output.

**Decisione/nota tecnica:**

`security_task_fix_setgid` non viene aggiunto come kprobe diretto perche' il
simbolo non e' disponibile sulla VM Rocky Linux 4.18 usata come target. Per
coprire anche le transizioni GID sara' preferibile valutare `commit_creds`,
filtrando gli eventi in cui cambiano `gid`, `egid`, `sgid` o `fsgid`.

`security_task_kill` viene implementato come hook semantico invece che come
evento syscall Tracee-like: il kernel fornisce gia' il task target, quindi
l'evento e' piu' leggibile. Il limite da ricordare e' l'assenza del return
value finale della syscall.

Per `ptrace`, invece, e' stato scelto il modello enter/exit perche' il return
value e' parte fondamentale della semantica: permette di distinguere tentativi
autorizzati e negati.

Per `security_file_open` il return value non e' stato incluso: il valore
principale e' il `struct file *` gia' risolto. Un eventuale evento
`open/openat/openat2` enter/exit potra' essere aggiunto in futuro per coprire
esplicitamente successo/fallimento.

Sul lato CLI e' stata aggiunta anche `--list-events`, che stampa gli eventi
supportati direttamente dal registry delle probe e termina prima del caricamento
eBPF. Serve come comando rapido di discovery per scegliere i nomi da usare con
`--events` e `--drop-events`.

E' stato inoltre aggiunto `chmod` come evento logico unico per
`chmod/fchmod/fchmodat`. L'implementazione usa tracepoint syscall enter/exit,
salva gli argomenti all'ingresso e invia l'evento all'uscita con
`returnValue`. La scelta segue il pattern syscall con esito finale, ma riduce
la superficie CLI rispetto a tre eventi separati.

Come complemento di `security_file_open`, e' stato aggiunto anche `open` come
evento syscall logico per `open/openat`. Questo evento porta il return value,
quindi permette di distinguere file descriptor restituiti da errori negativi.
`openat2` e' stato escluso per compatibilita' con Rocky Linux 4.18.

E' stato aggiunto anche `chown` come evento logico unico per
`chown/fchown/fchownat/lchown`. Come `chmod`, usa tracepoint syscall enter/exit,
mantiene il `returnValue` e distingue la variante tramite il campo
`operation`. L'obiettivo e' coprire cambi ownership riusciti o negati senza
moltiplicare gli eventi esposti nella CLI.

Gli hook process/security storicamente rimasti su ring buffer sono stati
migrati al perf buffer. `events_ringbuf_submit`, la mappa `events_ringbuf` e il
reader Go restano disponibili, ma per ora il transport principale e' `events`
tramite `events_perf_submit`.

## 2026-06-10 - Memory mapping e protection changes

Sono stati aggiunti gli eventi `mmap`, `mprotect` e `pkey_mprotect` usando
tracepoint syscall dedicati in coppia entry/exit. Gli argomenti vengono salvati
all'ingresso e uniti al valore di ritorno all'uscita, prima dell'invio tramite
perf buffer.

Il layer di output traduce ora i bit `PROT_*` e `MAP_*`, mostra gli indirizzi
restituiti da `mmap` in esadecimale e converte gli errori negativi in errno
simbolici. Per `mprotect` e `pkey_mprotect`, il risultato zero viene mostrato
come `success`.

E' stata inoltre estesa da sei a sette slot la struttura condivisa `args_t` e
la relativa helper `load_args`. La correzione rende valido il settimo campo
gia' usato dall'evento `chown` e mantiene un unico meccanismo di correlazione
entry/exit per le syscall.

E' stato poi aggiunto `process_vm_writev`, evento ad alto impatto per osservare
scritture dirette nella memoria di un processo. L'implementazione conserva i
metadati degli `iovec` e il numero di byte scritti, senza copiare il payload,
ed e' pronta a supportare una futura policy per process injection.

Infine e' stato aggiunto `setns`, correlando entry ed exit per distinguere i
cambi di namespace riusciti dai tentativi negati. L'output traduce `nstype`
nei nomi `CLONE_NEW*`, rendendo l'evento utilizzabile in future policy su
isolamento e container escape.

La copertura dei namespace e' stata completata con `unshare`. L'evento mostra
la bitmask `CLONE_*` e il risultato della syscall, distinguendo quindi la
creazione o separazione effettiva di risorse da un semplice tentativo.

Sono stati poi aggiunti `process_vm_readv` e `commit_creds`. Il primo completa
la visibilita' sugli accessi diretti alla memoria cross-process senza copiare
il payload. Il secondo confronta credenziali vecchie e nuove realmente
applicate, incluse capability e user namespace.

La copertura e' stata poi estesa alle transizioni namespace e alle operazioni
filesystem semantiche: `switch_task_ns`, `security_sb_mount`,
`security_sb_umount` e `security_inode_unlink`. I test live hanno verificato
un nuovo mount namespace, mount/umount di un tmpfs e cancellazione di un file.

**File tecnici principali:**

- `demo_project/pkg/ebpf/c/project.bpf.c`
- `demo_project/pkg/ebpf/c/types.h`
- `demo_project/pkg/ebpf/probes/probes.go`
- `demo_project/pkg/events/ids.go`
- `demo_project/pkg/events/spec.go`
- `documentation/implementation/hooks.md`
- `documentation/debugging/commands.md`

**Note collegate:**

- [Diario dettagliato del giorno](daily/2026-06-10.md)
- [Hook implementati](implementation/hooks.md)
- [Comandi utili](debugging/commands.md)

## 2026-06-16 - Hook process/security ad alta copertura

Sono stati aggiunti nuovi hook verificati nel sorgente locale di Tracee:
`security_bpf`, `security_kernel_read_file`,
`security_bprm_creds_for_exec`, `clone`, `fork`, `vfork`, la famiglia
`set*uid`/`set*gid` e `prlimit64`.

L'obiettivo della giornata e' stato aumentare la copertura process/security
senza introdurre hook generici troppo rumorosi. Gli eventi syscall usano il
modello entry/exit quando serve il valore di ritorno; gli hook security
rimangono invece semanticamente vicini agli oggetti kernel gia' risolti.

Il layer di output e' stato arricchito con mapping human-readable per comandi
`bpf(2)`, tipi `kernel_read_file`, flag `CLONE_*` e risorse `RLIMIT_*`.
`prlimit64` e' stato adattato al formato tracepoint del kernel Rocky Linux
4.18, usando slot a 8 byte per evitare truncation dei puntatori.

**Note collegate:**

- [Diario dettagliato del giorno](daily/2026-06-16.md)
- [Hook implementati](implementation/hooks.md)
- [Comandi utili](debugging/commands.md)

## 2026-06-17 - Hook BPF object, cgroup, moduli e signal handling

La copertura security/process e' stata estesa con nuovi hook aggiuntivi:
`security_bpf_map`, `security_bpf_prog`, `security_kernel_post_read_file`,
`security_file_ioctl`, `security_file_permission`, `cgroup_attach_task`,
`cgroup_mkdir`, `cgroup_rmdir`, `call_usermodehelper`, `do_sigaction`,
`module_load`, `module_free`, `do_init_module` e `process_execute_failed`.

Questi eventi completano alcune aree gia' aperte il giorno precedente:
`security_bpf` ora puo' essere affiancato da eventi sugli oggetti eBPF
specifici, `security_kernel_read_file` ha un checkpoint successivo alla lettura
del file, e il percorso file include anche ioctl e controlli successivi di
permesso su file gia' aperti. Inoltre il tool osserva ora migrazioni tra
cgroup, creazione/rimozione di cgroup, helper userspace avviati dal kernel,
modifiche alla gestione dei segnali, lifecycle dei moduli kernel e tentativi
falliti di exec.

Il layer di output e' stato aggiornato con label per `BPF_PROG_TYPE_*`,
`READING_*`, maschere `MAY_*`, comandi ioctl noti, modalita' `UMH_*` e metodi
handler `SIG_*`. Per i nuovi eventi sono stati aggiunti anche mapping per il
return value di `do_init_module` e per l'operazione `execve`/`execveat` fallita.
La mappa `kernel_read_file_id` e' stata corretta per il kernel target: su Rocky
Linux 4.18 `READING_MODULE` corrisponde a `2`.

**Note collegate:**

- [Diario dettagliato del giorno](daily/2026-06-17.md)
- [Hook implementati](implementation/hooks.md)
- [Comandi utili](debugging/commands.md)
- [Roadmap hook process e security](implementation/hook-roadmap.md)

## 2026-06-18 - Hook di hardening kernel

Sono stati aggiunti tre hook orientati al kernel tampering e al comportamento
dei moduli: `proc_create`, `register_kprobe` e `kallsyms_lookup_name`.

`proc_create` rende visibile la creazione di entry procfs, utile per osservare
moduli che espongono interfacce di controllo sotto `/proc`. `register_kprobe`
usa una coppia kprobe/kretprobe per mostrare il simbolo osservato, gli handler
registrati e l'esito della registrazione. `kallsyms_lookup_name` osserva lookup
runtime di simboli kernel e stampa il nome richiesto insieme all'indirizzo
restituito.

Il layer di output e' stato aggiornato per mostrare correttamente puntatori e
indirizzi kernel in esadecimale e per tradurre il `returnValue` di
`register_kprobe` in `success` o errno. I test hanno verificato build eBPF,
test Go, build del binario, presenza degli eventi in `--list-events` e attach
runtime dei nuovi hook.

**Note collegate:**

- [Diario dettagliato del giorno](daily/2026-06-18.md)
- [Hook implementati](implementation/hooks.md)
- [Comandi utili](debugging/commands.md)
- [Roadmap hook process e security](implementation/hook-roadmap.md)

## 2026-07-02 - Policy engine, detector YAML e prossimi step

Il lavoro recente si e' concentrato sul livello superiore alla raccolta eventi:
policy, detector dichiarativi e alert correlati.

E' stato consolidato il report sul policy engine di Tracee e sono stati
organizzati i prossimi passi del tool in una nuova cartella dedicata,
`next-steps/`. La direzione scelta e' userspace-first: prima policy e detector
YAML caricabili da file, poi eventuale kernel-side filtering minimo per ridurre
rumore e costo runtime.

E' stato inoltre avviato il primo passo implementativo nel tool: la
configurazione centrale ora prevede campi dedicati a policy, detector e alert
separati (`Policies`, `Detectors`, `Alerts`). La CLI espone ora le prime flag
per questi campi (`--policy`, `--detectors`, `--alerts`, `--alerts-output`).
La flag `--detectors` abilita automaticamente il motore detector quando riceve
almeno un path; il collegamento al runtime verra' fatto nei passi successivi.

Sono stati separati concettualmente:

- eventi singoli;
- policy di filtro;
- detector che generano alert;
- alert correlati costruiti da sequenze brevi di eventi.

**Note collegate:**

- [Diario dettagliato del giorno](daily/2026-07-02.md)
- [Tracee Policy Engine](implementation/tracee-policies.md)
- [Prossimi step del tool](next-steps/README.md)
- [Detector YAML e alert correlati](next-steps/detectors-and-correlations.md)
- [Ordine implementazione policy/detector](next-steps/policy-detector-implementation-order.md)

## 2026-07-24 - Ottimizzazione runtime e benchmark controllati

Le condizioni dei detector YAML vengono ora compilate durante la costruzione
del detector. Operatori, liste `in` e valori numerici attesi non vengono piu'
reinterpretati per ogni evento.

La pulizia dello stato dei detector collective e' diventata periodica: il
runtime mantiene il controllo puntuale della sequenza corrente, ma evita di
attraversare tutta la mappa a ogni record. Test e microbenchmark verificano
scadenza, `Flush()` forzato e costo stabile tra una e 1024 sequenze aperte.

Il prossimo intervento previsto e' la compilazione dei campi `group_by` e la
riduzione delle allocazioni in `groupKeys()`.

E' stato inoltre introdotto il primo pre-filtro kernel-side sull'effective UID
e la selezione policy-driven dei programmi eBPF. Le allowlist delle policy
raggiungono il registry prima del load: vengono caricati e attaccati soltanto
gli eventi richiesti e i probe interni risolti come dipendenze tramite
`impliedBy`.

La ring buffer e il relativo reader sono stati rimossi. Il perf buffer `events`
e' ora l'unico transport operativo.

La nuova suite benchmark automatizza avvio, PID target, workload, warm-up,
misure e shutdown per i profili `raw`, `point`, `collective` e
`kernel-filter-uid`. Nella run completa del 24 luglio tutte le medie userspace
sono rimaste sotto il 5% (`4.36%`, `0.33%`, `3.52%`, `2.92%`). Un test
`all-events`, invece, ha mantenuto circa `77-90%` ed e' stato interrotto: il
target prestazionale viene quindi valutato sui profili operativi policy-driven,
non sul caso limite con ogni hook pubblico contemporaneamente attivo.

**Note collegate:**

- [Diario dettagliato del giorno](daily/2026-07-24.md)
- [Misurazione prestazioni](implementation/performance.md)
- [Lifecycle userspace e autoload](implementation/userspace-lifecycle.md)
- [Detector YAML e alert correlati](next-steps/detectors-and-correlations.md)
