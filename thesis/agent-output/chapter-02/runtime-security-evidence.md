# Runtime Security and Detection Theory Evidence for Chapter 2

## Purpose and Scope

This dossier supports the following parts of Chapter 2:

- 2.3.1 Point, Contextual and Collective Anomalies;
- 2.3.2 Policies, Detectors and Alert Generation;
- 2.3.3 MITRE ATT&CK as a Classification Framework.

The chapter should introduce the academic taxonomy of anomalies, but it must
also distinguish that taxonomy from the detection method implemented by the
prototype. The current system evaluates deterministic conditions over
normalised runtime events. Some detectors operate on one event, while others
match a short, ordered and process-aware sequence. The system does not learn a
model of normal behaviour, calculate an anomaly score, or claim to discover
previously unknown anomalies.

This distinction is important but does not weaken the contribution. It gives
the system a precise position: it uses an anomaly taxonomy to reason about the
structure of relevant behaviours, while implementing explainable rule-based
detection and bounded temporal correlation.

## Terminology

The terms `event`, `policy`, `detector`, and `alert` below are operational
definitions for this thesis. They are compatible with common security
monitoring terminology, but they should not be presented as a universal
standard.

| Term | Recommended definition | Theoretical basis | Boundary to preserve |
|---|---|---|---|
| Event | A normalised record of an observed runtime occurrence, including its type, process context, timestamp, and decoded arguments when available. | NIST describes intrusion detection as monitoring and analysing events; event-stream literature treats events as timestamped information items processed as they arrive. | An event is an observation, not by itself evidence of malicious intent. |
| Policy | A declarative specification that selects the event domain and detector set relevant to a monitoring objective. | NIST notes that security policies and configuration determine which activities an IDPS should monitor and report. | In this thesis, a policy scopes monitoring; it is not an enforcement policy and does not itself identify an attack. |
| Detector | A deterministic evaluator that applies declared conditions to one normalised event or to a bounded ordered sequence of related events. | Signature-based detection compares known patterns with observed events; event-stream systems evaluate declared patterns over flows of events. | The detector is rule-based. It does not learn normal behaviour or perform general anomaly discovery. |
| Alert | A structured output emitted when a detector condition is satisfied, carrying the matched evidence and classification metadata. | NIST describes IDPS technologies as notifying administrators about events considered significant and producing supporting reports. | An alert reports a rule match. It is not automatically a confirmed incident. |
| Point anomaly | An individual data instance that is anomalous relative to the remainder of the data or to an expected notion of normal behaviour. | Chandola et al., 2009. | A single-event rule match is only structurally analogous to a point anomaly unless a normality model is actually evaluated. |
| Contextual anomaly | An instance that is anomalous in a particular context but may be normal in another context. Contextual and behavioural attributes must therefore be distinguished. | Chandola et al., 2009. | Local process metadata is useful context, but the prototype does not implement general contextual anomaly detection or cluster-wide behavioural baselines. |
| Collective anomaly | A related collection of instances whose joint occurrence is anomalous even when its members may not be anomalous individually. | Chandola et al., 2009. | A declared two-step or short sequence represents one known collective pattern; it does not discover arbitrary anomalous groups. |

## Recommended Narrative

### 2.3.1 Point, Contextual and Collective Anomalies

The subsection should begin from the general anomaly-detection problem.
Chandola et al. define anomalies as patterns that do not conform to expected
behaviour and distinguish point, contextual, and collective forms. This
taxonomy concerns the structure of the observation judged anomalous:

- a point anomaly concerns one instance;
- a contextual anomaly depends on the context in which an instance occurs;
- a collective anomaly concerns a related group whose joint occurrence is
  significant.

The categories are not necessarily disjoint. Chandola et al. note that point
or collective anomalies may also be contextual when additional context is
included. Chapter 2 should therefore present them as analytical perspectives,
not as three mutually exclusive detector classes.

The transition to runtime security should be explicit. A security-relevant
event can be suspicious in isolation, such as an unexpected privilege change,
or only as part of an ordered sequence, such as a privilege transition
followed by execution. A contextual judgement would instead require evidence
that the behaviour is unusual for a particular user, workload, time, host, or
cluster state.

The subsection must then separate statistical or machine-learning anomaly
detection from deterministic detection:

- anomaly-based systems ordinarily compare current activity with a model or
  profile of expected behaviour;
- deterministic detectors compare observations with explicit conditions or
  declared event patterns;
- the prototype belongs to the second category.

The words *point* and *collective* may still describe the structural scope of
the implemented rules. A precise formulation is:

> The prototype implements deterministic detections over individual events
> and bounded ordered event sequences. These rules represent behaviours with
> point-like and collective structure, but they do not constitute a general
> statistical anomaly-discovery mechanism.

This wording connects the implementation to the academic taxonomy without
claiming that a normal model, anomaly score, or unsupervised learner exists.

### 2.3.2 Policies, Detectors and Alert Generation

This subsection should explain why runtime telemetry is separated from
detection logic. Kernel instrumentation produces observations; user space
normalises those observations into events; policies choose which event domain
and detectors are relevant; detectors evaluate explicit conditions; and
successful matches generate alerts.

The recommended conceptual pipeline is:

```text
runtime occurrence
    -> collected and normalised event
    -> policy-defined scope
    -> detector evaluation
    -> alert with supporting evidence
```

This separation has three defensible benefits:

1. collection and detection logic can evolve independently;
2. monitoring can be scoped to a concrete security objective;
3. the reason for an alert remains inspectable because the matched event or
   sequence is retained as evidence.

NIST distinguishes signature-based detection, which compares known patterns
with observed events, from anomaly-based detection, which compares observed
activity with a profile of normal behaviour. The prototype is closer to the
former, although its detector language can also express short stateful
patterns rather than only independent signatures.

Event-stream research provides the appropriate abstraction for those
multi-event detectors. A detector may specify:

- an ordered list of event types;
- predicates over event attributes;
- a finite temporal window;
- a grouping or relationship key, such as process identity;
- an alert to emit when the complete pattern matches.

This is a bounded temporal query over an event stream. It should not be
described as inferring causality: temporal order and shared process identity
provide correlation evidence, but do not prove that one operation caused the
next.

#### Approved description of a bounded sequence

> A collective detector evaluates a predefined ordered pattern over a
> normalised event stream. Candidate partial matches are retained only for a
> finite time window and are correlated through an explicit process
> relationship. An alert is generated when the declared sequence is
> completed.

#### Claims to avoid

- "The system automatically discovers collective anomalies."
- "The detector learns normal process behaviour."
- "The sequence proves an attack chain."
- "Two events with the same process identifier are causally related."
- "The system performs general complex-event processing."

The implementation supports a constrained subset of temporal pattern
matching. Calling it a bounded event-sequence detector is more accurate than
claiming a general-purpose complex-event processing engine.

### 2.3.3 MITRE ATT&CK as a Classification Framework

MITRE ATT&CK should be introduced as a knowledge base and structured model of
observed adversary behaviour. Its tactics express the objective behind an
adversary action, while techniques express how that objective may be achieved;
sub-techniques provide a more specific behavioural description.

For this thesis, ATT&CK metadata has three principal values:

- it gives alerts a security vocabulary shared beyond the implementation;
- it connects low-level runtime evidence to recognisable adversary
  objectives and behaviours;
- it makes detector catalogues and alert output easier to compare, review,
  exchange, and integrate with downstream security workflows.

The mapping should be framed as semantic enrichment rather than as an
evaluation result. A positive and accurate formulation is:

> Associating detector output with MITRE ATT&CK tactics and techniques anchors
> low-level runtime evidence in a widely used behavioural taxonomy. This
> improves the interpretability and portability of alerts and provides a
> consistent basis for reviewing the security behaviours represented by the
> detector catalogue.

The chapter may then add one short boundary:

> ATT&CK coverage is assessed at the level of concrete detector behaviours;
> a metadata association is not treated as evidence that every procedure
> covered by a technique can be detected.

This is not dismissive. It follows MITRE's own guidance that one observed
implementation of a technique should not be treated as complete coverage.

## Claim-to-Source Matrix

### Claims for 2.3.1

| ID | Cautious academic formulation | Source | Supported meaning | Limitations |
|---|---|---|---|---|
| A1 | Anomaly detection is commonly formulated as identifying patterns that do not conform to an expected notion of behaviour. | Chandola, Banerjee and Kumar, 2009. DOI: [10.1145/1541880.1541882](https://doi.org/10.1145/1541880.1541882). | Provides the general anomaly-detection definition and survey context. | It does not imply that every security rule is an anomaly detector. |
| A2 | A point anomaly concerns an individual instance that is anomalous relative to the rest of the data. | Chandola et al., 2009, Section 2.2.1. DOI: [10.1145/1541880.1541882](https://doi.org/10.1145/1541880.1541882). | Supports the point-anomaly definition. | A single-event deterministic match is only point-like unless compared with expected behaviour. |
| A3 | A contextual anomaly is abnormal in a particular context while the same behavioural value may be normal in another context. | Chandola et al., 2009, Section 2.2.2. DOI: [10.1145/1541880.1541882](https://doi.org/10.1145/1541880.1541882). | Supports the distinction between contextual and behavioural attributes. | Context must be explicitly available and modelled; a process identifier alone is not a general context model. |
| A4 | A collective anomaly is a related set of instances whose joint occurrence is anomalous even when its members need not be anomalous independently. | Chandola et al., 2009, Section 2.2.3. DOI: [10.1145/1541880.1541882](https://doi.org/10.1145/1541880.1541882). | Supports the collective-anomaly definition and its applicability to sequences. | It does not prescribe the prototype's sequence algorithm or show that every declared sequence is empirically anomalous. |
| A5 | Point, contextual, and collective anomaly categories describe different analytical structures and may overlap when context is added. | Chandola et al., 2009, Section 2.2.3. DOI: [10.1145/1541880.1541882](https://doi.org/10.1145/1541880.1541882). | Supports the statement that point or collective anomalies can also be contextual. | Do not present the taxonomy as mutually exclusive implementation classes. |
| A6 | In conventional anomaly-based intrusion detection, observations are compared with profiles or definitions of normal activity, whereas signature-based detection compares observations with known patterns. | Scarfone and Mell, 2007, Sections 2.3.1-2.3.2. DOI: [10.6028/NIST.SP.800-94](https://doi.org/10.6028/NIST.SP.800-94). | Provides an authoritative distinction between anomaly-based and signature-based detection. | NIST SP 800-94 is guidance rather than a peer-reviewed research paper, and terminology varies across publications. |
| A7 | Machine-learning intrusion detection faces operational challenges when training and deployment conditions differ, so results obtained in a closed experimental setting do not automatically transfer to live monitoring. | Sommer and Paxson, 2010. DOI: [10.1109/SP.2010.25](https://doi.org/10.1109/SP.2010.25). | Supports a cautious separation between learned anomaly detection and deterministic production rules. | The paper focuses on network intrusion detection; it should not be used to reject machine learning in all runtime-security settings. |

### Claims for 2.3.2

| ID | Cautious academic formulation | Source | Supported meaning | Limitations |
|---|---|---|---|---|
| P1 | Host-based intrusion detection monitors events and characteristics occurring on an individual host to identify suspicious activity. | Scarfone and Mell, 2007. DOI: [10.6028/NIST.SP.800-94](https://doi.org/10.6028/NIST.SP.800-94). | Positions runtime events as input to host-based detection. | The source is broader than eBPF and does not define this prototype's event schema. |
| P2 | Signature-based detection evaluates observed events against patterns associated with known threats or policy violations. | Scarfone and Mell, 2007, Section 2.3.1. DOI: [10.6028/NIST.SP.800-94](https://doi.org/10.6028/NIST.SP.800-94). | Supports deterministic condition matching. | Simple signatures may not retain state; the prototype extends the idea with bounded sequences. |
| P3 | Detection systems commonly record observed events and notify operators about events considered significant. | Scarfone and Mell, 2007. DOI: [10.6028/NIST.SP.800-94](https://doi.org/10.6028/NIST.SP.800-94). | Supports the conceptual transition from event to alert. | An alert remains an indication requiring interpretation, not a confirmed incident. |
| P4 | Event-stream processing can evaluate declarative patterns over continuously arriving information rather than treating every observation independently. | Cugola and Margara, 2012. DOI: [10.1145/2187671.2187677](https://doi.org/10.1145/2187671.2187677). | Provides the broad data-stream and complex-event-processing background. | The prototype implements only a bounded subset and should not be called a complete CEP system. |
| P5 | Ordered sequence patterns over event streams can be given explicit semantics and evaluated by maintaining intermediate matches. | Agrawal et al., 2008. DOI: [10.1145/1376616.1376634](https://doi.org/10.1145/1376616.1376634). | Supports modelling a detector as an ordered event-pattern query. | The paper's language and execution model are more general than the prototype. |
| P6 | A finite time window and explicit grouping key bound the interpretation and state of a local sequence detector. | Agrawal et al., 2008; Cugola and Margara, 2012. DOIs: [10.1145/1376616.1376634](https://doi.org/10.1145/1376616.1376634), [10.1145/2187671.2187677](https://doi.org/10.1145/2187671.2187677). | Supports temporal constraints and correlation attributes as event-pattern concepts. | The exact memory bound and process relationship are implementation properties to document in Chapter 4. |
| P7 | Separating event selection from detector evaluation permits different monitoring objectives to reuse a stable event representation. | Operational design claim for this thesis, consistent with configurable IDPS detection and alert settings in Scarfone and Mell, 2007. DOI: [10.6028/NIST.SP.800-94](https://doi.org/10.6028/NIST.SP.800-94). | Explains the value of the policy/detector split. | This is an architectural rationale, not a conclusion directly evaluated by NIST. It should be presented as a design choice. |
| P8 | Temporal order and a shared process relationship provide correlation evidence, but do not alone establish causality between events. | Methodological limitation derived from the bounded pattern model in Agrawal et al., 2008. DOI: [10.1145/1376616.1376634](https://doi.org/10.1145/1376616.1376634). | Prevents overclaiming what an ordered detector establishes. | This is a cautious inference, not a quoted theorem from the source. |

### Claims for 2.3.3

| ID | Cautious academic formulation | Source | Supported meaning | Limitations |
|---|---|---|---|---|
| M1 | MITRE ATT&CK is a knowledge base and model of adversary behaviour grounded in reported real-world observations. | Strom et al., 2020; [MITRE ATT&CK Get Started](https://attack.mitre.org/resources/). | Supports the basic description and provenance of ATT&CK. | ATT&CK documents observed behaviours and is not an exhaustive model of every possible adversary action. |
| M2 | In ATT&CK, tactics describe why an adversary performs an action, while techniques describe how a tactical objective may be achieved. | [MITRE ATT&CK FAQ](https://attack.mitre.org/resources/faq/); [Enterprise Tactics](https://attack.mitre.org/tactics/); [Enterprise Techniques](https://attack.mitre.org/techniques/). | Supports the tactic/technique distinction. | The chapter need not enumerate the complete matrix or current counts, which change across releases. |
| M3 | ATT&CK provides a shared behavioural vocabulary used across detection, threat hunting, security engineering, threat intelligence, and related activities. | Strom et al., 2020; [MITRE publication page](https://www.mitre.org/news-insights/publication/mitre-attck-design-and-philosophy). | Supports ATT&CK as a communication and classification framework. | Widespread adoption does not establish the quality of a specific detector mapping. |
| M4 | Mapping an alert to ATT&CK can connect low-level evidence to a recognisable adversary objective and behaviour, improving interpretation and exchange. | Strom et al., 2020; [MITRE ATT&CK Get Started](https://attack.mitre.org/resources/). | Supports the practical value of ATT&CK metadata in alert output. | The mapping should identify the concrete behaviour represented by the detector. |
| M5 | Defensive coverage should be assessed against relevant behaviours and procedures rather than inferred from a coloured technique label alone. | [MITRE ATT&CK Get Started, "How should I not use ATT&CK?"](https://attack.mitre.org/resources/). | Supports a precise but constructive qualification of coverage claims. | This does not prevent detector catalogues from reporting mapped tactics and techniques; it guides how those mappings are interpreted. |

## Suggested Paragraph Order

### 2.3.1

1. Define anomaly detection and the expected-behaviour reference.
2. Define point, contextual, and collective anomalies.
3. Explain that the categories can overlap.
4. Translate the taxonomy into runtime-security examples.
5. Distinguish learned/statistical anomaly detection from deterministic
   pattern matching.
6. Position the prototype as deterministic point-like and bounded collective
   detection, with general contextual anomaly detection outside its scope.

### 2.3.2

1. Introduce runtime events as normalised observations.
2. Define the thesis-specific roles of policy, detector, and alert.
3. Present the event-to-alert pipeline.
4. Explain single-event deterministic matching.
5. Introduce bounded ordered patterns for multi-event detection.
6. State the temporal, process-related, and causality limitations.
7. Close with the configurability and explainability gained from separating
   collection scope and detection logic.

### 2.3.3

1. Define ATT&CK as a behavioural knowledge base.
2. Explain tactics, techniques, sub-techniques, and procedures briefly.
3. Explain why ATT&CK metadata is useful for detector and alert semantics.
4. Describe mapping as a basis for interpretation, comparison, and exchange.
5. State that coverage is evaluated through concrete represented behaviours,
   not inferred from metadata labels alone.

## Verified BibTeX

The entries `chandola2009anomaly` and `strom2020attack` already exist in
`Thesis/bibliography.bib`. They are repeated here so that this dossier remains
self-contained. No Thesis file has been modified.

```bibtex
@article{chandola2009anomaly,
  author  = {Chandola, Varun and Banerjee, Arindam and Kumar, Vipin},
  title   = {Anomaly Detection: A Survey},
  journal = {ACM Computing Surveys},
  year    = {2009},
  volume  = {41},
  number  = {3},
  pages   = {15:1--15:58},
  month   = jul,
  doi     = {10.1145/1541880.1541882}
}

@techreport{strom2020attack,
  author      = {Strom, Blake E. and Applebaum, Andy and Miller, Douglas P. and Nickels, Kathryn C. and Pennington, Adam G. and Thomas, Cody B.},
  title       = {{MITRE ATT\&CK}: Design and Philosophy},
  institution = {The MITRE Corporation},
  year        = {2020},
  month       = mar,
  url         = {https://www.mitre.org/news-insights/publication/mitre-attck-design-and-philosophy}
}

@techreport{scarfone2007idps,
  author      = {Scarfone, Karen and Mell, Peter},
  title       = {Guide to Intrusion Detection and Prevention Systems ({IDPS})},
  institution = {National Institute of Standards and Technology},
  type        = {Special Publication},
  number      = {800-94},
  year        = {2007},
  month       = feb,
  doi         = {10.6028/NIST.SP.800-94},
  url         = {https://doi.org/10.6028/NIST.SP.800-94}
}

@article{cugola2012processing,
  author  = {Cugola, Gianpaolo and Margara, Alessandro},
  title   = {Processing Flows of Information: From Data Stream to Complex Event Processing},
  journal = {ACM Computing Surveys},
  year    = {2012},
  volume  = {44},
  number  = {3},
  pages   = {15:1--15:62},
  month   = jun,
  doi     = {10.1145/2187671.2187677}
}

@inproceedings{agrawal2008eventstreams,
  author    = {Agrawal, Jagrati and Diao, Yanlei and Gyllstrom, Daniel and Immerman, Neil},
  title     = {Efficient Pattern Matching over Event Streams},
  booktitle = {Proceedings of the 2008 ACM SIGMOD International Conference on Management of Data},
  year      = {2008},
  pages     = {147--159},
  publisher = {Association for Computing Machinery},
  address   = {New York, NY, USA},
  isbn      = {978-1-60558-102-6},
  doi       = {10.1145/1376616.1376634}
}

@inproceedings{sommer2010closedworld,
  author    = {Sommer, Robin and Paxson, Vern},
  title     = {Outside the Closed World: On Using Machine Learning for Network Intrusion Detection},
  booktitle = {2010 IEEE Symposium on Security and Privacy},
  year      = {2010},
  pages     = {305--316},
  publisher = {IEEE},
  doi       = {10.1109/SP.2010.25}
}

@online{mitre2026attackresources,
  author  = {{MITRE Corporation}},
  title   = {Get Started with {MITRE ATT\&CK}},
  year    = {2026},
  url     = {https://attack.mitre.org/resources/},
  urldate = {2026-07-27}
}

@online{mitre2026attackfaq,
  author  = {{MITRE Corporation}},
  title   = {{MITRE ATT\&CK} Frequently Asked Questions},
  year    = {2026},
  url     = {https://attack.mitre.org/resources/faq/},
  urldate = {2026-07-27}
}
```

## Source Verification Notes

- Chandola et al.: title, authors, journal, volume, issue, article pagination,
  year, and DOI were verified against the author-hosted ACM manuscript.
- Scarfone and Mell: authors, title, publication number, date, and DOI were
  verified against the official NIST publication.
- Cugola and Margara: title, authors, journal, volume, issue, article
  pagination, and DOI were verified against the published survey metadata and
  manuscript.
- Agrawal et al.: title, authors, venue, pagination, ISBN, and DOI were
  verified against the institutional research record and ACM DOI metadata.
- Sommer and Paxson: title, authors, venue, pagination, year, and DOI were
  verified against DBLP and the IEEE DOI record.
- Strom et al. and the ATT&CK web resources were verified against official
  MITRE pages.

## Final Editorial Boundaries

- Use *rule-based detection* or *deterministic detection* for the implemented
  detector engine.
- Use *anomaly taxonomy* when discussing point, contextual, and collective
  forms from the literature.
- Use *bounded collective sequence* or *bounded ordered event pattern* for
  the implemented multi-event capability.
- Do not claim statistical, unsupervised, or machine-learning anomaly
  detection.
- Do not claim contextual anomaly detection merely because events contain
  process metadata.
- Do not claim general anomaly discovery, causal inference, or complete
  ATT&CK coverage.
- Present ATT&CK mapping positively as semantic classification, traceability,
  interpretability, and interoperability.
