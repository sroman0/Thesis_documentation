# Proposte Per Allineamento MITRE ATT&CK

Questo documento raccoglie le proposte da discutere nella prossima call tecnica
prima di implementare in modo stabile il supporto MITRE ATT&CK nel sistema di
policy e detector.

L'obiettivo e' evitare di costruire policy e detector come semplici filtri
tecnici. Il tool dovrebbe poter dire non solo "quale evento e' successo", ma
anche "quale comportamento avversario rappresenta" secondo il framework MITRE
ATT&CK.

## Stato Attuale

Una parte della proposta e' stata implementata nella versione corrente del
tool:

- i detector YAML possono dichiarare un blocco `threat`;
- gli ID di tattiche e tecniche ATT&CK vengono validati sintatticamente;
- i metadati MITRE vengono propagati nel modello `detectors.Alert`;
- l'output table degli alert mostra un campo compatto `mitre=...`;
- l'output JSON degli alert espone il blocco strutturato `threat`;
- i detector senza mapping MITRE restano accettati, ma sono considerati meno
  adatti a report professionali e demo threat-informed.

Restano invece da implementare:

- selezione di policy o detector direttamente per tactic/technique;
- report automatico di copertura ATT&CK;
- validazione degli ID contro un dataset MITRE locale;
- eventuale comando CLI dedicato alla vista di coverage.

## Punto Di Partenza

La struttura attuale e' gia' utile:

- le policy selezionano eventi;
- i detector ricevono eventi decodificati;
- i detector possono diventare stateful e correlare eventi ravvicinati;
- gli alert saranno separati dagli eventi grezzi.

La parte detector e' ora esplicita: i detector possono dichiarare tattiche,
tecniche e data source MITRE ATT&CK. La parte ancora mancante e' usare questi
metadati come criterio operativo di selezione e reporting, non solo come
informazione allegata all'alert.

## Proposta 1: Aggiungere Metadati MITRE Nei Detector

La proposta principale e' inserire i metadati MITRE nella definizione del
detector, non solo nella policy.

Motivo: il detector e' il componente che rappresenta una logica di detection.
Quindi e' il punto piu' corretto per dichiarare quale tecnica ATT&CK viene
coperta.

Esempio di modello Go:

```go
type ThreatMetadata struct {
    Framework      string
    Version        string
    Tactics        []AttackTactic
    Techniques     []AttackTechnique
    DataSources    []string
    DataComponents []string
}

type AttackTactic struct {
    ID   string
    Name string
}

type AttackTechnique struct {
    ID           string
    Name         string
    SubTechnique string
}
```

Esempio pratico:

```go
ThreatMetadata{
    Framework: "MITRE ATT&CK Enterprise",
    Version: "v19.1",
    Tactics: []AttackTactic{
        {ID: "TA0004", Name: "Privilege Escalation"},
    },
    Techniques: []AttackTechnique{
        {ID: "T1548.001", Name: "Setuid and Setgid"},
    },
    DataSources: []string{"Process"},
    DataComponents: []string{"Process Creation", "User Account Metadata"},
}
```

## Proposta 2: Rendere Le Policy Capaci Di Selezionare Per MITRE

Le policy non dovrebbero solo selezionare eventi come `execve` o
`security_file_open`. Dovrebbero poter selezionare anche detector o alert per
tattica e tecnica ATT&CK.

Esempio YAML futuro:

```yaml
name: privilege-escalation-monitoring
mode: detect
intent: threat_hunting
threat:
  framework: MITRE ATT&CK Enterprise
  tactics:
    - TA0004
  techniques:
    - T1548
    - T1548.001
rules:
  - name: privilege-events
    events:
      include:
        - security_task_fix_setuid
        - execve
```

In questo modo una policy puo' dire:

```text
abilita tutto cio' che riguarda privilege escalation
```

senza dover conoscere ogni detector nel dettaglio.

## Proposta 3: Definire Un Formato YAML Dei Detector MITRE-Aware

Quando arriveremo allo step dei detector YAML, il formato dovrebbe includere un
blocco `threat`.

Esempio:

```yaml
id: privilege-change-then-exec
name: Privilege change followed by execution
description: Detects a short sequence where a process changes identity and then executes a binary.
requires:
  - security_task_fix_setuid
  - execve
stateful: true
window: 2s
threat:
  framework: MITRE ATT&CK Enterprise
  version: v19.1
  tactics:
    - id: TA0004
      name: Privilege Escalation
  techniques:
    - id: T1548.001
      name: Setuid and Setgid
  data_sources:
    - Process
  data_components:
    - Process Creation
    - User Account Metadata
output:
  severity: high
  title: Privilege change followed by execution
```

Questa struttura rende i detector piu' facili da documentare, confrontare e
presentare come copertura ATT&CK.

## Proposta 4: Usare MITRE Per Misurare La Copertura

Una volta associati detector e hook a tattiche/tecniche, il tool potrebbe
produrre una vista di copertura:

```text
TA0002 Execution             covered by: execve, execveat, security_bprm_check
TA0003 Persistence           partially covered by: chmod, chown, file_open
TA0004 Privilege Escalation  covered by: setuid/setgid hooks, cap_capable
TA0005 Defense Evasion       covered by: module_load, ptrace, chmod/chown
```

Questa vista non e' detection runtime, ma e' utile per presentare il valore del
progetto, spiegare quali aree ATT&CK sono coperte e scegliere i prossimi hook in
modo meno arbitrario.

## Proposta 5: Validazione Minima Degli ID ATT&CK

Prima di introdurre integrazione automatica con dataset MITRE, possiamo fare
una validazione leggera:

- tactic ID in formato `TA0000`;
- technique ID in formato `T0000`;
- sub-technique ID in formato `T0000.000`;
- framework e versione come campi espliciti;
- detector senza metadata MITRE accettati ma marcati come `unmapped`.

Questo evita di rendere subito il progetto dipendente da download o dataset
esterni.

## Proposta 6: Integrazione Futura Con Dataset MITRE

In una fase successiva si potrebbe usare il dataset MITRE ATT&CK in formato
STIX o JSON per validare davvero tattiche, tecniche e relazioni.

Prima versione:

```text
validazione formato ID
```

Versione successiva:

```text
validazione ID contro dataset MITRE locale
```

Questo permetterebbe anche di generare report automatici di copertura e di
segnalare detector con metadata obsoleti.

## Impatto Sul Piano Di Implementazione

Gli step piu' impattati sono:

- `pkg/detectors/definition.go`
- `pkg/detectors/yaml/schema.go`
- `pkg/detectors/yaml/parser.go`
- `pkg/policy/policy.go`
- `pkg/policy/loader.go`
- `pkg/policy/manager.go`
- `pkg/output/alert.go`
- documentazione e report finale

La modifica iniziale su `pkg/detectors/definition.go` e' stata completata.
Le prossime modifiche piu' importanti riguardano `pkg/policy`, per permettere
selezione threat-informed, e un eventuale comando/report di coverage.

## Domande Da Portare In Call

1. Vogliamo usare MITRE ATT&CK Enterprise come riferimento ufficiale del tool?
2. Serve supportare solo Linux oppure vogliamo lasciare il modello aperto anche
   ad altri domini ATT&CK in futuro?
3. I detector devono avere obbligatoriamente una tecnica MITRE oppure possiamo
   accettare detector `unmapped`?
4. La policy deve selezionare per event name, per detector ID, per tactic,
   per technique, oppure per tutte queste dimensioni?
5. Quale versione ATT&CK vogliamo dichiarare come riferimento iniziale?
6. Vogliamo una validazione solo sintattica degli ID o una validazione contro
   un dataset MITRE locale?
7. Gli alert devono mostrare sempre tactic/technique MITRE in output?
8. Serve una vista di coverage ATT&CK generabile da CLI?
9. Come gestiamo detector che coprono piu' tecniche contemporaneamente?
10. La correlazione stateful deve essere collegata esplicitamente a tecniche
    MITRE oppure puo' restare una feature indipendente?

## Raccomandazione Tecnica

La raccomandazione aggiornata e' continuare MITRE in modo progressivo:

1. mantenere `ThreatMetadata` nei detector YAML come sorgente principale;
2. permettere alle policy di selezionare detector per tactic/technique;
3. generare una vista di coverage ATT&CK dai detector disponibili;
4. introdurre, solo dopo, validazione contro dataset MITRE locale.

In questo modo il tool resta semplice da sviluppare, ma acquisisce subito un
valore professionale piu' forte: ogni detection puo' essere spiegata usando un
linguaggio standard e riconosciuto.
