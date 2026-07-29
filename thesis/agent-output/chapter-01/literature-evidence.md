# Chapter 1 literature evidence

Status: literature-research output for Wave 1  
Scope: evidence collection for Chapter 1, not chapter prose  
Last verification: 2026-07-17

## 1. Method and evidentiary boundaries

This dossier maps the claims planned in the Chapter 1 outline to sources that can be cited without relying on generic technical blogs. Sources are ordered by authority: peer-reviewed papers and conference proceedings, official Linux and Kubernetes documentation, MITRE publications, and official project documentation.

The following distinctions should be preserved when writing the chapter:

- Official documentation is authoritative for the current behavior and declared architecture of a project, but it is not independent evidence of effectiveness or performance.
- A tool being based on eBPF does not by itself establish low overhead. Performance claims about Vesuvius require the project's own reproducible benchmarks.
- A DaemonSet is a conventional mechanism for running an agent on Kubernetes nodes. It does not by itself prove that a node agent cannot consume cluster context; the local/central boundary is an architectural choice made by Vesuvius.
- YAML policies, multi-event detection, MITRE metadata, process-aware selection, and kernel-side filtering already exist in related tools. The contribution must therefore be stated in terms of Vesuvius's specific combination, target constraints, implementation, and evaluation.
- The 2025 comparative IEEE Access paper is useful but postdates several foundational project decisions and should not be treated as the sole authority for any claim.

## 2. Claim-to-source map

| Chapter 1 claim | Best supporting sources | Recommended strength |
|---|---|---|
| Runtime monitoring benefits from visibility at kernel execution points | Gbadamosi et al.; Linux eBPF Userspace API; Tracee and Falco documentation | Strong for technical capability; avoid claiming complete visibility |
| eBPF programs are checked by an in-kernel verifier | Linux verifier documentation; Gbadamosi et al.; `bpf(2)` | Strong |
| eBPF may be JIT-compiled into native instructions | `bpf(2)` | Strong, but say "may" because JIT availability/configuration is platform-dependent |
| Node-level agents are a normal Kubernetes monitoring deployment model | Kubernetes DaemonSet documentation | Strong |
| eBPF security tools are used in cloud-native and Kubernetes environments | Her et al.; Tracee; Falco; Tetragon | Strong |
| Point, contextual, and collective anomalies are established categories | Chandola, Banerjee, and Kumar | Strong |
| ATT&CK organizes observed adversary behavior using tactics and techniques | MITRE ATT&CK Design and Philosophy | Strong |
| Tracee provides eBPF-based runtime security, policies, and signatures | Official Tracee documentation | Strong for product architecture, not independent validation |
| Multi-event and process-aware detection predates Vesuvius | Tracee K8s API Connection signature; Tetragon selectors | Strong enough to constrain novelty claims |
| Kernel-side filtering can reduce userspace event traffic | Tetragon selector/enforcement documentation | Strong for existence and rationale; Vesuvius benefit still needs measurement |
| eBPF in containers has security trade-offs because containers share a kernel | He et al.; BPFContain | Strong |

## 3. Source records

### S1. Gbadamosi et al., The eBPF Runtime in the Linux Kernel

1. **Supported claim.** eBPF is a programmable runtime inside the kernel; programs run at designated hooks after verifier analysis. It supports runtime extension and instrumentation while preserving kernel integrity goals.
2. **Bibliographic metadata.** Bolaji Gbadamosi, Luigi Leonardi, Tobias Pulls, Toke Høiland-Jørgensen, Simone Ferlin-Reiter, Simo Sorce, and Anna Brunström. *The eBPF Runtime in the Linux Kernel*. arXiv preprint arXiv:2410.00026, 2024.
3. **DOI / canonical URL.** DOI: <https://doi.org/10.48550/arXiv.2410.00026>. Canonical record: <https://arxiv.org/abs/2410.00026>.
4. **Concise paraphrase.** eBPF allows programs to be loaded dynamically and executed at selected kernel hooks. A verifier reasons about program safety before execution, providing a controlled form of kernel programmability.
5. **Relevant Chapter 1 section.** Sections 1.1, 1.2, and the high-level part of 1.3.
6. **Confidence and limitations.** High confidence for architecture and terminology. It is a recent preprint rather than a peer-reviewed systems paper, and its general statements do not prove Vesuvius performance on Rocky Linux 4.18.
7. **Verified BibTeX proposal.** Metadata was checked against the arXiv and Red Hat Research records.

```bibtex
@article{gbadamosi2024ebpf,
  author        = {Gbadamosi, Bolaji and Leonardi, Luigi and Pulls, Tobias and H{\o}iland-J{\o}rgensen, Toke and Ferlin-Reiter, Simone and Sorce, Simo and Brunstr{\"o}m, Anna},
  title         = {The {eBPF} Runtime in the {Linux} Kernel},
  year          = {2024},
  journal       = {arXiv preprint arXiv:2410.00026},
  doi           = {10.48550/arXiv.2410.00026},
  url           = {https://arxiv.org/abs/2410.00026}
}
```

### S2. Linux kernel eBPF Userspace API documentation

1. **Supported claim.** eBPF is a sandboxed kernel runtime used to extend operating-system capabilities and attach programs to networking, tracing, and security hooks.
2. **Bibliographic metadata.** Linux kernel community. *eBPF Userspace API*. Linux Kernel Documentation, version 6.9 documentation snapshot.
3. **DOI / canonical URL.** <https://www.kernel.org/doc/html/v6.9/userspace-api/ebpf/index.html>.
4. **Concise paraphrase.** The kernel exposes eBPF as a facility for loading constrained programs and associating them with supported hook types, making kernel events available without modifying kernel source code for each use case.
5. **Relevant Chapter 1 section.** Sections 1.1, 1.2, and 1.3.
6. **Confidence and limitations.** Very high authority for Linux interfaces. Documentation is versioned and descriptive, not an evaluation of security tools or overhead.
7. **Verified BibTeX proposal.** Corporate authorship and URL were checked against the official kernel documentation.

```bibtex
@online{linux2024ebpfuserspaceapi,
  author  = {{Linux Kernel Community}},
  title   = {{eBPF} Userspace {API}},
  url     = {https://www.kernel.org/doc/html/v6.9/userspace-api/ebpf/index.html},
  urldate = {2026-07-17},
  note    = {Linux kernel 6.9 documentation}
}
```

### S3. Linux kernel eBPF verifier documentation

1. **Supported claim.** The verifier validates control flow and symbolically explores program paths while tracking register, pointer, stack, bounds, and helper-call safety.
2. **Bibliographic metadata.** Linux kernel community. *eBPF verifier*. Linux Kernel Documentation.
3. **DOI / canonical URL.** <https://docs.kernel.org/bpf/verifier.html>.
4. **Concise paraphrase.** Before loading a program, the verifier checks its control-flow graph and simulates instructions across feasible paths. It rejects unsafe memory use, invalid pointer operations, uninitialized values, and invalid helper calls.
5. **Relevant Chapter 1 section.** Section 1.3, limited to a conceptual explanation; implementation details belong in Chapter 2.
6. **Confidence and limitations.** Very high authority. The verifier evolves across kernel versions, so details from current documentation must not be assumed to exist unchanged on Rocky Linux's 4.18-based kernel.
7. **Verified BibTeX proposal.** URL and title were checked against official kernel documentation.

```bibtex
@online{linux2026ebpfverifier,
  author  = {{Linux Kernel Community}},
  title   = {{eBPF} verifier},
  url     = {https://docs.kernel.org/bpf/verifier.html},
  urldate = {2026-07-17}
}
```

### S4. `bpf(2)` Linux manual page

1. **Supported claim.** The `bpf()` system call manages BPF programs and maps; programs are verified before loading and may be JIT-compiled to native machine code.
2. **Bibliographic metadata.** Linux man-pages project. *bpf(2) -- perform a command on an extended BPF map or program*. Linux manual page.
3. **DOI / canonical URL.** <https://man7.org/linux/man-pages/man2/bpf.2.html>.
4. **Concise paraphrase.** Userspace interacts with eBPF objects through the `bpf()` system call. The kernel verifies submitted bytecode and can translate accepted instructions into native code using a JIT compiler.
5. **Relevant Chapter 1 section.** Section 1.3.
6. **Confidence and limitations.** High authority for the userspace interface. JIT support and activation are architecture- and configuration-dependent; Chapter 1 should use conditional wording.
7. **Verified BibTeX proposal.** Title and canonical URL were checked against the Linux man-pages rendering.

```bibtex
@manual{linuxmanpages_bpf2,
  author  = {{Linux man-pages project}},
  title   = {{bpf(2)} -- Perform a Command on an Extended {BPF} Map or Program},
  url     = {https://man7.org/linux/man-pages/man2/bpf.2.html},
  urldate = {2026-07-17}
}
```

### S5. McCanne and Jacobson, The BSD Packet Filter

1. **Supported claim.** BPF originated as an efficient packet-filtering architecture using a register-based filter machine and code generation, providing historical context for eBPF.
2. **Bibliographic metadata.** Steven McCanne and Van Jacobson. “The BSD Packet Filter: A New Architecture for User-level Packet Capture.” In *Proceedings of the USENIX Winter 1993 Conference*, pages 259–270, San Diego, California, January 1993. USENIX Association.
3. **DOI / canonical URL.** No DOI was found. <https://www.usenix.org/conference/usenix-winter-1993-conference/bsd-packet-filter-new-architecture-user-level-packet>.
4. **Concise paraphrase.** The original BPF design moved selective packet filtering close to the capture path and used a compact execution model designed for efficient filtering.
5. **Relevant Chapter 1 section.** Optional historical sentence in Section 1.3; deeper history belongs in Chapter 2.
6. **Confidence and limitations.** High confidence and peer-reviewed conference provenance. It describes classic BPF, not modern eBPF tracing or runtime security.
7. **Verified BibTeX proposal.** Proceedings metadata was checked against the USENIX record.

```bibtex
@inproceedings{mccanne1993bsd,
  author    = {McCanne, Steven and Jacobson, Van},
  title     = {The {BSD} Packet Filter: A New Architecture for User-level Packet Capture},
  booktitle = {Proceedings of the USENIX Winter 1993 Conference},
  year      = {1993},
  pages     = {259--270},
  publisher = {USENIX Association},
  address   = {San Diego, California},
  url       = {https://www.usenix.org/conference/usenix-winter-1993-conference/bsd-packet-filter-new-architecture-user-level-packet}
}
```

### S6. Chandola, Banerjee, and Kumar, Anomaly Detection: A Survey

1. **Supported claim.** Point, contextual, and collective anomalies are established anomaly categories with materially different interpretation requirements.
2. **Bibliographic metadata.** Varun Chandola, Arindam Banerjee, and Vipin Kumar. “Anomaly Detection: A Survey.” *ACM Computing Surveys*, volume 41, issue 3, Article 15, pages 15:1–15:58, July 2009.
3. **DOI / canonical URL.** DOI: <https://doi.org/10.1145/1541880.1541882>.
4. **Concise paraphrase.** A point anomaly is anomalous in isolation; a contextual anomaly is anomalous only under a specified context; a collective anomaly is identified because a related group of observations is anomalous as a whole.
5. **Relevant Chapter 1 section.** Sections 1.2 and 1.3, especially the motivation for local point and collective detectors and the exclusion of cluster-contextual analysis.
6. **Confidence and limitations.** Very high confidence; this is a foundational peer-reviewed survey. The taxonomy is domain-independent and does not prescribe how Vesuvius must correlate Linux events.
7. **Verified BibTeX proposal.** Metadata and DOI were checked against the ACM-linked institutional record and the paper.

```bibtex
@article{chandola2009anomaly,
  author     = {Chandola, Varun and Banerjee, Arindam and Kumar, Vipin},
  title      = {Anomaly Detection: A Survey},
  journal    = {ACM Computing Surveys},
  year       = {2009},
  volume     = {41},
  number     = {3},
  articleno  = {15},
  pages      = {15:1--15:58},
  month      = jul,
  doi        = {10.1145/1541880.1541882},
  publisher  = {Association for Computing Machinery}
}
```

### S7. Strom et al., MITRE ATT&CK: Design and Philosophy

1. **Supported claim.** ATT&CK is a knowledge base and model of adversary behavior grounded in observed activity; tactics represent adversary goals and techniques represent ways to achieve them.
2. **Bibliographic metadata.** Blake E. Strom, Andy Applebaum, Douglas P. Miller, Kathryn C. Nickels, Adam G. Pennington, and Cody B. Thomas. *MITRE ATT&CK: Design and Philosophy*. MITRE technical report, March 2020.
3. **DOI / canonical URL.** No DOI was found. <https://www.mitre.org/news-insights/publication/mitre-attck-design-and-philosophy>.
4. **Concise paraphrase.** ATT&CK provides a common representation of real-world adversary behavior. It organizes behavior according to operational goals and the techniques used to pursue those goals, making it useful for detection engineering and threat-informed communication.
5. **Relevant Chapter 1 section.** Sections 1.2, 1.3, and 1.4.
6. **Confidence and limitations.** Very high authority for ATT&CK's design. ATT&CK mapping adds semantic interoperability, but a mapping alone does not demonstrate detection correctness, completeness, or novelty.
7. **Verified BibTeX proposal.** Authors, title, and date were checked against the official MITRE publication page.

```bibtex
@techreport{strom2020attack,
  author      = {Strom, Blake E. and Applebaum, Andy and Miller, Douglas P. and Nickels, Kathryn C. and Pennington, Adam G. and Thomas, Cody B.},
  title       = {{MITRE ATT\&CK}: Design and Philosophy},
  institution = {The MITRE Corporation},
  year        = {2020},
  month       = mar,
  url         = {https://www.mitre.org/news-insights/publication/mitre-attck-design-and-philosophy}
}
```

### S8. Kubernetes DaemonSet documentation

1. **Supported claim.** A DaemonSet ensures that all or selected Kubernetes nodes run a copy of a Pod, and node monitoring is a standard DaemonSet use case.
2. **Bibliographic metadata.** The Kubernetes Authors. *DaemonSet*. Kubernetes Documentation.
3. **DOI / canonical URL.** <https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/>.
4. **Concise paraphrase.** Kubernetes provides DaemonSets for node-scoped services whose Pod replicas should follow node membership. Monitoring and logging agents are typical examples.
5. **Relevant Chapter 1 section.** Sections 1.1, 1.2, and 1.3 when explaining the intended Vesuvius deployment model.
6. **Confidence and limitations.** Very high authority for Kubernetes behavior. It supports deployment rationale, not security efficacy, and does not require that all analysis remain node-local.
7. **Verified BibTeX proposal.** Page title and corporate authorship convention were checked against official Kubernetes documentation.

```bibtex
@online{kubernetes_daemonset,
  author  = {{The Kubernetes Authors}},
  title   = {DaemonSet},
  url     = {https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/},
  urldate = {2026-07-17},
  note    = {Kubernetes Documentation}
}
```

### S9. He et al., Cross Container Attacks

1. **Supported claim.** eBPF is powerful in cloud environments, but privileged eBPF use and a shared host kernel create security risks across container boundaries.
2. **Bibliographic metadata.** Yi He, Roland Guo, Yunlong Xing, Xijia Che, Kun Sun, Zhuotao Liu, Ke Xu, and Qi Li. “Cross Container Attacks: The Bewildered eBPF on Clouds.” In *32nd USENIX Security Symposium (USENIX Security 23)*, pages 5971–5988, Anaheim, California, August 2023. USENIX Association. ISBN 978-1-939133-37-3.
3. **DOI / canonical URL.** No DOI was found. <https://www.usenix.org/conference/usenixsecurity23/presentation/he>.
4. **Concise paraphrase.** Because containers share a kernel, powerful eBPF privileges can create cross-container attack surfaces if capabilities and program behavior are not tightly controlled.
5. **Relevant Chapter 1 section.** Sections 1.1 and 1.2; it supports both the value and the security sensitivity of kernel-level monitoring in container platforms.
6. **Confidence and limitations.** Very high confidence; peer-reviewed USENIX Security paper. It studies attack risks and should not be used to imply that all eBPF monitoring agents are insecure.
7. **Verified BibTeX proposal.** Authors, pages, venue, location, year, and ISBN were checked against the official USENIX record.

```bibtex
@inproceedings{he2023crosscontainer,
  author    = {He, Yi and Guo, Roland and Xing, Yunlong and Che, Xijia and Sun, Kun and Liu, Zhuotao and Xu, Ke and Li, Qi},
  title     = {Cross Container Attacks: The Bewildered {eBPF} on Clouds},
  booktitle = {32nd USENIX Security Symposium (USENIX Security 23)},
  year      = {2023},
  pages     = {5971--5988},
  publisher = {USENIX Association},
  address   = {Anaheim, CA},
  isbn      = {978-1-939133-37-3},
  url       = {https://www.usenix.org/conference/usenixsecurity23/presentation/he}
}
```

### S10. Her et al., Analysis of eBPF-Based Security Tools

1. **Supported claim.** KubeArmor, Falco, Tetragon, and Tracee are established eBPF-based security tools for cloud-native environments and differ in architecture, audit capabilities, and performance.
2. **Bibliographic metadata.** Jin Her, Jongseop Kim, Jinwoo Kim, and Seungsoo Lee. “An In-Depth Analysis of eBPF-Based System Security Tools in Cloud-Native Environments.” *IEEE Access*, volume 13, pages 155588–155604, 2025.
3. **DOI / canonical URL.** DOI: <https://doi.org/10.1109/ACCESS.2025.3605432>. IEEE record: <https://ieeexplore.ieee.org/document/11146725/>.
4. **Concise paraphrase.** Current cloud-native security tools use eBPF to observe process and network activity, emit security-relevant events, and in some cases enforce policy; their engineering trade-offs differ enough to require comparative evaluation.
5. **Relevant Chapter 1 section.** Sections 1.1, 1.2, and 1.4.
6. **Confidence and limitations.** High confidence; peer-reviewed journal article with direct relevance. It is recent and should be complemented by each project's primary documentation when describing current features.
7. **Verified BibTeX proposal.** Metadata was checked against the DOI/IEEE record.

```bibtex
@article{her2025ebpfsecuritytools,
  author    = {Her, Jin and Kim, Jongseop and Kim, Jinwoo and Lee, Seungsoo},
  title     = {An In-Depth Analysis of {eBPF}-Based System Security Tools in Cloud-Native Environments},
  journal   = {IEEE Access},
  year      = {2025},
  volume    = {13},
  pages     = {155588--155604},
  doi       = {10.1109/ACCESS.2025.3605432},
  publisher = {IEEE},
  url       = {https://ieeexplore.ieee.org/document/11146725/}
}
```

### S11. Tracee official documentation

1. **Supported claim.** Tracee is an eBPF-based runtime security and forensic tool that observes system activity, supports Kubernetes deployment, and exposes event and detection functionality.
2. **Bibliographic metadata.** Aqua Security. *Tracee Documentation*. Official project documentation.
3. **DOI / canonical URL.** <https://aquasecurity.github.io/tracee/latest/>.
4. **Concise paraphrase.** Tracee collects Linux runtime events using eBPF and turns them into security and forensic information. Its documented deployment modes include container and Kubernetes environments.
5. **Relevant Chapter 1 section.** Sections 1.1, 1.3, and 1.4.
6. **Confidence and limitations.** High authority for declared project features, but it is vendor/project documentation rather than independent evidence of accuracy or performance. Features may differ between documented versions.
7. **Verified BibTeX proposal.** Project owner, page title, and URL were checked against official documentation.

```bibtex
@online{aquasecurity_tracee,
  author  = {{Aqua Security}},
  title   = {Tracee Documentation},
  url     = {https://aquasecurity.github.io/tracee/latest/},
  urldate = {2026-07-17}
}
```

### S12. Tracee policy documentation

1. **Supported claim.** Tracee already supports declarative policy files with event selection, scope, and data filters; policy-driven event selection is therefore prior art.
2. **Bibliographic metadata.** Aqua Security. *Tracee Policies: Overview*. Official Tracee development documentation.
3. **DOI / canonical URL.** <https://aquasecurity.github.io/tracee/dev/docs/policies/>.
4. **Concise paraphrase.** Tracee policies describe which events to observe and constrain them by workload/process scope and event data. Policies can be supplied as structured configuration rather than hard-coded event lists.
5. **Relevant Chapter 1 section.** Sections 1.3 and 1.4, especially the comparison between Vesuvius policies and Tracee.
6. **Confidence and limitations.** High authority for current development documentation. The `dev` URL can change; a thesis citation should ideally pin the Tracee release or Git commit used in the comparison.
7. **Verified BibTeX proposal.** Title and URL were checked against official Tracee documentation.

```bibtex
@online{aquasecurity_tracee_policies,
  author  = {{Aqua Security}},
  title   = {Tracee Policies: Overview},
  url     = {https://aquasecurity.github.io/tracee/dev/docs/policies/},
  urldate = {2026-07-17}
}
```

### S13. Tracee K8s API Connection signature

1. **Supported claim.** Tracee has detection logic that consumes more than one event type and retains process-related state, so multi-event/stateful detection is not new by itself.
2. **Bibliographic metadata.** Aqua Security. *K8s API Connection*. Tracee built-in signature documentation, version 0.22 documentation.
3. **DOI / canonical URL.** <https://aquasecurity.github.io/tracee/v0.22/docs/events/builtin/signatures/kubernetes_api_connection/>.
4. **Concise paraphrase.** The documented signature combines process execution and socket-connection evidence and keeps information learned from prior process events to interpret later connections to the Kubernetes API.
5. **Relevant Chapter 1 section.** Section 1.4 and the novelty discussion in 1.3.
6. **Confidence and limitations.** High confidence that Tracee contains stateful multi-event signatures. This is one signature and does not establish that Tracee uses the same generic short-window collective engine or process-key semantics as Vesuvius.
7. **Verified BibTeX proposal.** Versioned title and URL were checked against official Tracee documentation.

```bibtex
@online{aquasecurity_tracee_k8s_api_connection,
  author  = {{Aqua Security}},
  title   = {{K8s API Connection}},
  url     = {https://aquasecurity.github.io/tracee/v0.22/docs/events/builtin/signatures/kubernetes_api_connection/},
  urldate = {2026-07-17},
  note    = {Tracee v0.22 built-in signature documentation}
}
```

### S14. Falco official documentation and rule style guide

1. **Supported claim.** Falco performs runtime threat detection on hosts, containers, and Kubernetes using event sources and rules; its rule conventions include MITRE tactic and technique tags.
2. **Bibliographic metadata.** The Falco Authors. *The Falco Project* and *Style Guide of Falco Rules*. Official Falco documentation.
3. **DOI / canonical URL.** <https://falco.org/docs/> and <https://falco.org/docs/concepts/rules/style-guide/>.
4. **Concise paraphrase.** Falco evaluates runtime events against rules and emits alerts. Rule metadata may classify detections with MITRE ATT&CK tactic and technique identifiers, demonstrating that ATT&CK-enriched detector output is established practice.
5. **Relevant Chapter 1 section.** Sections 1.2, 1.3, and 1.4.
6. **Confidence and limitations.** High authority for Falco's intended operation and rule format. It does not show that Vesuvius's exact alert contract or collective correlations are identical.
7. **Verified BibTeX proposal.** Titles and URLs were checked against official Falco documentation. Two entries are preferable because they support distinct claims.

```bibtex
@online{falco_documentation,
  author  = {{The Falco Authors}},
  title   = {The Falco Project},
  url     = {https://falco.org/docs/},
  urldate = {2026-07-17}
}

@online{falco_rule_style,
  author  = {{The Falco Authors}},
  title   = {Style Guide of Falco Rules},
  url     = {https://falco.org/docs/concepts/rules/style-guide/},
  urldate = {2026-07-17}
}
```

### S15. Tetragon tracing policies and selectors

1. **Supported claim.** Tetragon exposes declarative tracing policies and selectors, including process, parent-process, namespace, capability, and workload criteria; filtering can be performed in the kernel to reduce events sent to userspace.
2. **Bibliographic metadata.** The Tetragon Authors. *Tracing Policy*, *Selectors*, and *Policy Enforcement*. Official Tetragon documentation.
3. **DOI / canonical URL.** <https://tetragon.io/docs/concepts/tracing-policy/>, <https://tetragon.io/docs/concepts/tracing-policy/selectors/>, and <https://tetragon.io/docs/getting-started/enforcement/>.
4. **Concise paraphrase.** Tetragon policies define hooks and matching criteria. Selectors can use process identity and lineage-related fields, while in-kernel filtering avoids forwarding every observed event to userspace.
5. **Relevant Chapter 1 section.** Sections 1.3 and 1.4; particularly relevant to policy architecture, process-aware matching, and the hybrid filtering trade-off.
6. **Confidence and limitations.** High authority for Tetragon features. Project documentation does not prove that a specific selector configuration meets Vesuvius's 5% CPU target or works on Rocky Linux 4.18.
7. **Verified BibTeX proposal.** Titles and URLs were checked against official Tetragon documentation. Separate entries preserve claim provenance.

```bibtex
@online{tetragon_tracing_policy,
  author  = {{The Tetragon Authors}},
  title   = {Tracing Policy},
  url     = {https://tetragon.io/docs/concepts/tracing-policy/},
  urldate = {2026-07-17}
}

@online{tetragon_selectors,
  author  = {{The Tetragon Authors}},
  title   = {Selectors},
  url     = {https://tetragon.io/docs/concepts/tracing-policy/selectors/},
  urldate = {2026-07-17}
}

@online{tetragon_policy_enforcement,
  author  = {{The Tetragon Authors}},
  title   = {Policy Enforcement},
  url     = {https://tetragon.io/docs/getting-started/enforcement/},
  urldate = {2026-07-17}
}
```

### S16. Findlay, Barrera, and Somayaji, BPFContain

1. **Supported claim.** eBPF-backed policy mechanisms for container security and flexible policy languages existed before Vesuvius.
2. **Bibliographic metadata.** William Findlay, David Barrera, and Anil Somayaji. *BPFContain: Fixing the Soft Underbelly of Container Security*. arXiv preprint arXiv:2102.06972, 2021.
3. **DOI / canonical URL.** Canonical record: <https://arxiv.org/abs/2102.06972>. No publisher DOI was verified.
4. **Concise paraphrase.** BPFContain combines an eBPF implementation with a policy language intended to express container confinement rules and integrate with container management systems.
5. **Relevant Chapter 1 section.** Section 1.4 and novelty delimitation.
6. **Confidence and limitations.** Medium-high confidence. It is a research preprint and focuses on confinement/enforcement rather than the same monitoring and detector pipeline as Vesuvius.
7. **Verified BibTeX proposal.** Authors, title, year, and arXiv identifier were checked against the canonical arXiv record. No unverified DOI is included.

```bibtex
@article{findlay2021bpfcontain,
  author  = {Findlay, William and Barrera, David and Somayaji, Anil},
  title   = {{BPFContain}: Fixing the Soft Underbelly of Container Security},
  journal = {arXiv preprint arXiv:2102.06972},
  year    = {2021},
  url     = {https://arxiv.org/abs/2102.06972}
}
```

## 4. Candidate novelty assessment

### N1. YAML point and collective local detectors with configurable correlation

**Classification: requiring further comparison.**

The exact implementation may be a defensible engineering contribution because it provides a generic bounded-window engine with compiled process, resource, cgroup and composite identities, predictable state limits, and compatibility with an older enterprise kernel. However, the broad claim of stateful or multi-event detection is not novel: Tracee has stateful multi-event signatures and process-aware scopes; Falco has declarative runtime rules; Tetragon has process and parent-process selectors. The thesis must compare semantics, extensibility, state ownership, memory bounds, false-positive controls, and resource costs.

Potentially defensible formulation:

> A lightweight, bounded and locally correlated collective detector pipeline designed and evaluated for a node-local eBPF agent on Rocky Linux 8's 4.18-based kernel.

Evidence still required: feature matrix, detector-state algorithm, benchmark under controlled load, false-positive examples, and a precise difference from Tracee signatures.

### N2. MITRE ATT&CK as alert metadata and interface contract

**Classification: already established.**

Tracee signatures and Falco rules already carry MITRE ATT&CK identifiers or tags. Treating ATT&CK identifiers as stable alert metadata is useful and professionally relevant, but not a standalone research novelty. Vesuvius may claim a consistent alert schema and validation discipline as an implementation contribution.

If future policy selection is driven by ATT&CK identifiers, that also requires comparison with tools that enable or group rules by tags. The contribution would need to be stronger than simply adding tactic and technique fields.

### N3. Adapting Tracee-inspired patterns to Rocky Linux 4.18

**Classification: potentially defensible.**

This is the strongest current thesis contribution if written as a constrained engineering and empirical study rather than a new eBPF mechanism. The target kernel creates concrete compatibility limits involving helper availability, attach mechanisms, verifier behavior, BTF/CO-RE realities, and ring-buffer availability. Vesuvius's decisions can be defended if each adaptation is documented and tested.

Necessary limitation: "Rocky Linux 4.18" should be described carefully because enterprise distributions backport features; upstream version numbers alone do not establish the available feature set.

Evidence still required: target kernel capability table, exact unsupported patterns, fallback decisions, verifier evidence, and reproducible runtime results.

### N4. Separation of local point/collective detection from centralized contextual analysis

**Classification: already established at the architectural level; potentially defensible as a scoped design decision.**

Node agents deployed as DaemonSets and centralized Kubernetes security backends are common. Therefore, separating node-local collection from cluster-wide context is not an invention. What can be defended is the explicit boundary chosen by Vesuvius: point and short-window collective anomalies are evaluated locally, while cluster-contextual analysis is deliberately delegated to a future central component.

The chapter must present this as scope and architecture, not as a proven universal rule. A node agent can technically receive Kubernetes metadata, so "cannot detect contextual anomalies" would be too absolute.

### N5. Generic detector engine plus minimal kernel-side prefilter

**Classification: requiring further comparison.**

The hybrid principle is established: Tetragon explicitly provides in-kernel selectors to reduce userspace traffic while retaining higher-level policy logic. Vesuvius may still contribute a deliberately minimal prefilter compatible with its old target kernel and a generic userspace detector engine.

The novelty depends on measured trade-offs, not architecture diagrams alone. Required evidence includes CPU, RSS, event throughput, dropped events, alert equivalence with and without filtering, and the cost of each detector profile.

### N6. Vesuvius as a new general-purpose eBPF runtime-security architecture

**Classification: too broad.**

Tracee, Falco, Tetragon, KubeArmor, BPFContain, and related research already occupy this space. The thesis should avoid claiming a new category of runtime-security system. It should instead claim a focused prototype, a constrained adaptation, a detector/policy implementation, and an empirical evaluation.

### N7. Low-overhead runtime security below 5% of one CPU core

**Classification: unsupported until the benchmark campaign is complete.**

No external source can establish this property for Vesuvius. Current project measurements vary by event profile, workload, warm-up, detector set, and filter configuration. The 5% value is a project requirement, not yet a scientific result. Chapter 1 may state it as an objective; the final conclusion must depend on Chapter 5 measurements.

## 5. Recommended citation placement in Chapter 1

### Section 1.1: context and motivation

- Use Gbadamosi et al. and the Linux eBPF API for kernel-level runtime instrumentation.
- Use Kubernetes DaemonSet documentation for the node-agent deployment model.
- Use He et al. to explain why shared-kernel container environments require careful security monitoring and privilege management.
- Use Her et al. to establish that eBPF-based runtime security is an active cloud-native tool category.

### Section 1.2: problem statement

- Use Chandola et al. for point, contextual, and collective anomaly definitions.
- Use MITRE's design paper for threat-informed terminology.
- Present the old-kernel target, local/central split, event-registry integrity, and CPU target as project constraints supported by repository evidence, not literature claims.

### Section 1.3: objectives and contributions

- Cite Linux verifier documentation and `bpf(2)` only for the technical basis.
- State the 5% CPU threshold as an evaluation objective.
- Describe MITRE enrichment and YAML policies as features, not standalone novelty.
- Frame Rocky Linux adaptation and bounded collective detection as candidate contributions subject to Chapter 4/5 evidence.

### Section 1.4: thesis organization and relation to later chapters

- Use Tracee, Falco, Tetragon, BPFContain, and Her et al. to motivate the related-work comparison that belongs mainly in Chapter 2.
- Avoid turning Section 1.4 into a full product comparison; it should explain where the comparison and evaluation will appear.

## 6. Claims that should not be written without qualification

- **Avoid:** "eBPF provides complete visibility into the operating system."  
  **Use:** eBPF exposes a broad set of supported kernel hooks; coverage depends on selected hooks, kernel support, privileges, and tool implementation.

- **Avoid:** "The verifier guarantees that an eBPF security tool is secure."  
  **Use:** the verifier enforces program-safety properties before loading; it does not prove the detection logic correct or eliminate privileged deployment risks.

- **Avoid:** "JIT compilation always makes eBPF faster."  
  **Use:** supported kernels may JIT-compile accepted eBPF bytecode into native instructions; actual performance is configuration- and workload-dependent.

- **Avoid:** "Vesuvius introduces collective anomaly detection for eBPF runtime security."  
  **Use:** The prototype implements a bounded, locally correlated collective detector model whose specific trade-offs are compared with existing stateful signatures and policy engines.

- **Avoid:** "MITRE ATT&CK integration is novel."  
  **Use:** Vesuvius uses ATT&CK identifiers as interoperable alert metadata and validates their presence according to its detector contract.

- **Avoid:** "A node agent cannot detect contextual anomalies."  
  **Use:** Vesuvius deliberately limits the local agent to point and short-window collective detections and reserves cluster-wide context for a centralized component.

- **Avoid:** "Vesuvius has negligible overhead" or "remains below 5%."  
  **Use before final experiments:** the project targets a steady-state userspace CPU cost below 5% of one core under a defined benchmark profile.

## 7. Remaining literature work before Chapter 1 writing

1. Pin the exact Tracee, Falco, and Tetragon versions or Git commits used for the final comparison.
2. Add an authoritative Rocky Linux/RHEL source documenting the target kernel's backported eBPF capabilities; upstream kernel version alone is insufficient.
3. Build a feature-comparison table for bounded collective state, process lineage, policy language, kernel filtering, ATT&CK metadata, and old-kernel compatibility.
4. Link every quantitative Vesuvius claim to a repository benchmark artifact with workload, duration, warm-up, event set, detector set, and host configuration.
5. Decide whether Chapter 1 needs the historical BPF citation; otherwise move McCanne and Jacobson entirely to Chapter 2.
