# Materiale strutturato per la tesi

Questa cartella e' il livello intermedio tra la documentazione tecnica del
progetto e il repository LaTeX `Thesis`.

Il workflow concordato e':

```text
codice + documentazione tecnica + fonti esterne
  -> dossier del capitolo
  -> verifica tecnica e bibliografica
  -> scrittura nel repository Thesis
  -> revisione e compilazione PDF
```

Il repository `Thesis` deve contenere testo accademico gia' consolidato. Il
materiale ancora incompleto, le fonti da verificare e i dubbi di scope restano
qui fino alla loro risoluzione.

## Documenti canonici

- [Indice definitivo e confini dei capitoli](definitive-outline.md)
- [Terminologia e regole editoriali](terminology-and-style.md)
- [Dossier preparatorio del Capitolo 1](chapters/chapter-01-introduction.md)
- [Sintesi editoriale vincolante del Capitolo 1](chapters/chapter-01-editorial-synthesis.md)
- [Workflow e prompt degli agenti per il Capitolo 1](chapter-01-agent-workflow.md)
- [Dossier preparatorio del Capitolo 2](chapters/chapter-02-background-related-work.md)
- [Workflow e prompt degli agenti per il Capitolo 2](chapter-02-agent-workflow.md)
- [Dossier preparatorio del Capitolo 3](chapters/chapter-03-requirements-system-design.md)
- [Workflow e prompt degli agenti per il Capitolo 3](chapter-03-agent-workflow.md)
- [Dossier preparatorio del Capitolo 4](chapters/chapter-04-system-implementation.html)
- [Workflow e prompt degli agenti per il Capitolo 4](chapter-04-agent-workflow.html)
- [Output intermedi degli agenti per il Capitolo 4](agent-output/chapter-04/index.html)
- [Spunti di scrittura storici](writing-notes.md)

## Regole operative

- La tesi viene scritta in ordine, dal Capitolo 1 al Capitolo 6.
- Prima di modificare un capitolo LaTeX deve esistere il relativo dossier.
- Gli agenti di ricerca producono evidenze e proposte, non modificano tutti lo
  stesso file `.tex`.
- Un solo agente editoriale integra il testo finale del capitolo.
- Ogni affermazione tecnica deve essere supportata dal codice, dalla
  documentazione del progetto o da una fonte bibliografica verificata.
- I contributi implementativi vanno distinti dalle novelty scientifiche, che
  richiedono confronto con lo stato dell'arte e una valutazione esplicita.
- L'abstract resta invariato fino alla conclusione degli altri capitoli.
