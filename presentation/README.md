# Materiale testuale per NotebookLM

Questa cartella contiene file Markdown pensati per sostituire i file di codice
quando si prepara una bozza di slide con NotebookLM.

NotebookLM puo' usare meglio documenti narrativi, riassunti e tabelle rispetto a
file sorgente C/Go molto lunghi. Per questo motivo questi file spiegano il tool
senza richiedere di allegare direttamente il codice.

## File da allegare in NotebookLM

Per preparare una versione beta delle slide sulla parte condivisa e sulla parte
di Simone, allegare prima questi file:

- `documentation/Shattered-Bytes Presentation - Romano Simone.pdf`
- `documentation/presentation/tracee-rocky-linux-context.md`
- `documentation/presentation/tool-architecture-for-slides.md`
- `documentation/presentation/hooks-for-slides.md`
- `documentation/presentation/event-decoder-for-slides.md`
- `documentation/presentation/policy-detectors-for-slides.md`
- `documentation/report.md`
- `documentation/timeline.md`
- `documentation/implementation/hooks.md`
- `documentation/implementation/recent-hooks-summary.md`
- `documentation/next-steps/implementation-plan.md`
- `documentation/next-steps/policy-detector-implementation-order.md`

Se NotebookLM limita il numero di sorgenti, usare questa lista ridotta:

- `documentation/Shattered-Bytes Presentation - Romano Simone.pdf`
- `documentation/presentation/tracee-rocky-linux-context.md`
- `documentation/presentation/tool-architecture-for-slides.md`
- `documentation/presentation/hooks-for-slides.md`
- `documentation/presentation/policy-detectors-for-slides.md`
- `documentation/report.md`
- `documentation/next-steps/implementation-plan.md`

## Uso consigliato

Nel prompt per NotebookLM specificare che questi file sono riassunti testuali
dei componenti implementati e che devono essere trattati come fonte principale
per produrre slide semplici, minimali e coerenti con il PDF di riferimento.

