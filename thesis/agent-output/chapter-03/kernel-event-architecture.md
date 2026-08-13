# Chapter 3 Kernel-Space and Event-Contract Architecture

## Purpose and Evidence Baseline

This dossier reconstructs the kernel-space collection architecture and the
kernel-to-user-space event contract for Chapter 3. It describes design
responsibilities and boundaries rather than providing a source-code
walkthrough. Concrete function names, structure layouts, map offsets, verifier
workarounds, command-line flags, and test procedures should be deferred to
Chapter 4.

The analysis is based on the current implementation at commit `163ad72`
(`fix(ebpf): preserve argument offsets in long event records`), together with
the latest project documentation and tests. The working tree currently contains
an uncommitted change that reclassifies several networking support probes as
public events. That change is not treated as part of the stable architecture:
it makes the probe-registry consistency test fail because those names do not
have corresponding decoder specifications. The committed classification of
those programs as internal dependencies is therefore the reliable baseline for
this chapter.

## 1. Kernel-Space Responsibilities and the User-Space Boundary

The architecture assigns the kernel side a deliberately restricted role. eBPF
programs observe selected kernel execution points, extract the minimum context
and arguments required by the corresponding logical event, apply the available
early rejection checks, serialise the result into a shared binary format, and
submit it to user space. They may also retain short-lived state when two
physical probes are required to construct one event, such as preserving syscall
entry arguments until the return value is available.

The Go runtime owns the more changeable and semantically rich responsibilities:

- selecting public events and resolving their probe dependencies;
- loading the eBPF object and attaching only the required programs;
- receiving and decoding binary records;
- resolving numeric event identifiers through the event registry;
- normalising typed arguments into the user-space event model;
- applying user-space event selection and policy scope;
- evaluating point and collective detectors;
- producing raw event output, alert output, and operational logs.

This boundary is motivated by both compatibility and evolvability. Keeping
kernel programs focused on bounded collection reduces verifier complexity and
avoids embedding the full policy and detection model in eBPF. Conversely,
keeping policy and detector semantics in Go permits those mechanisms to evolve
without recompiling each kernel program. The boundary is not absolute:
attach-time probe selection and the optional UID guard provide early reduction,
while a limited amount of kernel state supports multi-probe event construction.
Nevertheless, the current implementation is not a kernel-resident policy
engine and does not enforce security decisions.

## 2. Hooks, Support Probes, Dependencies, and Logical Events

### 2.1 Hook-selection criteria

The implementation uses several attachment mechanisms because no single hook
type provides the required semantics across the supported event domains. Hook
selection should be explained through four criteria:

1. **Semantic position.** The hook must observe the relevant operation at a
   point where the required context exists. Process-lifecycle tracepoints are
   suitable for state transitions, whereas security-related kernel functions
   can expose resolved kernel objects or mediation-stage information.
2. **Need for an outcome.** If the success or failure of an operation is
   essential, an entry-only hook is insufficient. A paired entry/exit syscall
   design or a kprobe/kretprobe pair can preserve the request and add the return
   value before emitting a single event.
3. **Target-kernel availability.** A hook is usable only if the concrete Rocky
   Linux kernel exposes the required tracepoint, symbol, attachment mechanism,
   and compatible calling convention. The upstream version number is not
   sufficient evidence because enterprise kernels include vendor backports and
   ABI adaptations.
4. **Cost and robustness.** A dedicated tracepoint is preferred when it offers
   the required semantics on the target. Generic high-frequency hooks can be
   useful as shared infrastructure, but they increase the amount of filtering
   performed on hot paths. Kprobes provide valuable coverage but are coupled
   more closely to kernel symbols and function signatures.

The resulting design combines raw tracepoints, tracepoints, kprobes,
kretprobes, and cgroup-skb programs. Chapter 3 should group their purpose by
domain rather than enumerate every implemented hook.

### 2.2 Four distinct architectural concepts

The following terms must remain distinct:

- A **kernel hook** is the physical execution point exposed by the kernel, such
  as a tracepoint, function entry, function return, or cgroup attachment point.
- An **eBPF probe** is one compiled program attached to a hook through a
  particular attachment mechanism.
- A **support probe** is an internal program that collects or preserves context
  needed by another probe. It is attachable, but it is not a stable public
  event and has no independent decoder contract.
- A **logical event** is the normalised observation exposed to user space. It
  has a stable event identifier, name, and typed argument schema.

These concepts are not in one-to-one correspondence. Several syscall variants
or entry/exit probes may produce the same logical event. Conversely, selecting
one public event may imply the attachment of multiple internal probes. This
distinction prevents implementation mechanics from leaking into the public
event API and prevents users from selecting support programs whose output is
not independently decodable.

The probe registry is therefore more than a list of hooks. It records the
public event associated with a program, its attachment kind, and any
dependencies activated by selecting that event. A separate event registry
defines the user-space identifier, name, and argument schema. Registry tests
protect the invariant that every public probe has a decoder specification,
while internal probes are explicitly exempt because they are dependencies
rather than event producers.

## 3. Attach-Time Selection and Minimal Kernel Filtering

### 3.1 Event-driven probe selection

Event selection begins from public logical event names. User inclusion and
exclusion choices are validated against the public registry, with exclusions
taking precedence. The runtime then expands the selected event set with any
internal probe dependencies. Before the eBPF object is loaded, autoload is
disabled for registered programs outside this resolved set. Only the remaining
programs are loaded, verified, and subsequently attached.

This is stronger than filtering unwanted records after decoding. It reduces the
number of active hooks and avoids loading and verifying unrelated registered
programs. It does not, however, remove maps or other shared object contents, and
it cannot prevent activity from an unregistered program that might remain
autoloaded. The design claim should consequently be phrased as selective
loading and attachment of programs represented in the authoritative probe
registry, not as arbitrary object-level dead-code elimination.

Policy configuration can narrow the public event domain before this selection
when explicit allow rules determine the required events. Detector loading
remains separate from probe dependency resolution.

### 3.2 Optional UID guard

The current kernel-side filter is intentionally minimal. User space writes an
enabled flag and one UID value into the shared configuration map. The common
event initialisation path reads the effective UID associated with the current
task and stops processing when it does not equal the configured value. The
check occurs before full context enrichment, argument capture, and event
submission, so rejected observations avoid most of the per-event work.

This mechanism is an exact-UID inclusion guard. It is not a general policy
language, does not evaluate detector conditions, and does not express ranges,
sets, namespaces, or workload identities. Richer policy and detector
evaluation remains in Go. Chapter 3 may present it as an example of optional
early noise reduction while preserving a stable user-space detection model.

## 4. Conceptual Binary Event Contract

Every public event is transmitted as a compact record with two conceptual
parts:

1. a common context header describing the event and the task that generated it;
2. an indexed sequence of typed event arguments.

The common context carries the event timestamp, process and thread identities
in both visible and host forms, parent identity, effective UID, namespace and
cgroup context, command name, event identifier, associated syscall metadata,
CPU identity, and reserved policy-related fields. This shared prefix gives
different event families a consistent identity and provenance model.

Arguments are event-specific. Each encoded argument carries its schema index,
followed by a payload whose representation is determined by the corresponding
user-space event specification. The typed model supports fixed-width scalar
values, bounded strings and byte sequences, string arrays, compact
null-delimited argument arrays, socket addresses, and credential snapshots.
Only successfully captured argument records are included, and user space
reconstructs their declared order through the argument indices.

The contract is shared rather than self-describing. The kernel producer writes
an event identifier and indexed values; the Go event registry supplies the
names and expected types. This keeps records compact but creates a strict
compatibility obligation:

- C and Go must agree on the common context layout;
- numeric event identifiers must remain aligned;
- argument indices and encodings must match the event schema;
- size limits and byte order must be interpreted consistently.

Unknown identifiers and malformed payloads are treated as decoding errors
rather than silently accepted. This makes registry drift visible. The current
implementation also includes a regression test for argument records extending
beyond 256 bytes, following a correction to kernel-side offset handling for the
non-power-of-two argument buffer.

Chapter 3 should retain this conceptual description. The exact 136-byte
context, field offsets, concrete type constants, maximum sizes, and the
long-record verifier-safe bounds correction belong in Chapter 4.

## 5. Perf-Buffer Transport and Failure Boundaries

The operational event path uses a BPF perf-event array. Each eBPF producer
submits the common context and only the populated portion of its argument
buffer on the current CPU. The Go runtime initialises a perf-buffer reader for
the shared event map and consumes records from a finite channel before
decoding them.

**The current runtime does not implement BPF ring-buffer transport.** Repository
searches find the ring-buffer map-type constants only in generated kernel type
definitions; there is no ring-buffer map used for the event stream, no
ring-buffer submission helper, and no Go ring-buffer reader. The ring buffer may
be discussed in Chapter 2 as an alternative transport, but Chapter 3 must
describe the perf buffer as the implemented design.

The transport has explicit failure boundaries:

- submission from eBPF may fail, for example because the per-CPU buffer cannot
  accept another record;
- the perf-buffer consumer can report a count of lost records, which the
  runtime logs but cannot reconstruct;
- records are processed after crossing a finite kernel/user-space boundary, so
  sustained producer rates can exceed consumer capacity;
- malformed or schema-incompatible records fail during decoding;
- ordering is naturally meaningful within a local stream, but the design must
  not claim a total ordering across CPUs;
- the current path does not provide durable queuing, retransmission, or
  exactly-once delivery.

Optional compile-time metrics can count submission attempts and helper
failures, whereas the standard runtime observes perf-buffer loss notifications.
These are related but not identical signals and should not be conflated. For
the thesis, loss and decode failures are observability boundaries of the
monitoring pipeline rather than evidence that the originating kernel operation
did not occur.

## 6. Enterprise-Kernel Compatibility Decisions

The reference environment is Rocky Linux 8.10 with kernel
`4.18.0-553.137.1.el8_10.x86_64` on x86-64. Compatibility is established
against that concrete kernel, not inferred from upstream Linux 4.18.

The design combines the following measures:

- BTF and CO-RE are used to relocate accesses to kernel data structures where
  possible.
- Flavoured structure definitions accommodate known layout differences,
  including Red Hat-specific variants.
- Missing macros and constants that are not represented in BTF are supplied
  explicitly.
- Field, type, and helper-existence checks select compatible access paths where
  practical.
- Architecture-specific syscall identifiers, register access, and the x86-64
  calling convention are treated as explicit assumptions.
- Bounded loops, bounded copies, explicit size checks, and verifier-visible
  limits are used to satisfy the older target verifier.
- Fallback logic is required where a newer helper is unavailable, including
  locating saved user registers from task state under documented target
  assumptions.
- Dedicated tracepoints and kprobe symbols are validated on the target at load
  or attachment time. Cgroup-skb monitoring additionally requires an available
  cgroup-v2 hierarchy and sufficient privileges.

CO-RE reduces dependence on compile-time structure offsets; it does not create
missing hooks, helpers, program types, attachment mechanisms, BTF information,
permissions, or verifier capabilities. Furthermore, kprobes remain dependent
on symbol availability and compatible function signatures. The current
implementation should therefore be described as target-aware and
CO-RE-assisted, not universally portable.

## 7. Proposed Chapter 3 Diagrams

### Diagram A: Responsibility boundary and end-to-end event flow

Use a left-to-right diagram divided into three vertical zones.

**Zone 1: Linux kernel**

- `Selected kernel hook`
- `eBPF probe`
- decision diamond: `UID guard enabled and UID mismatched?`
- `Context and typed argument capture`
- `Binary record serialisation`
- `Perf-event array`

The rejection branch from the decision diamond terminates at `Drop before
submission`. The accepted branch continues through capture and serialisation.

**Zone 2: Go runtime**

- `Perf-buffer reader`
- `Binary decoder`
- `Event registry and normalised event`
- `Policy event scope`
- `Detector dispatch`

The event registry should be drawn beside the decoder with a bidirectional
contract label, because it supplies the ID-to-schema interpretation rather than
being another sequential transformation.

**Zone 3: Presentation**

- branch to `Raw event output`
- branch to `Alert with evidence and ATT&CK metadata`
- separate `Operational logging` lane receiving lifecycle, loss, attachment,
  and decoding diagnostics from the Go runtime.

Dashed control arrows should run from `Runtime configuration` to `Probe
selection/autoload` in the kernel loading path and to the `UID guard`
configuration. This separates control-plane selection from the event data
plane.

### Diagram B: Physical probes, dependencies, and one logical event

Use a two-part conceptual mapping diagram.

The left part shows a selected public event expanding through a dependency
resolver:

```text
Public logical event
        |
        v
Probe registry and dependency resolution
        |--------------------|
        v                    v
Public producer probe   Internal support probe(s)
        |                    |
        |------ shared state-|
        v
Compact binary record with one public event ID
```

The right part shows multiple physical probes converging on one event:

```text
Entry probe -- stores request arguments --\
                                           > logical operation event
Exit probe --- adds operation result ------/
```

A note below the figure should state: “Attachment units and public event units
are deliberately distinct. Internal probes are not selectable event schemas.”
The figure should use generic labels rather than naming individual hooks.

## 8. Claim-to-Code Matrix for Sections 3.2, 3.3, and 3.4

| Chapter section | Architectural claim | Primary implementation evidence | Qualification for Chapter 3 |
|---|---|---|---|
| 3.2.1 | Kernel space collects and serialises bounded observations; Go owns lifecycle, decoding, policy, detection, and output. | `pkg/ebpf/c/common/context.h`; `pkg/ebpf/c/common/buffer.h`; `pkg/ebpf/project.go`; `pkg/bufferdecoder`; `pkg/events` | Describe responsibilities, not functions or packages. Policy-related fields in the event context do not establish a complete kernel policy engine. |
| 3.2.1 | Detection logic can evolve independently of individual eBPF probes. | `pkg/detectors`; `pkg/policy`; `pkg/ebpf/project.go` | This is a design benefit of the boundary, not a claim that the binary contract can change without coordinated C/Go updates. |
| 3.2.2 | The data path is hook, eBPF capture, binary record, perf buffer, decoder, normalised event, policy, detector, and output. | `pkg/ebpf/c/project.bpf.c`; `pkg/ebpf/c/common/buffer.h`; `pkg/ebpf/project.go:126`; `pkg/ebpf/project.go:201`; `pkg/ebpf/project.go:255` | Operational logging is a separate diagnostic path and should be shown separately. |
| 3.2.2 | Only selected registered programs are loaded and attached. | `pkg/ebpf/probes/probes.go:882`; `pkg/ebpf/probes/probes.go:982`; `pkg/ebpf/project.go:159`; `pkg/ebpf/project.go:362` | Scope this to programs represented in the registry. |
| 3.3.1 | Hook type is selected according to semantics, return-value needs, target support, and cost. | Probe kinds in `pkg/ebpf/probes/probes.go:11`; attachment routing at `pkg/ebpf/probes/probes.go:1037`; entry/exit helpers in `pkg/ebpf/c/common/probes.h` | The criteria are reconstructed from established patterns and documentation; do not present them as automatic runtime decisions. |
| 3.3.1 | Physical hooks and logical events are not one-to-one. | `Spec` and `impliedBy` in `pkg/ebpf/probes/probes.go:23`; multi-probe registrations; internal support registrations | Avoid enumerating the complete hook inventory in Chapter 3. |
| 3.3.1 | Internal probes are dependencies rather than public event schemas. | `EmitsEvent` at `pkg/ebpf/probes/probes.go:75`; registry comments at `pkg/ebpf/probes/probes.go:85`; committed networking dependency entries | The current uncommitted removal of several `internal` markers fails the registry test and must not be treated as stable design. |
| 3.3.2 | Event selection is resolved before load and attachment. | `pkg/ebpf/probes/probes.go:882`; `pkg/ebpf/probes/probes.go:982`; `pkg/ebpf/project.go:159` | This is selective autoload and attachment, not post-decode filtering alone. |
| 3.3.2 | An optional exact-UID guard can reject events before expensive capture. | `pkg/ebpf/c/types.h:471`; `pkg/ebpf/c/common/context.h:117`; `pkg/ebpf/c/common/context.h:137`; `pkg/ebpf/project.go:391` | It compares the current effective UID with one configured UID. It is not a general kernel policy engine. |
| 3.3.2 | Cgroup-skb probes require additional attachment context. | `pkg/ebpf/probes/cgroupv2.go:19`; `pkg/ebpf/probes/cgroupv2.go:67`; cgroup probe kinds and dependencies in `probes.go` | Describe only as an attachment requirement; do not equate local cgroup context with Kubernetes workload identity. |
| 3.4.1 | Every event starts with a common context followed by indexed typed arguments. | `pkg/ebpf/c/types.h:10`; `pkg/ebpf/c/types.h:29`; `pkg/ebpf/c/types.h:510`; `pkg/bufferdecoder/protocol.go:39`; `pkg/bufferdecoder/eventsreader.go:15` | Keep the exact byte layout and field offsets for Chapter 4. |
| 3.4.1 | Event IDs and schemas form an explicit shared producer-consumer contract. | `pkg/events/ids.go`; `pkg/events/spec.go:27`; `pkg/events/spec.go:871`; `pkg/bufferdecoder/eventsreader.go:44` | The record is compact, not fully self-describing. Unknown IDs fail decoding intentionally. |
| 3.4.1 | The typed model supports scalars and bounded structured payloads. | `pkg/events/data`; `pkg/bufferdecoder/eventsreader.go:100`; `pkg/bufferdecoder/protocol.go` | Mention type categories conceptually; defer encodings and limits. |
| 3.4.1 | Long argument records are protected by a regression-tested bounds correction. | Commit `163ad72`; `pkg/ebpf/c/common/buffer.h`; `pkg/bufferdecoder/eventsreader_test.go:55` | Use as Chapter 4 evidence of contract validation, not as a Chapter 3 design detail. |
| 3.4.2 | The implemented runtime transport is a perf buffer. | `pkg/ebpf/c/maps.h:690`; `pkg/ebpf/c/common/buffer.h:732`; `pkg/ebpf/project.go:177` | State explicitly that BPF ring-buffer transport is not implemented. |
| 3.4.2 | Transport loss is observable but not recoverable. | Lost channel handling at `pkg/ebpf/project.go:220`; optional submission metrics in `pkg/ebpf/c/common/buffer.h` | Perf loss notifications and eBPF helper return metrics are distinct; there is no retransmission or durable queue. |
| 3.4.2 | Decoder failures expose malformed records and registry drift. | `pkg/bufferdecoder/eventsreader.go:44`; `pkg/bufferdecoder/eventsreader.go:88`; decoder tests | Such failures describe observability limits and do not imply that the kernel operation failed. |

## 9. Details to Defer to Chapter 4

The following material is important implementation evidence but would make
Chapter 3 read as a code walkthrough:

- concrete file, package, function, structure, map, and eBPF section names;
- the complete hook inventory and event identifier table;
- exact probe-registry entries and dependency lists;
- command-line flags for event selection and UID filtering;
- the precise configuration-map byte offsets used by Go;
- the 136-byte common context layout and every field offset;
- argument type constants, maximum record sizes, byte order, and specialised
  decoder routines;
- the entry-state map and the mechanics of each entry/exit probe pair;
- perf-buffer channel capacities, initialisation calls, and cleanup methods;
- the long-record bit-mask defect, verifier-visible bounds fix, and regression
  test construction;
- architecture macros, x86 register extraction, stack arithmetic, and
  compatibility flavour structures;
- cgroup-v2 mount discovery, temporary mounting, and teardown;
- attachment errors for individual symbols or tracepoints;
- operational commands and target-host validation procedures;
- the unresolved uncommitted networking registry reclassification.

Chapter 3 should instead establish the architectural reasons for selective
attachment, bounded kernel collection, an explicit binary contract,
perf-buffer transport, and target-aware compatibility. Chapter 4 can then show
how those decisions are realised and tested.

