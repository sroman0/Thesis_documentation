# Prossimi step del tool

Questa cartella raccoglie i prossimi passi tecnici per evolvere il tool oltre la
semplice raccolta di eventi eBPF.

Il focus non e' aggiungere altri hook in modo incrementale, ma costruire un
livello superiore capace di:

- selezionare eventi tramite policy dichiarative;
- generare alert singoli quando un evento e' rilevante da solo;
- correlare piu' eventi in alert comportamentali;
- mantenere output separato tra eventi raw/operativi e alert correlati;
- ridurre rumore e costo runtime senza perdere segnali utili.

## Ordine consigliato

1. [Roadmap tecnica](roadmap.md)
2. [Detector YAML e alert correlati](detectors-and-correlations.md)
3. [Piano di implementazione](implementation-plan.md)

## Idea centrale

La direzione piu' solida e' separare tre livelli:

```text
event collection
  -> point policies
  -> detector/correlation engine
  -> alert output
```

Il primo livello raccoglie e decodifica eventi. Il secondo decide quali eventi
singoli meritano attenzione. Il terzo costruisce alert piu' ricchi quando piu'
eventi, presi insieme, descrivono un comportamento sospetto.

