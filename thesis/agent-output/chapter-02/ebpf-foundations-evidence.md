# Chapter 2 Evidence Dossier: Linux Observability and eBPF Foundations

## Purpose and scope

This dossier supports the preparation of Sections 2.1 and 2.2 of Chapter 2. It
is an evidence map rather than thesis prose. It deliberately avoids describing
the implementation of the proposed monitoring system.

The claims below use cautious academic wording and distinguish what each source
establishes from conclusions that would require additional evidence. Each
claim identifies a proposed citation key. Full, deduplicated, and URL-verified
BibTeX proposals are provided in the final section.

## Terminology decision: BPF and eBPF

RFC 9669 states that **eBPF is no longer an acronym** and that the technology is
also commonly called **BPF**. In this thesis, `eBPF` should remain the default
term because it clearly distinguishes the modern instruction set and Linux
runtime from classic BPF (`cBPF`). The expansion “extended Berkeley Packet
Filter” may be mentioned once as historical terminology, but it should not be
presented as the current formal name.

This also implies that an acronym list should not define eBPF only as “extended
Berkeley Packet Filter”. A more accurate entry would be:

> **eBPF**: the modern BPF instruction set and execution environment; the name
> is retained historically but is no longer treated as an acronym.

## 2.1 Linux Kernel Observability

### Claim 2.1-A: Linux exposes several complementary instrumentation facilities

1. **Planned claim**

   Linux kernel observability is not provided by a single interface. The
   upstream tracing documentation describes multiple facilities, including
   ftrace, tracepoints, kprobes, and event-based tracing, whose visibility,
   stability, and attachment semantics differ.

2. **Source**

   Linux kernel documentation, *Linux Tracing Technologies*:
   <https://docs.kernel.org/trace/index.html>

3. **Placement**

   Section **2.1 Linux Kernel Observability**, opening paragraph.

4. **What the source supports**

   The tracing index is an authoritative inventory of tracing subsystems
   maintained with the kernel. It supports introducing kernel observability as
   an ecosystem of complementary mechanisms rather than as one homogeneous
   API.

5. **Limitations**

   The source does not provide a controlled performance comparison and should
   not be used to rank all mechanisms by overhead, safety, or suitability for
   security monitoring.

6. **Verified BibTeX proposal**

   Use `linux_tracing_index` from the verified entries below.

### Claim 2.1-B: Tracepoints are explicit static instrumentation sites

1. **Planned claim**

   A kernel tracepoint is an explicit static hook placed in kernel code. When a
   probe is registered, the callback is invoked in the execution context of the
   caller; when no probe is enabled, the tracepoint primarily retains the cost
   of a small conditional branch and associated metadata.

2. **Source**

   Linux kernel documentation, *Using the Linux Kernel Tracepoints*:
   <https://docs.kernel.org/trace/tracepoints.html>

3. **Placement**

   Section **2.1 Linux Kernel Observability**, paragraph comparing static and
   dynamic instrumentation.

4. **What the source supports**

   The document defines tracepoints as static probe points and explains their
   enabled and disabled execution behaviour.

5. **Limitations**

   “Low overhead” should not be turned into a universal numerical claim.
   Enabled cost depends on the registered probe, event rate, data copied, and
   target architecture. Tracepoints should also not be described as an
   immutable ABI unless a particular tracepoint is explicitly documented as
   such.

6. **Verified BibTeX proposal**

   Use `linux_tracepoints` from the verified entries below.

### Claim 2.1-C: Kprobes provide dynamic instrumentation with weaker coupling guarantees

1. **Planned claim**

   Kprobes can dynamically instrument many kernel instructions, while
   kretprobes observe function returns. This flexibility permits observation
   where no dedicated tracepoint exists, but it can couple a probe to internal
   symbols, signatures, or instruction locations that may change between
   kernels.

2. **Source**

   Linux kernel documentation, *Kernel Probes (Kprobes)*:
   <https://docs.kernel.org/trace/kprobes.html>

3. **Placement**

   Section **2.1 Linux Kernel Observability**, after the tracepoint discussion.

4. **What the source supports**

   The document defines kprobes and kretprobes, describes their registration,
   and states that kprobes can instrument almost any kernel instruction.

5. **Limitations**

   The portability sentence is a reasoned consequence of instrumenting
   internal code, not a claim that every kprobe is unstable. A probe on a
   long-lived symbol may work across many releases, while an internal or
   inlined function may not.

6. **Verified BibTeX proposal**

   Use `linux_kprobes` from the verified entries below.

### Claim 2.1-D: eBPF adds programmable event handling, not a universal observation guarantee

1. **Planned claim**

   In Linux, eBPF permits verified programs to execute at designated kernel
   hooks and therefore provides programmable processing close to observed
   events. Its visibility remains bounded by the hooks, program types, helper
   functions, privileges, and kernel features available on the target system.

2. **Sources**

   - Gbadamosi et al., *The eBPF Runtime in the Linux Kernel*:
     <https://doi.org/10.48550/arXiv.2410.00026>
   - Linux kernel documentation, *Program Types and ELF Sections*:
     <https://docs.kernel.org/bpf/libbpf/program_types.html>

3. **Placement**

   Section **2.1 Linux Kernel Observability**, transition into Section 2.2.

4. **What the sources support**

   The paper describes the Linux eBPF runtime and execution at designated
   hooks. The libbpf documentation demonstrates that program and attachment
   support is differentiated by program type, attach type, and ELF section.

5. **Limitations**

   This should not be written as “eBPF observes everything without kernel
   changes”. Instrumentation coverage is finite, target dependent, and subject
   to permission and configuration constraints.

6. **Verified BibTeX proposal**

   Use `gbadamosi2024ebpf_runtime` and `linux_libbpf_program_types`.

## 2.2.1 Execution Model, Verifier and JIT Compilation

### Claim 2.2.1-A: Modern BPF is a specified instruction set

1. **Planned claim**

   Modern BPF defines a register-based instruction set with specified
   instruction encodings and conformance groups. Executing that instruction
   set requires a runtime implementation, such as an interpreter or a compiler
   that translates BPF instructions into native instructions.

2. **Source**

   Thaler, *BPF Instruction Set Architecture (ISA)*, RFC 9669:
   <https://doi.org/10.17487/RFC9669>

3. **Placement**

   Section **2.2.1 Execution Model, Verifier and JIT Compilation**, opening
   paragraph.

4. **What the source supports**

   RFC 9669 standardises instruction encodings, registers, operations, and
   conformance groups, and explicitly identifies interpretation or compilation
   as necessary implementation components.

5. **Limitations**

   The RFC specifies the ISA, not every Linux runtime policy. Linux-specific
   verifier rules, helper availability, program types, and attachment APIs must
   be sourced from Linux documentation.

6. **Verified BibTeX proposal**

   Use `rfc9669_bpf_isa`.

### Claim 2.2.1-B: Verification precedes accepted program execution

1. **Planned claim**

   For programs loaded through the Linux BPF interface, the kernel verifier
   analyses the submitted instructions before accepting the program for
   execution.

2. **Sources**

   - Linux kernel documentation, *eBPF verifier*:
     <https://docs.kernel.org/bpf/verifier.html>
   - Linux manual pages, `bpf(2)`:
     <https://man7.org/linux/man-pages/man2/bpf.2.html>

3. **Placement**

   Section **2.2.1 Execution Model, Verifier and JIT Compilation**, verifier
   overview.

4. **What the sources support**

   The verifier documentation describes control-flow and state analysis. The
   `bpf(2)` interface documents program loading and the kernel's acceptance or
   rejection of a submitted program.

5. **Limitations**

   Verification should not be equated with a proof of the program's intended
   security policy or semantic correctness.

6. **Verified BibTeX proposal**

   Use `linux_ebpf_verifier` and `linux_bpf_syscall`.

### Claim 2.2.1-C: The verifier tracks abstract machine state and constrained pointers

1. **Planned claim**

   During verification, Linux explores feasible control-flow paths while
   tracking abstract register and stack states, including scalar ranges,
   pointer classes, offsets, and permitted memory regions. It also checks
   helper-call arguments against the calling constraints known for the
   relevant program type.

2. **Sources**

   - Linux kernel documentation, *eBPF verifier*:
     <https://docs.kernel.org/bpf/verifier.html>
   - Linux manual pages, `bpf-helpers(7)`:
     <https://man7.org/linux/man-pages/man7/bpf-helpers.7.html>

3. **Placement**

   Section **2.2.1 Execution Model, Verifier and JIT Compilation**, detailed
   verifier paragraph.

4. **What the sources support**

   The verifier document explains register-state, pointer, range, stack, and
   path tracking. The helper manual states that helper availability is
   restricted and depends on program type.

5. **Limitations**

   “Explores all paths” should be understood in terms of the verifier's state
   exploration, pruning, and complexity limits. It should not be presented as
   unconstrained exhaustive model checking of all possible system states.

6. **Verified BibTeX proposal**

   Use `linux_ebpf_verifier` and `linux_bpf_helpers`.

### Claim 2.2.1-D: Verifier acceptance is a safety gate, not a functional correctness proof

1. **Planned claim**

   Verifier acceptance establishes that a program satisfies the safety and
   interface constraints modelled by that kernel's verifier. It does not prove
   that the program implements its intended monitoring or security semantics,
   is free from logical errors, observes every relevant event, or will be
   accepted by a different kernel.

2. **Sources**

   - Gershuni et al., *Simple and Precise Static Analysis of Untrusted Linux
     Kernel Extensions*: <https://doi.org/10.1145/3314221.3314590>
   - Linux kernel documentation, *BPF Design Q&A*:
     <https://docs.kernel.org/bpf/bpf_design_QA.html>

3. **Placement**

   Section **2.2.1 Execution Model, Verifier and JIT Compilation**, verifier
   limitations paragraph.

4. **What the sources support**

   The peer-reviewed work frames eBPF verification as static analysis for
   kernel safety and discusses precision, scalability, and false-positive
   limitations. The kernel Q&A documents implementation and complexity limits
   that may cause rejection.

5. **Limitations**

   The 2019 paper describes the verifier state at that time; specific
   limitations such as loop support have since evolved. It remains suitable
   evidence for the distinction between safety analysis and functional
   correctness, but not for an up-to-date list of unsupported language
   constructs.

6. **Verified BibTeX proposal**

   Use `gershuni2019prevail` and `linux_bpf_design_qa`.

### Claim 2.2.1-E: Accepted BPF instructions may be interpreted or JIT compiled

1. **Planned claim**

   After verification, a Linux BPF program may execute through an interpreter
   or be translated by an architecture-specific just-in-time compiler into
   native instructions. The effective path depends on kernel configuration,
   architecture support, runtime policy, and hardening settings.

2. **Sources**

   - RFC 9669: <https://doi.org/10.17487/RFC9669>
   - Linux kernel documentation, `bpf_jit_enable` and `bpf_jit_harden`:
     <https://docs.kernel.org/admin-guide/sysctl/net.html#bpf-jit-enable>

3. **Placement**

   Section **2.2.1 Execution Model, Verifier and JIT Compilation**, final
   paragraph.

4. **What the sources support**

   RFC 9669 distinguishes interpretation from translation to native
   instructions. The kernel sysctl documentation identifies supported JIT
   architectures and runtime controls, including hardening options.

5. **Limitations**

   JIT compilation should not be described as universally enabled or
   invariably faster under every workload. The exact controls and defaults may
   differ across kernel versions and vendor configurations.

6. **Verified BibTeX proposal**

   Use `rfc9669_bpf_isa` and `linux_bpf_jit_sysctl`.

## 2.2.2 Program Types, Hooks and Attachment Mechanisms

### Claim 2.2.2-A: Program type, hook, and attachment mechanism are distinct concepts

1. **Planned claim**

   A **program type** is the kernel-declared class under which a BPF program is
   loaded and verified; it influences the expected context, permitted
   operations, helper availability, and return semantics. A **hook** is the
   kernel event or execution point at which a program may run. An
   **attachment mechanism** is the API and lifecycle operation that binds a
   loaded program to a compatible hook.

2. **Sources**

   - Linux kernel documentation, *Program Types and ELF Sections*:
     <https://docs.kernel.org/bpf/libbpf/program_types.html>
   - Linux kernel documentation, *BPF sk_lookup program*:
     <https://docs.kernel.org/bpf/prog_sk_lookup.html>
   - Linux manual pages, `bpf-helpers(7)`:
     <https://man7.org/linux/man-pages/man7/bpf-helpers.7.html>

3. **Placement**

   Section **2.2.2 Program Types, Hooks and Attachment Mechanisms**, opening
   definition paragraph.

4. **What the sources support**

   The libbpf table separately identifies program type, expected attach type,
   and ELF section. The sk_lookup documentation separately describes its
   program type, hook semantics, and `BPF_LINK_CREATE` attachment. The helper
   manual confirms that helper sets differ by program type.

5. **Limitations**

   These relationships are not always one-to-one. A program type may support
   several attachment forms, and similar hooks may be reached through
   different APIs. ELF section names are libbpf conventions, not themselves
   kernel hooks.

6. **Verified BibTeX proposal**

   Use `linux_libbpf_program_types`, `linux_bpf_sk_lookup`, and
   `linux_bpf_helpers`.

### Claim 2.2.2-B: Attachment APIs encode both compatibility and lifecycle

1. **Planned claim**

   Linux supports multiple BPF attachment paths, including perf-event-based
   tracing interfaces, dedicated `bpf()` attach commands, and BPF links.
   A BPF link represents an attachment as a kernel object whose lifetime can
   be managed through a file descriptor, but link support is not uniform
   across all program and hook combinations.

2. **Sources**

   - Linux kernel documentation, *BPF sk_lookup program*:
     <https://docs.kernel.org/bpf/prog_sk_lookup.html>
   - Linux kernel documentation, *BPF Iterators*:
     <https://docs.kernel.org/bpf/bpf_iterators.html>
   - Linux kernel documentation, *Program Types and ELF Sections*:
     <https://docs.kernel.org/bpf/libbpf/program_types.html>

3. **Placement**

   Section **2.2.2 Program Types, Hooks and Attachment Mechanisms**, attachment
   lifecycle paragraph.

4. **What the sources support**

   The sources provide concrete `BPF_LINK_CREATE` and `bpf_link_create()`
   examples and show that libbpf attachment conventions vary by program type.

5. **Limitations**

   The chapter should not imply that every supported attachment is represented
   by `bpf_link` on every target kernel. Older kernels and some interfaces use
   different ownership and detachment models.

6. **Verified BibTeX proposal**

   Use `linux_bpf_sk_lookup`, `linux_bpf_iterators`, and
   `linux_libbpf_program_types`.

### Claim 2.2.2-C: Static and dynamic tracing hooks have different compatibility properties

1. **Planned claim**

   Tracepoints expose named, explicit instrumentation sites, whereas kprobes
   dynamically target kernel instructions or functions. Consequently,
   tracepoints can reduce dependence on internal symbol placement, while
   kprobes offer broader placement flexibility at the cost of potentially
   stronger coupling to kernel implementation details.

2. **Sources**

   - <https://docs.kernel.org/trace/tracepoints.html>
   - <https://docs.kernel.org/trace/kprobes.html>

3. **Placement**

   Section **2.2.2 Program Types, Hooks and Attachment Mechanisms**, tracing
   examples.

4. **What the sources support**

   The two kernel documents define the placement model of each mechanism.

5. **Limitations**

   “Tracepoint” should not be used as a synonym for every tracing hook, and
   tracepoint presence or field stability must still be checked on the target
   kernel.

6. **Verified BibTeX proposal**

   Use `linux_tracepoints` and `linux_kprobes`.

## 2.2.3 Maps, Helpers and Kernel-to-Userspace Transport

### Claim 2.2.3-A: Maps are typed kernel objects used for shared state

1. **Planned claim**

   BPF maps are kernel-managed objects that provide storage with
   map-type-specific semantics. BPF programs can access maps through permitted
   operations, while user space can create and manipulate maps through the
   `bpf()` system call. This makes maps suitable for sharing configuration,
   state, counters, and references across program invocations and between
   kernel and user space.

2. **Source**

   Linux kernel documentation, *BPF maps*:
   <https://docs.kernel.org/bpf/maps.html>

3. **Placement**

   Section **2.2.3 Maps, Helpers and Kernel-to-Userspace Transport**, maps
   paragraph.

4. **What the source supports**

   The documentation defines maps as generic storage shared between kernel and
   user space, lists map types, and documents user-space operations through
   the `bpf()` syscall.

5. **Limitations**

   Maps should not be described as unrestricted shared memory. Their key,
   value, allocation, concurrency, update, and lifetime semantics vary by map
   type, and some specialised map types do not expose the usual
   lookup/update/delete model.

6. **Verified BibTeX proposal**

   Use `linux_bpf_maps`.

### Claim 2.2.3-B: Helpers are a constrained kernel-provided callable interface

1. **Planned claim**

   BPF helpers are kernel-provided functions through which a program performs
   operations that are not expressed directly by the instruction set, such as
   map access, time retrieval, controlled memory reads, or event submission.
   The available helper subset depends on program type and kernel support, and
   helper arguments are checked by the verifier.

2. **Sources**

   - Linux manual pages, `bpf-helpers(7)`:
     <https://man7.org/linux/man-pages/man7/bpf-helpers.7.html>
   - Linux kernel documentation, *eBPF verifier*:
     <https://docs.kernel.org/bpf/verifier.html>

3. **Placement**

   Section **2.2.3 Maps, Helpers and Kernel-to-Userspace Transport**, helpers
   paragraph.

4. **What the sources support**

   The helper manual describes the restricted helper interface and
   program-type-dependent availability. The verifier documentation explains
   argument and pointer validation.

5. **Limitations**

   Helpers are not a stable, universal function set shared identically by all
   kernels and program types. Availability should be probed or validated
   against the deployment kernel.

6. **Verified BibTeX proposal**

   Use `linux_bpf_helpers` and `linux_ebpf_verifier`.

### Claim 2.2.3-C: Perf-event output streams records through per-CPU perf infrastructure

1. **Planned claim**

   The `bpf_perf_event_output()` helper writes a raw record to a perf event
   referenced through a `BPF_MAP_TYPE_PERF_EVENT_ARRAY`. User space prepares
   perf-event file descriptors and consumes the resulting records, commonly
   through memory-mapped per-CPU buffers.

2. **Sources**

   - Linux manual pages, `bpf-helpers(7)`, entry for
     `bpf_perf_event_output()`:
     <https://man7.org/linux/man-pages/man7/bpf-helpers.7.html>
   - Linux kernel documentation, *Perf ring buffer*:
     <https://docs.kernel.org/userspace-api/perf_ring_buffer.html>

3. **Placement**

   Section **2.2.3 Maps, Helpers and Kernel-to-Userspace Transport**, perf
   buffer paragraph.

4. **What the sources support**

   The helper manual documents the required map and perf-event configuration.
   The perf documentation explains the memory-mapped ring-buffer protocol used
   to exchange records with user space.

5. **Limitations**

   “Perf buffer” is common ecosystem terminology rather than a distinct BPF
   map type. The actual BPF map is `BPF_MAP_TYPE_PERF_EVENT_ARRAY`, which
   connects programs to perf events and their buffers.

6. **Verified BibTeX proposal**

   Use `linux_bpf_helpers` and `linux_perf_ring_buffer`.

### Claim 2.2.3-D: The BPF ring buffer addresses ordering and memory-sharing limitations

1. **Planned claim**

   `BPF_MAP_TYPE_RINGBUF` provides a multi-producer, single-consumer transport
   that may be shared across CPUs. Its design was motivated by more efficient
   shared memory use and preservation of reservation order across CPUs,
   properties that per-CPU perf buffers do not inherently provide.

2. **Source**

   Linux kernel documentation, *BPF ring buffer*:
   <https://docs.kernel.org/bpf/ringbuf.html>

3. **Placement**

   Section **2.2.3 Maps, Helpers and Kernel-to-Userspace Transport**, ring
   buffer comparison.

4. **What the source supports**

   The kernel documentation explicitly states the design motivations, MPSC
   topology, cross-CPU ordering property, map representation, and
   reserve/commit and copy-based APIs.

5. **Limitations**

   The source does not justify saying that ring buffers are always faster or
   always preferable. Shared buffers may introduce contention, and transport
   choice depends on ordering, memory, event size, compatibility, and workload
   requirements.

6. **Verified BibTeX proposal**

   Use `linux_bpf_ringbuf`.

### Claim 2.2.3-E: Neither transport provides unbounded, blocking delivery

1. **Planned claim**

   Kernel-to-user-space event transport is capacity bounded. For the BPF ring
   buffer, reservation or output can fail when space is unavailable; perf
   buffers can also overrun when producers outpace consumption. Monitoring
   designs should therefore treat loss accounting, buffer sizing, and consumer
   throughput as correctness-relevant operational concerns.

2. **Sources**

   - Linux kernel documentation, *BPF ring buffer*:
     <https://docs.kernel.org/bpf/ringbuf.html>
   - Linux kernel documentation, *Perf ring buffer*:
     <https://docs.kernel.org/userspace-api/perf_ring_buffer.html>

3. **Placement**

   Section **2.2.3 Maps, Helpers and Kernel-to-Userspace Transport**, closing
   paragraph.

4. **What the sources support**

   The ring-buffer document states that reservation fails rather than blocks
   when capacity is unavailable. The perf-buffer documentation describes the
   producer/consumer head-tail protocol and overrun-related record semantics.

5. **Limitations**

   The sources support the possibility of loss, not an expected loss rate.
   Loss depends on workload, record size, topology, scheduling, buffer
   capacity, and consumer behaviour.

6. **Verified BibTeX proposal**

   Use `linux_bpf_ringbuf` and `linux_perf_ring_buffer`.

## 2.2.4 BTF and CO-RE Portability

### Claim 2.2.4-A: BTF carries compact type and source-related metadata

1. **Planned claim**

   BPF Type Format (BTF) encodes type and string information and can also be
   associated with function and line information. Within BPF ELF objects, the
   `.BTF` section contains type data, while `.BTF.ext` can contain function
   information, line information, and CO-RE relocation records.

2. **Source**

   Linux kernel documentation, *BPF Type Format (BTF)*:
   <https://docs.kernel.org/bpf/btf.html>

3. **Placement**

   Section **2.2.4 BTF and CO-RE Portability**, BTF definition.

4. **What the source supports**

   The document specifies BTF encodings, kernel interfaces, ELF sections, and
   the organisation of CO-RE relocation records.

5. **Limitations**

   BTF should not be described as equivalent to complete DWARF debugging
   information or as automatically present for every kernel, module, or
   user-space binary.

6. **Verified BibTeX proposal**

   Use `linux_btf`.

### Claim 2.2.4-B: CO-RE combines compiler metadata, BTF, and loader relocation

1. **Planned claim**

   Compile Once - Run Everywhere (CO-RE) combines compiler-emitted relocation
   metadata, BTF type information, and libbpf loader processing. At load time,
   CO-RE relocations update relevant BPF instruction offsets or immediates
   according to type information from the target kernel.

2. **Sources**

   - Linux kernel documentation, *BPF LLVM Relocations*:
     <https://docs.kernel.org/bpf/llvm_reloc.html>
   - Linux kernel documentation, *libbpf Overview*:
     <https://docs.kernel.org/bpf/libbpf/libbpf_overview.html>
   - Linux kernel documentation, *BPF Type Format (BTF)*:
     <https://docs.kernel.org/bpf/btf.html>

3. **Placement**

   Section **2.2.4 BTF and CO-RE Portability**, CO-RE mechanism.

4. **What the sources support**

   The relocation documentation explains compiler-preserved access metadata
   and load-time instruction patching. The libbpf overview identifies BTF,
   compiler support, and libbpf as the principal CO-RE components. The BTF
   specification defines the relocation records stored in `.BTF.ext`.

5. **Limitations**

   “Run Everywhere” is a design objective, not an unconditional compatibility
   guarantee. The phrase should always be followed by the target prerequisites
   and limits described in the next claim.

6. **Verified BibTeX proposal**

   Use `linux_bpf_llvm_relocations`, `linux_libbpf_overview`, and `linux_btf`.

### Claim 2.2.4-C: CO-RE addresses structural variation but not all runtime compatibility

1. **Planned claim**

   CO-RE can adapt supported type, field, size, offset, and existence
   differences when suitable target BTF is available. It does not by itself
   create missing hooks, program types, helpers, kfuncs, attachment APIs, or
   privileges, nor does it guarantee that a program will satisfy the target
   kernel's verifier.

2. **Sources**

   - <https://docs.kernel.org/bpf/llvm_reloc.html>
   - <https://docs.kernel.org/bpf/libbpf/libbpf_overview.html>
   - <https://docs.kernel.org/bpf/libbpf/program_types.html>
   - <https://docs.kernel.org/bpf/verifier.html>

3. **Placement**

   Section **2.2.4 BTF and CO-RE Portability**, portability limits.

4. **What the sources support**

   The CO-RE sources delimit relocation to type-aware load-time adaptation.
   The program-type and verifier documents separately define capabilities and
   acceptance constraints that relocation does not supply.

5. **Limitations**

   This is a synthesis across interfaces rather than a quotation from one
   source. Avoid reducing CO-RE to field-offset rewriting alone because current
   relocation kinds cover a broader set of type and enum queries. Conversely,
   avoid calling the resulting object universally portable.

6. **Verified BibTeX proposal**

   Use `linux_bpf_llvm_relocations`, `linux_libbpf_overview`,
   `linux_libbpf_program_types`, and `linux_ebpf_verifier`.

### Claim 2.2.4-D: Enterprise backports make version-only capability inference unreliable

1. **Planned claim**

   Enterprise Linux vendors may retain an older upstream base version while
   backporting selected fixes and features. Consequently, an upstream kernel
   version string alone is insufficient evidence that a particular BPF
   program type, helper, map type, verifier behaviour, BTF feature, or
   attachment mechanism is present or absent.

2. **Sources**

   - Red Hat, *What is backporting and how does it affect Red Hat Enterprise
     Linux?*: <https://access.redhat.com/solutions/57665>
   - Linux manual pages, `bpf-helpers(7)`, implementation note:
     <https://man7.org/linux/man-pages/man7/bpf-helpers.7.html>

3. **Placement**

   Section **2.2.4 BTF and CO-RE Portability**, final paragraph.

4. **What the sources support**

   Red Hat explains that patches are applied to older package versions and
   that upstream version-number comparisons can therefore be misleading. The
   helper manual recommends inspecting the actual kernel and using
   `bpftool feature probe` because BPF capabilities evolve.

5. **Limitations**

   Backporting does not imply that every newer upstream BPF feature is
   available. The defensible conclusion is that capabilities must be checked
   on the actual vendor kernel through documentation, configuration,
   feature probing, and load-and-attach tests.

6. **Verified BibTeX proposal**

   Use `redhat_backporting` and `linux_bpf_helpers`.

## Recommended conceptual sequence for the chapter

The evidence above supports the following narrative order:

1. Introduce Linux observability as a set of complementary kernel facilities.
2. Contrast explicit static tracepoints with dynamic kprobe instrumentation.
3. Introduce modern BPF as a programmable execution environment attached to
   designated hooks.
4. Explain load, verification, and interpreter/JIT execution without treating
   verifier acceptance as functional correctness.
5. Define program type, hook, attach type, attachment API, and ELF section name
   as related but distinct concepts.
6. Explain maps and helpers before presenting event transport.
7. Compare perf-event transport and `BPF_MAP_TYPE_RINGBUF` by semantics rather
   than by an unsupported claim of universal performance superiority.
8. Present BTF and CO-RE as load-time structural adaptation, followed
   immediately by their prerequisites and limits.
9. Close with enterprise backports and the need to verify capabilities on the
   actual deployment kernel.

## Wording to avoid

- “eBPF stands for Extended Berkeley Packet Filter” as a current formal
  definition.
- “The verifier proves that an eBPF program is correct.”
- “Every possible execution is exhaustively tested by the verifier.”
- “eBPF programs always execute as native machine code.”
- “A program type is the hook.”
- “An ELF section name is a kernel attachment point.”
- “Perf buffer is a dedicated BPF map type.”
- “Ring buffer is always faster than perf buffer.”
- “CO-RE makes one BPF binary work on every Linux kernel.”
- “Kernel 4.x cannot contain a BPF feature introduced upstream in kernel 5.x.”

## Verified BibTeX proposals

The URLs below were checked on 2026-07-27. For continuously maintained kernel
documentation, `year` records the access year rather than asserting a discrete
publication release.

```bibtex
@online{linux_tracing_index,
  author  = {{The Linux Kernel Documentation}},
  title   = {Linux Tracing Technologies},
  year    = {2026},
  url     = {https://docs.kernel.org/trace/index.html},
  urldate = {2026-07-27}
}

@online{linux_tracepoints,
  author  = {{The Linux Kernel Documentation}},
  title   = {Using the Linux Kernel Tracepoints},
  year    = {2026},
  url     = {https://docs.kernel.org/trace/tracepoints.html},
  urldate = {2026-07-27}
}

@online{linux_kprobes,
  author  = {{The Linux Kernel Documentation}},
  title   = {Kernel Probes (Kprobes)},
  year    = {2026},
  url     = {https://docs.kernel.org/trace/kprobes.html},
  urldate = {2026-07-27}
}

@article{gbadamosi2024ebpf_runtime,
  author        = {Gbadamosi, Bolaji and Leonardi, Luigi and Pulls, Tobias and
                   H{\o}iland-J{\o}rgensen, Toke and Ferlin-Reiter, Simone and
                   Sorce, Simo and Brunstr{\"o}m, Anna},
  title         = {The eBPF Runtime in the Linux Kernel},
  year          = {2024},
  journal       = {arXiv preprint arXiv:2410.00026},
  doi           = {10.48550/arXiv.2410.00026},
  url           = {https://arxiv.org/abs/2410.00026}
}

@techreport{rfc9669_bpf_isa,
  author      = {Thaler, Dave},
  title       = {BPF Instruction Set Architecture (ISA)},
  institution = {Internet Engineering Task Force},
  type        = {RFC},
  number      = {9669},
  year        = {2024},
  month       = oct,
  doi         = {10.17487/RFC9669},
  url         = {https://www.rfc-editor.org/info/rfc9669}
}

@online{linux_ebpf_verifier,
  author  = {{The Linux Kernel Documentation}},
  title   = {eBPF Verifier},
  year    = {2026},
  url     = {https://docs.kernel.org/bpf/verifier.html},
  urldate = {2026-07-27}
}

@manual{linux_bpf_syscall,
  author  = {{Linux man-pages project}},
  title   = {bpf(2) --- Perform a Command on an Extended BPF Map or Program},
  year    = {2026},
  url     = {https://man7.org/linux/man-pages/man2/bpf.2.html},
  urldate = {2026-07-27}
}

@manual{linux_bpf_helpers,
  author  = {{Linux man-pages project}},
  title   = {bpf-helpers(7) --- List of eBPF Helper Functions},
  year    = {2026},
  url     = {https://man7.org/linux/man-pages/man7/bpf-helpers.7.html},
  urldate = {2026-07-27}
}

@inproceedings{gershuni2019prevail,
  author    = {Gershuni, Elazar and Amit, Nadav and Gurfinkel, Arie and
               Narodytska, Nina and Navas, Jorge A. and Rinetzky, Noam and
               Ryzhyk, Leonid and Sagiv, Mooly},
  title     = {Simple and Precise Static Analysis of Untrusted Linux Kernel
               Extensions},
  booktitle = {Proceedings of the 40th ACM SIGPLAN Conference on Programming
               Language Design and Implementation},
  series    = {PLDI '19},
  year      = {2019},
  pages     = {1069--1084},
  publisher = {Association for Computing Machinery},
  doi       = {10.1145/3314221.3314590},
  url       = {https://doi.org/10.1145/3314221.3314590}
}

@online{linux_bpf_design_qa,
  author  = {{The Linux Kernel Documentation}},
  title   = {BPF Design Q\&A},
  year    = {2026},
  url     = {https://docs.kernel.org/bpf/bpf_design_QA.html},
  urldate = {2026-07-27}
}

@online{linux_bpf_jit_sysctl,
  author  = {{The Linux Kernel Documentation}},
  title   = {Documentation for /proc/sys/net/: BPF JIT Controls},
  year    = {2026},
  url     = {https://docs.kernel.org/admin-guide/sysctl/net.html#bpf-jit-enable},
  urldate = {2026-07-27}
}

@online{linux_libbpf_program_types,
  author  = {{The Linux Kernel Documentation}},
  title   = {Program Types and ELF Sections},
  year    = {2026},
  url     = {https://docs.kernel.org/bpf/libbpf/program_types.html},
  urldate = {2026-07-27}
}

@online{linux_bpf_sk_lookup,
  author  = {{The Linux Kernel Documentation}},
  title   = {BPF sk\_lookup Program},
  year    = {2026},
  url     = {https://docs.kernel.org/bpf/prog_sk_lookup.html},
  urldate = {2026-07-27}
}

@online{linux_bpf_iterators,
  author  = {{The Linux Kernel Documentation}},
  title   = {BPF Iterators},
  year    = {2026},
  url     = {https://docs.kernel.org/bpf/bpf_iterators.html},
  urldate = {2026-07-27}
}

@online{linux_bpf_maps,
  author  = {{The Linux Kernel Documentation}},
  title   = {BPF Maps},
  year    = {2026},
  url     = {https://docs.kernel.org/bpf/maps.html},
  urldate = {2026-07-27}
}

@online{linux_perf_ring_buffer,
  author  = {{The Linux Kernel Documentation}},
  title   = {Perf Ring Buffer},
  year    = {2026},
  url     = {https://docs.kernel.org/userspace-api/perf_ring_buffer.html},
  urldate = {2026-07-27}
}

@online{linux_bpf_ringbuf,
  author  = {{The Linux Kernel Documentation}},
  title   = {BPF Ring Buffer},
  year    = {2026},
  url     = {https://docs.kernel.org/bpf/ringbuf.html},
  urldate = {2026-07-27}
}

@online{linux_btf,
  author  = {{The Linux Kernel Documentation}},
  title   = {BPF Type Format (BTF)},
  year    = {2026},
  url     = {https://docs.kernel.org/bpf/btf.html},
  urldate = {2026-07-27}
}

@online{linux_bpf_llvm_relocations,
  author  = {{The Linux Kernel Documentation}},
  title   = {BPF LLVM Relocations},
  year    = {2026},
  url     = {https://docs.kernel.org/bpf/llvm_reloc.html},
  urldate = {2026-07-27}
}

@online{linux_libbpf_overview,
  author  = {{The Linux Kernel Documentation}},
  title   = {libbpf Overview},
  year    = {2026},
  url     = {https://docs.kernel.org/bpf/libbpf/libbpf_overview.html},
  urldate = {2026-07-27}
}

@online{redhat_backporting,
  author  = {{Red Hat}},
  title   = {What Is Backporting and How Does It Affect Red Hat Enterprise
             Linux?},
  year    = {2025},
  url     = {https://access.redhat.com/solutions/57665},
  urldate = {2026-07-27},
  note    = {Solution verified; updated 2025-06-10}
}
```

## Final source-quality notes

- RFC 9669 is the strongest source for modern naming and the portable BPF ISA.
- Linux kernel documentation is the primary source for Linux verifier,
  program-type, map, transport, BTF, CO-RE, and attachment semantics.
- The PREVAIL paper is valuable for explaining static-analysis limits, but
  claims tied to the 2019 feature set must be updated with current kernel
  documentation.
- The arXiv runtime paper is a comprehensive technical synthesis, but it is not
  a replacement for the kernel ABI documentation where exact Linux behaviour
  is at issue.
- The Red Hat source supports the methodological claim about backports and
  version strings. Actual feature availability still requires examination of
  the deployed kernel.
