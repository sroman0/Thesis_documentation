# Project documentation

Questa cartella contiene la documentazione tecnica e operativa del tool eBPF.

Il punto di ingresso principale resta la [timeline](timeline.md), che collega i
diari giornalieri, le note di implementazione, le decisioni architetturali e il
materiale per la tesi.

Stato corrente: il tool ha una pipeline eBPF/userspace operativa, un layer
policy/detector YAML collegato al runtime, output separato per eventi e alert,
logging runtime con zap, dedup temporale degli alert ripetuti, primi detector
collective con correlazione locale `process`, `process_tree`, `resource` e
`cgroup`, chiavi composite, metadati MITRE visibili negli alert,
selezione policy-driven di autoload/attach, filtro UID kernel-side minimale e
suite benchmark multi-profilo. Le prossime attivita' principali riguardano
misura del rate per evento, costo kernel-side e ulteriori filtri derivati dalle
policy.

## Percorsi principali

- [Timeline della tesi](timeline.md)
- [Report tecnico corrente](report.md)
- [Overview implementazione](implementation/overview.md)
- [Hook implementati](implementation/hooks.md)
- [Roadmap hook process/security](implementation/hook-roadmap.md)
- [Tracee Policy Engine](implementation/tracee-policies.md)
- [Workflow runtime Tracee: policy e detector](implementation/tracee-detectors-policies-runtime.md)
- [Prossimi step del tool](next-steps/README.md)
- [Comandi utili](debugging/commands.md)
- [Comandi rapidi](useful_commands.md)

## Implementazione

- [Userspace lifecycle](implementation/userspace-lifecycle.md)
- [Event buffer](implementation/event-buffer.md)
- [Scelta del transport: perf buffer e ring buffer](implementation/perf-buffer-vs-ring-buffer.html)
- [Decoder](implementation/decoder.md)
- [Output](implementation/output.md)
- [Docker](implementation/docker.md)
- [Performance](implementation/performance.md)
- [Correlazione locale dei detector](implementation/correlation.md)
- [Sintesi ultimi hook implementati](implementation/recent-hooks-summary.md)

## Prossimi step

- [Roadmap tecnica](next-steps/roadmap.md)
- [Detector YAML e alert correlati](next-steps/detectors-and-correlations.md)
- [Piano di implementazione](next-steps/implementation-plan.md)
- [Ordine implementazione policy/detector](next-steps/policy-detector-implementation-order.md)
- [Proposte allineamento MITRE ATT&CK](next-steps/mitre-attack-alignment-proposals.md)
- [Piano logging strutturato con zap](next-steps/zap-logging-plan.md)

## Debugging e decisioni

- [Verifier eBPF](debugging/ebpf-verifier.md)
- [Decision log](decisions/decision-log.md)

## Materiale tesi e demo

- [Workspace documentale della tesi](thesis/README.md)
- [Indice canonico della tesi](thesis/definitive-outline.md)
- [Dossier preparatorio del Capitolo 1](thesis/chapters/chapter-01-introduction.md)
- [Sintesi editoriale del Capitolo 1](thesis/chapters/chapter-01-editorial-synthesis.md)
- [Workflow agenti del Capitolo 1](thesis/chapter-01-agent-workflow.md)
- [Terminologia e regole editoriali](thesis/terminology-and-style.md)
- [Mappa capitoli storica](thesis/thesis-outline.md)
- [Spunti scrittura](thesis/writing-notes.md)
- [Prima demo](demo/first_demo.md)
- [Discorso demo](demo/speech.md)
- [Materiale testuale per NotebookLM](presentation/README.md)

## Note sul tracciamento git

La cartella `tmp/` e' volutamente ignorata: contiene materiale temporaneo o
grezzo, come trascrizioni e appunti non rifiniti.

I documenti tecnici stabili devono invece essere collegati da questo README o
dalla timeline, in modo da non restare isolati.
