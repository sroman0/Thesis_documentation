# Verifica delle correzioni del professore

Confronto del 12 settembre 2026 tra `Thesis/Tesi_Romano_Simone_prof_review.pdf` (107 pagine) e `Thesis/main.pdf` (112 pagine, file datato 12 settembre 2026 ore 07:35).

## Metodo ed esito

Sono state estratte tutte le annotazioni del PDF commentato: **23 commenti testuali di Riccardo Sisto e una annotazione senza testo** sulla tabella dei requisiti funzionali. Le pagine indicate sotto sono le pagine fisiche PDF, contando dalla copertina. Nel corpo dei due documenti il numero stampato corrisponde alla pagina PDF meno 11.

Il confronto riguarda il testo realmente presente nel PDF aggiornato, non le proposte discusse in chat. Sono state ispezionate anche le figure del contratto binario, della build, dell'organizzazione dei sorgenti e dello scenario AES-beta. Non si tratta di una nuova verifica completa dell'implementazione o dei dati sperimentali.

Esito dopo la ricompilazione delle 07:48 UTC: **21 commenti risolti, 2 parzialmente risolti, nessun commento completamente aperto.** Il giudizio indica se la richiesta specifica trova risposta; non garantisce l'approvazione finale del professore.

Priorita prima dell'invio:

1. Ripristinare una descrizione dei limiti della correlazione nel capitolo 3 e correggere il rinvio alla sezione 3.1.3.
2. Correggere gli indici nel diagramma del contratto binario.
3. Completare il contesto metodologico del confronto Falco/Tetragon e precisare la media riportata in tabella.
4. Correggere i piccoli errori grammaticali e di punteggiatura indicati nelle note finali.

## Verifica commento per commento

### 1. Context and Motivation: descrizione del prototipo fuori contesto

- **Commento:** «Questa frase non ha molto senso in questa section che è di contesto, prima ancora di aver detto qual è il contesto.»
- **Vecchio PDF:** p. 13, sezione 1.1.
- **Nuovo PDF:** pp. 12-13, sezione 1.1.
- **Esito: risolto.** La frase che anticipava il funzionamento del prototipo e la detection locale e stata rimossa. La sezione termina con il contesto industriale e il vincolo di compatibilita. La soluzione viene introdotta successivamente.

### 2. Problem Statement: definizioni di point e collective

- **Commento:** «Manca una definizione precisa, ci sono solo esempi.»
- **Vecchio PDF:** p. 14, sezione 1.2.
- **Nuovo PDF:** pp. 13-14; definizioni a p. 43, sezione 3.6.2.
- **Esito: risolto.** La terminologia e stata eliminata dal problem statement. Nella sede progettuale un point detector valuta un evento; un collective detector valuta una sequenza ordinata di eventi correlati. Le definizioni precedono gli esempi e sono accompagnate da ordine di arrivo e finestra temporale.

### 3. Problem Statement: anticipazione del metodo di detection

- **Commento:** «Questa è la section del problem statement, non quella in cui si scrive come il problema è stato affrontato. Si attenga allo scope di ciascuna section.»
- **Vecchio PDF:** p. 14.
- **Nuovo PDF:** pp. 13-14, sezione 1.2.
- **Esito: risolto.** La discussione su regole deterministiche, baseline e anomaly score e stata rimossa. Il testo descrive esigenze dell'analista e difficolta di interpretazione delle osservazioni.

### 4. Problem Statement: separazione scope/detection e ATT&CK poco chiara

- **Commento:** «Non chiaro.»
- **Vecchio PDF:** p. 14, frase sulla modifica delle regole e sui metadati ATT&CK.
- **Nuovo PDF:** pp. 13-14 e 42-44.
- **Esito: risolto.** Il passaggio contestato non e piu nel problem statement. La sezione 3.6 spiega con esempi la funzione di policy e detector; la 3.6.4 descrive le etichette ATT&CK come classificazione dei detector, utilizzabile anche per selezionarli.

### 5. Problem Statement: problema e sfide non espliciti

- **Commento:** «Nuovamente ci sono aspetti impropriamente collocati nella section di problem statement. Arrivati alla fine della section il problema affrontato e le relative sfide non risultano chiari. Suggerisco di riscrivere questa section.»
- **Vecchio PDF:** p. 14.
- **Nuovo PDF:** pp. 13-14.
- **Esito: risolto.** La sezione e stata riscritta. Esplicita attribuzione a processi e risorse, distinzione tra tentativo e risultato, contesto delle operazioni, selezione dell'attivita e vincoli del sistema. L'ultimo paragrafo formula il problema e le condizioni da soddisfare.

### 6. Related Tools: scopo del confronto

- **Commento:** «Non è chiaro qual è lo scopo di questo confronto.»
- **Vecchio PDF:** p. 26, sezione 2.3.
- **Nuovo PDF:** p. 26, sezione 2.3.
- **Esito: risolto.** La sezione dichiara che il confronto serve a spiegare come i tool raccolgono attivita del kernel e identificano comportamenti rilevanti. Delimita inoltre il confronto a selezione degli eventi, detection e output, collegandolo al design del monitor.
- **Ritocco editoriale:** la citazione finale appare come «thesis. [19].». Va scritta senza spazio e senza doppia punteggiatura, per esempio `thesis~\cite{...}.`

### 7. Comparative Summary: spiegazione e conclusioni della tabella

- **Commento:** «Non è chiaro. La tabella va spiegata meglio e vanno tratte delle conclusioni sensate. Qui manca tutto questo.»
- **Vecchio PDF:** p. 28, sezione 2.3.3.
- **Nuovo PDF:** pp. 28-29, sezione 2.3.3 e tabella 2.3.
- **Esito: risolto.** Il testo ora spiega che la tabella riassume approccio e rilevanza di ciascun tool, indicando eventi, alert, filtri e risposta. Il paragrafo successivo ricava una conclusione distinta per Tracee, Falco e Tetragon e la collega al prototipo.

### 8. Positioning: contributo fumoso

- **Commento:** «E’ tutto estremamente fumoso e quindi inutile. va riscritto.»
- **Vecchio PDF:** p. 29, sezione 2.4.
- **Nuovo PDF:** p. 29, sezione 2.4.
- **Esito: risolto.** Il nuovo testo parte esplicitamente dai tre strumenti, identifica Tracee come riferimento architetturale e Falco/Tetragon come confronti prestazionali. Posiziona poi il lavoro sulla combinazione di osservazioni host per aiutare l'analista a ricostruire il comportamento di un processo.
- **Ritocco editoriale:** «A single operation may provide limited information, instead related observations...» non e grammaticalmente corretto. Usare due frasi, per esempio: `A single operation may provide limited information. Related observations can explain the behaviour more clearly.`

### 9. Functional Requirements: precisione e motivazioni

- **Commento:** «Questi requisiti sono molto di alto livello e poco precisi. P. es. FR-01: da quale insieme devono essere selezionabili gli eventi? Tutti o un sottoinsieme? Oppure, FR-03: quali sono i perf-event transport a cui fa riferimento? FR-04: non si capisce in che modo le policy dovrebbero vincolare gli eventi monitorati o il catalogo del detector. Servono più dettagli e più spiegazioni dei requisiti. Sarebbe importante anche discutere le motivazioni da cui ciascun requisito deriva.»
- **Vecchio PDF:** p. 32, tabella 3.1.
- **Nuovo PDF:** pp. 31-33, sezione 3.1.2.
- **Esito: risolto.** I sette requisiti sono spiegati e motivati singolarmente. FR-01 delimita la selezione al catalogo supportato; FR-03 chiarisce filtri su eventi/processi/UID/argomenti e selezione dei detector per identificatori o attributi. Il riferimento ambiguo ai trasporti e stato rimosso dal requisito e il trasporto e descritto nella sezione 3.4.2.
- L'annotazione senza testo a p. 32 del vecchio PDF riguarda la stessa tabella e non aggiunge una richiesta distinta.

### 10. Non-Functional Requirements: verifier, limiti e classificazione

- **Commento:** «Anche questi requisiti andrebbero motivati dpiegati e dettagliati meglio. NFR-02: a quale target verifier sta facendo riferimento? Quello di eBPF? Quanto è il bound? NFR-03: non capisco perchè è stato classificato come non funzionale. NFR-05: a quale stato esattamente fa riferimento e quando è il bound?»
- **Vecchio PDF:** p. 33.
- **Nuovo PDF:** pp. 33-34, sezione 3.1.3; p. 44, sezione 3.6.3; p. 78, tabella 5.1.
- **Esito: parzialmente risolto.** Sono presenti definizione, motivazioni, riferimento esplicito al verifier eBPF e limiti concreti: 20 componenti del percorso e 32.000 byte per gli argomenti. Il vecchio NFR-03 e stato eliminato dalla tabella non funzionale.
- **Residuo:** il requisito sullo stato collettivo e stato rimosso, ma la sezione 3.6.3 continua a dire che tempo e capacita sono indicati nella 3.1.3. Non lo sono. I valori compaiono solo nella tabella 5.1: finestra predefinita 2 s, massima 5 s, capacita 4.096 chiavi per detector.
- **Correzione:** aggiungere un requisito motivato sullo stato delle sequenze incomplete con quei limiti, oppure descriverli in 3.6.3 e correggere il rinvio. Chiarire che la scadenza logica non implica la rimozione immediata delle entry durante periodi senza eventi, come gia spiegato nel capitolo 5.

### 11. Hook Selection: attachment point frequenti

- **Commento:** «Non chiaro cosa voglia dire, da sviluppare meglio.»
- **Vecchio PDF:** p. 35, frase «avoid unnecessarily frequent attachment points».
- **Nuovo PDF:** p. 36, sezione 3.3.1.
- **Esito: risolto.** Il testo spiega che un hook attraversato da operazioni estranee richiede controlli ripetuti, e motiva la preferenza per punti specifici dell'operazione osservata.

### 12. Event Contract: progetto concreto ed esempio

- **Commento:** «Troppo generico, non viene documentato come stato progettato il contratto. Suggerirei anche di presentare un esempio per dare concretezza.»
- **Vecchio PDF:** p. 37, sezione 3.4.
- **Nuovo PDF:** pp. 38-39, sezione 3.4.1 e figura 3.2.
- **Esito: parzialmente risolto.** Il testo ora documenta contesto, conteggio, indici, schema, tipi, argomenti mancanti e accordo producer/decoder. L'esempio security_file_open risponde alla richiesta di concretezza.
- **Residuo verificato visivamente:** la figura assegna a dev e inode gli indici 1 e 2; il testo assegna 2 e 3. La figura non rappresenta quindi lo stesso schema descritto nel paragrafo.
- **Correzione:** aggiornare entrambi i blocchi della figura agli indici 0, 2, 3. Mantenere esplicito che gli altri argomenti sono omessi. Rendere le etichette leggermente piu grandi aiuterebbe la lettura.

### 13. Perf Buffer: gestione dei ritmi diversi

- **Commento:** «Non è chiaro se e come questo problema sia stato affrontato».
- **Vecchio PDF:** p. 38.
- **Nuovo PDF:** p. 40, sezione 3.4.2.
- **Esito: risolto.** La sezione presenta il problema, il reader separato dalla elaborazione, la coda, la raccolta selettiva, il filtro UID e i record compatti. Spiega anche saturazione prolungata, blocco del reader e diagnostiche. Le mitigazioni sono descritte senza promettere assenza di perdite.

### 14. Policy and Detection: esempio di piano

- **Commento:** «Anche qui il livello di dettaglio non è sufficiente. Aggiunga esempi concreti (per esempio un esempio di piano)».
- **Vecchio PDF:** p. 40.
- **Nuovo PDF:** pp. 42-44, sezione 3.6.
- **Esito: risolto.** La tabella 3.4 mostra selezione iniziale di tre eventi, selezione della policy, due eventi raccolti, UID e detector attivo. I paragrafi spiegano cosa viene disabilitato e dove si applica il filtro. Sono presenti anche esempi point e collective. Il rinvio errato sui limiti e gia registrato al commento 10.

### 15. Build e organizzazione: diagrammi

- **Commento:** «Suggerirei di presentare la build chain e l’organizzazione tramite diagrammi in modo che sia più chiara.»
- **Vecchio PDF:** p. 43.
- **Nuovo PDF:** pp. 45-47, sezione 4.1.
- **Esito: risolto.** Le figure 4.1 e 4.2 mostrano build e gruppi di sorgenti. Sono richiamate e spiegate nel testo. La figura della build distingue linking di libbpf e incorporamento dell'oggetto eBPF. Le immagini sono presenti e leggibili, anche se alcune etichette della prima sono piccole.

### 16. Output and Structured Logging: esempi

- **Commento:** «Anche qui esempi concreti potrebbero aiutare a dare concretezza e far capire meglio.»
- **Vecchio PDF:** p. 76.
- **Nuovo PDF:** pp. 79-80, sezione 5.6.
- **Esito: risolto.** Tre listing mostrano evento, alert con threat metadata e diagnostica dalla stessa sessione. Il testo spiega origine, campi omessi, etichette e canali. Distingue correttamente l'evento open mostrato dall'osservazione security_file_open che ha attivato il detector.

### 17. Testbed: scopo e contenuto della tabella

- **Commento:** «Bisogna essere più chiari. Spieghi a cosa serve la tabella e cosa contiene.»
- **Vecchio PDF:** p. 80.
- **Nuovo PDF:** pp. 84-85, sezione 6.3.
- **Esito: risolto.** Il testo presenta host, monitor, target e trigger, spiega i ruoli e anticipa deployment, identita e restrizioni. Collega i dati alla distinzione tra accesso alle interfacce host e permessi delle applicazioni.

### 18. Testbed: motivazione delle scelte operative

- **Commento:** «Bisogna spiegare il perchè delle scelte, non solo elencarle.»
- **Vecchio PDF:** p. 81.
- **Nuovo PDF:** p. 85.
- **Esito: risolto.** UID diversi sono motivati con l'attribuzione dell'attivita; privilegi ridotti con le esigenze applicative; token disabilitati con l'assenza di accesso API; requests con scheduling; assenza di limits con osservazione dei consumi senza un tetto aggiuntivo. La conseguente contesa e dichiarata.

### 19. AES-beta: riferimento e caption della figura

- **Commento:** «Ogni figura deve avere un riferimento nel testo. Conviene accorciare la caption e descrivere il contenuto nel testo.»
- **Vecchio PDF:** p. 83.
- **Nuovo PDF:** pp. 86-87, figura 6.3.
- **Esito: risolto per la figura segnalata.** Il richiamo precede la figura, la caption e breve e il testo descrive pannelli sinistro e destro, leak, canary, payload e accesso alla flag. Le immagini affiancate sono effettivamente presenti. Alcune scritte sono piccole: ingrandirle sarebbe un miglioramento grafico, non una mancata risposta alla richiesta specifica. Questo esito non costituisce una verifica di tutti i richiami alle figure dell'intera tesi.

### 20. Discussion: visibilita cross-Pod attesa

- **Commento:** «Questo risultato non è affatto sorprendente ed era abbastanza scontato. Spiegare perchè. Oppure, se per qualche ragione pensate che non lo fosse, dire perchè.»
- **Vecchio PDF:** p. 89.
- **Nuovo PDF:** p. 94, sezione 6.7.
- **Esito: risolto.** Il testo dichiara che la visibilita era attesa perche i Pod condividono il kernel dello stesso nodo. Il risultato viene presentato come verifica di deployment, raccolta e analisi.
- **Ritocco editoriale:** sostituire la virgola in «supported this deployment, the resulting alert» con un punto e aggiungere il punto finale dopo «four configured observations». La frase sull'alert e ripetuta anche nel paragrafo successivo.

### 21. Discussion: confronto sperimentale con altri tool

- **Commento:** «Non ci sono confronti con altri tool di monitoring. Aggiungerli darebbe più valore alla tesi.»
- **Vecchio PDF:** p. 89.
- **Nuovo PDF:** pp. 94-96, sezione 6.7.1 e tabella 6.4.
- **Esito: risolto.** Il confronto con Falco e Tetragon e presente e comprende versioni, tre run per profilo, CPU, risultati funzionali e differente collocazione dei filtri. Risponde direttamente alla richiesta di confrontare il prototipo con altri monitor.
- **Residui:** l'introduzione e il protocollo del capitolo descrivono soltanto le campagne del prototipo; la tabella non dice che riporta la media delle tre medie di run; per i tool manca l'indicazione dell'ordine variato. Non sono esplicitate qui le differenze di output ed enrichment: Tetragon esporta su file con API Kubernetes disabilitata, Falco include metadata di container/Pod. La sezione 6.8 menziona gia il diverso riavvio del target.
- **Correzione:** aggiungere poche frasi su configurazioni e misura, precisare l'aggregazione della tabella e includere questi fattori tra i limiti del confronto. Tenere il costo aggiuntivo del controllo UID come ipotesi: i dati non isolano il costo dei singoli stadi, e il filtro Tetragon e nel kernel mentre la metrica e cgroup CPU.

### 22. Threats to Validity: come affrontare i limiti

- **Commento:** «Sarebbe utile dire anche come queste limitazioni si potrebbero superare.»
- **Vecchio PDF:** p. 90.
- **Nuovo PDF:** p. 96, sezione 6.8, rinominata Current Limitations.
- **Esito: risolto per i limiti discussi.** Ognuno dei tre paragrafi propone azioni: nodo dedicato e avvio uniforme; altri scenari/carichi; piu run con ordine randomizzato.
- **Completezza da migliorare:** rispetto alla versione precedente sono stati eliminati alcuni limiti di misurazione. Conviene aggiungere una breve nota su overhead totale non misurato, completezza della raccolta non verificata e confronto fra configurazioni diverse, con relativi test futuri. Alcuni di questi vincoli sono gia dichiarati altrove nel capitolo, ma qui manca il collegamento con come valutarli.

### 23. Main Results: conclusioni semplici e meno dettagli specifici

- **Commento:** «Cerchi di evitare troppi riferimenti ad elementi specifici del progetto non essenziali. Vada diretto alle considerazioni che vuole fare in modo semplice.»
- **Vecchio PDF:** p. 92.
- **Nuovo PDF:** pp. 98-99, sezione 7.2.
- **Esito: risolto.** La sezione ora tratta utilita per l'analista, monitoraggio semi-automatizzato, supporto all'indagine, configurabilita e rapporto tra copertura e costo. Non ripete il dettaglio di Pod, flag, percentuali e cgroup della vecchia versione.
- **Ritocco consigliato:** l'introduzione del capitolo e la sezione 7.1 conservano il lessico tecnico precedente. Non riaprono il commento specifico sulla 7.2, ma semplificarle renderebbe piu uniforme il capitolo. Una frase di conferma sperimentale nella 7.2 collegherebbe meglio le considerazioni ai risultati.

## Note finali sul PDF controllato

Le revisioni concordate per il capitolo 2 risultano ora nel PDF e chiudono i tre commenti relativi allo scopo del confronto, alla spiegazione della tabella e al posizionamento del lavoro.

Le correzioni su requisiti, diagrammi, esempi di output, testbed e scenario sono ampiamente presenti. Prima dell'invio restano tre interventi sostanziali circoscritti: il rinvio ai limiti della correlazione, gli indici della figura 3.2 e il contesto metodologico del confronto Falco/Tetragon. Vanno inoltre corretti la punteggiatura della discussione a pagina PDF 94 e i due errori editoriali del capitolo 2 indicati sopra.
