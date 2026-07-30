# Go User-Space Runtime Architecture Evidence for Chapter 3

## Scope and audited baseline

This report reconstructs the current Go user-space architecture from the thesis
sources, project documentation, and implementation. It is an evidence dossier
for Sections 3.2 and 3.5, not proposed thesis prose and not a substitute for the
implementation discussion in Chapter 4.

The implementation audit was performed on branch
`feature/composite-collective-correlation` at commit `163ad72`
(`fix(ebpf): preserve argument offsets in long event records`). The worktree
also contained unrelated or uncommitted changes to the Makefile, README,
benchmark scripts, the probe registry, and a policy file. Those changes are
discussed separately where they affect architectural invariants; they must not
be presented as a stable feature of the audited commit.

The runtime is written in Go and uses `libbpfgo` as its binding to libbpf. This
is a suitable engineering choice because Go provides typed data structures,
explicit error handling, interfaces, contexts, channels, and a package model
that supports separation and testing of configuration, decoding, policy,
detection, and output responsibilities. `libbpfgo` gives the runtime access to
libbpf's object-loading, CO-RE, map, program, link, and perf-buffer facilities
without reimplementing those mechanisms. Neither the language choice nor the
binding is, by itself, a contribution or novelty claim.

## Architectural summary

The user-space runtime has four principal responsibilities:

1. it converts CLI input into a validated runtime configuration and resolves
   the eBPF object and BTF source;
2. it loads the eBPF object, disables programs that are outside the selected
   scope, attaches the required programs, and owns their lifecycle;
3. it consumes binary perf-buffer records and converts them into normalised,
   registry-backed events;
4. it routes accepted events independently to raw-event presentation and to
   the detector engine, then presents any generated alerts while keeping
   operational diagnostics on a separate logging path.

The principal design boundary is therefore:

```text
kernel space
  observation -> bounded extraction -> optional early filtering -> serialization

user space
  configuration -> loading/attachment -> decoding -> selection/policy checks
  -> raw output and detector dispatch -> alert output
  + separate operational logging
```

The architecture is selective at two different stages. Probe selection controls
which registered eBPF programs are loaded and attached. Later event selection,
command-name filtering, and policy evaluation decide whether a received event
continues through the user-space processing path. These stages are related but
are not interchangeable.

## 1. End-to-end user-space lifecycle

### 1.1 Entry point and configuration

The executable entry point is `cmd/project/main.go`. It delegates CLI handling
to the Cobra root command and prints any terminal error to standard error before
exiting with a non-zero status.

The root command in `cmd/project/cmd/root.go` follows this order:

1. normalise and validate the configuration;
2. serve `--list-events` immediately when requested, without loading an eBPF
   object;
3. resolve and read the eBPF object and the BTF path;
4. construct the application runner;
5. create a context cancelled by `SIGINT` or `SIGTERM`;
6. execute the runtime until cancellation or a terminal runtime error.

Configuration is centralised in `pkg/config/config.go`. Normalisation supplies
defaults and canonicalises user input, while validation rejects inconsistent
or unsupported combinations before kernel resources are acquired. This
boundary is useful architecturally: downstream packages receive a coherent
configuration rather than repeatedly interpreting raw CLI values.

### 1.2 eBPF object and BTF resolution

`pkg/cmd/initialize/bpfobject.go` resolves the object with the following
precedence:

1. the `PROJECT_BPF_OBJECT` environment variable;
2. the configured CLI path;
3. the embedded `dist/project.bpf.o` object.

The BTF source is similarly resolved from the `PROJECT_BTF_FILE` environment
variable, the configured value, or `/sys/kernel/btf/vmlinux`. The object bytes
are read at this stage. The BTF path is resolved to an absolute path, but its
actual usability is established later by libbpf during object creation and
loading.

This separation lets startup fail before attachment when the primary object
configuration is invalid, while leaving libbpf responsible for kernel-specific
loading and relocation checks.

### 1.3 Policy and detector preparation

`pkg/cmd/project.go` constructs runtime extensions before creating the eBPF
runtime:

- configured policy files are loaded and their event references are validated;
- the policy manager derives an effective event selection where possible;
- detector YAML files are loaded in deterministic path order;
- detector event references are checked against the event registry;
- the detector engine is created from the validated definitions.

The policy-derived selection can narrow the requested event set when all
relevant allow rules declare explicit event includes. An explicit CLI event
selection is intersected with that derived scope. Exclusions remain effective
when probes are selected.

Detector loading does not automatically enlarge the selected event scope.
Consequently, a valid detector can be loaded while receiving no matching
events if neither the CLI nor the active policy selects the events it consumes.
This is an important configuration boundary, not a decoder failure.

The structured logger does not yet exist during this preparation stage. Errors
from configuration, object resolution, policy loading, or detector loading
therefore return to the entry point and are reported as ordinary terminal
errors rather than zap records.

### 1.4 Runtime construction

`ProjectRunner.Run` creates the zap logger and reports the effective runtime
configuration. It then constructs the eBPF runtime in
`pkg/ebpf/project.go`, passing:

- the normalised configuration;
- the policy manager, when configured;
- the detector engine, when configured;
- the operational logger.

Runtime construction uses the public probe registry to select the requested
logical events and their support probes. It also creates:

- a raw-event printer on standard output;
- a distinct alert printer on standard output when alert output is enabled;
- an optional network-aware JSON printer for raw network events.

The raw-event and alert printers may use different formats even though the
current CLI sends both to the same standard-output stream.

### 1.5 Object loading and attachment

`Project.Init` owns kernel-facing initialisation:

1. initialise the detector engine;
2. install the libbpf logging callback;
3. create a libbpf module from the resolved object bytes and BTF path;
4. resolve the cgroup v2 attachment target when selected probes require it;
5. disable autoload for registered programs outside the selected probe set;
6. ask libbpf to relocate, verify, and load the object;
7. configure the optional kernel UID filter through the BPF configuration map;
8. initialise and start the `events` perf buffer;
9. attach the selected programs and retain their links.

Disabling autoload is significant: unselected registered programs are not
merely left unattached after loading; they are excluded from the verifier and
load path. Support probes brought in through registry dependencies remain
available even if their names are not public CLI event names.

The current implementation uses perf-event transport. Ring-buffer concepts
remain background or retained infrastructure and must not be described as the
active event path.

### 1.6 Event consumption

`Project.Run` waits on:

- context cancellation;
- the perf-event channel;
- the lost-event notification channel.

Cancellation and orderly closure of the event channel end the run normally.
Each record is handled synchronously by the event-processing path. Lost-event
notifications are emitted as operational warnings rather than reconstructed as
events.

For a valid received record, the current processing order is:

```text
decode
  -> public event-selection check
  -> command-name filter
  -> user-space policy check
  -> raw-event output, unless alerts-only
  -> detector evaluation
  -> alert output for each match
```

This order qualifies the simplified Chapter 3 pipeline. In combined-output
mode, the raw event is printed before detector evaluation. With
`--alerts-only`, only the raw presentation step is suppressed: collection,
decoding, user-space filtering, policy evaluation, and detector dispatch still
occur. The option is therefore a presentation control and is unrelated to the
optional kernel UID filter.

### 1.7 Shutdown

Shutdown starts when the signal-aware context is cancelled or the runtime
returns. `Project.Close`:

1. destroys retained links in reverse order;
2. unmounts a cgroup v2 hierarchy if the runtime mounted one;
3. closes the perf buffer;
4. closes the libbpf module;
5. aggregates cleanup errors internally.

The runner also synchronises the zap logger. Two qualifications are important:

- `ProjectRunner.Run` currently defers `Project.Close` without propagating its
  returned cleanup error;
- logger synchronisation errors are also ignored by the deferred call.

No explicit detector-engine flush is performed at shutdown. Incomplete
collective matches simply cease with the process. These are implementation
details and potential hardening tasks, not reasons to describe shutdown as
unsafe at the architectural level.

## 2. Decoding, normalisation, and event-registry responsibilities

### 2.1 Binary decoding

`pkg/bufferdecoder/eventsreader.go` is the top-level binary decoder. The record
contract begins with:

```text
[fixed event context][argument count][indexed argument records]
```

The current shared event-context length is 136 bytes. The decoder uses explicit
little-endian reads and rejects a record that cannot contain the complete
context or declared argument metadata.

After reading the event ID, the decoder obtains the corresponding specification
from `pkg/events/spec.go`. The specification declares the event name and the
ordered argument schema. Each binary argument carries its own index, allowing
the decoder to associate payloads with the schema and restore declared argument
order rather than relying solely on arrival order.

The decoder supports fixed-width scalar values and specialised variable-length
records, including strings, byte sequences, string arrays, argument arrays,
socket addresses, and credential structures. Size checks, buffer bounds, and
UTF-8 validation prevent malformed records from becoming partially trusted
events.

The long-record fix in commit `163ad72` is relevant implementation evidence:
argument offsets into the 32,000-byte argument buffer are now checked with
explicit bounds logic instead of a bit mask that was only valid for
power-of-two capacities. A regression test covers this failure mode. The exact
capacity and correction belong in Chapter 4.

### 2.2 Event registry

The userspace event registry in `pkg/events/spec.go` is the authoritative
decoder-side catalogue for:

- event IDs and names;
- argument names, types, and order;
- name-to-ID and ID-to-name lookup;
- validation of event references used by configuration, policies, and
  detectors.

It does not attach probes. Probe attachment metadata is owned separately by
`pkg/ebpf/probes/probes.go`. The two registries form different views of the same
logical event contract:

- the event registry describes how an event is identified and decoded;
- the probe registry describes which eBPF program or programs produce it and
  how they are loaded and attached.

Public events must remain coherent across the C event IDs, Go event
specifications, and probe registry. An unknown event ID is rejected explicitly;
the runtime does not fabricate an opaque event and continue as if its schema
were known.

### 2.3 Normalisation

Binary decoding yields a typed `bufferdecoder.Event`. Presentation-oriented
normalisation occurs later in `pkg/output/event.go`. It converts fixed C arrays
to strings, groups context into process, host, and kernel-facing structures,
and adds selected symbolic representations suitable for output.

This distinction should remain clear in the thesis:

- decoding establishes structural validity and typed values;
- registry lookup supplies event semantics;
- normalisation converts the internal event into a stable presentation model.

Detector input is adapted from the decoded event before presentation output.
Detectors are therefore not expected to parse terminal text or JSON emitted by
the output layer.

## 3. Raw output, alert output, and zap logging

The implementation deliberately maintains three logical channels.

### 3.1 Raw event output

Raw event output presents accepted, decoded events. It supports table and JSON
formats through the `output.Printer` abstraction. It represents telemetry and
can be disabled with the alerts-only presentation mode without affecting
collection or detector evaluation.

### 3.2 Detector alert output

Alert output is generated only after detector evaluation. It uses a separate
printer instance and can use a format independent of raw event output. Point
alerts preserve their source event; collective alerts preserve a bounded
sequence of source evidence. Alert rendering includes detector and threat
metadata, including MITRE ATT&CK classification when configured.

Alerts are semantic detection results, not operational log messages.

### 3.3 Operational logging

Zap is used for runtime diagnostics such as:

- startup and effective configuration;
- libbpf messages;
- attachment information;
- lost perf-buffer records;
- decoding failures;
- output failures;
- detector-processing failures;
- cleanup diagnostics.

Operational logs are written to standard error and can use console or JSON
encoding. Zap does not format eBPF events and does not format detector alerts.
Those remain the responsibility of the output package on standard output.

The earliest startup failures are a small exception to the structured logging
model: because logger creation occurs after configuration and extension
preparation, those errors are returned to `main` and printed directly.

## 4. Error propagation and malformed-event handling

The runtime distinguishes fatal setup errors from event-local processing
errors.

| Failure class | Current behaviour | Architectural consequence |
|---|---|---|
| Invalid CLI or configuration | Returned from Cobra execution; process exits non-zero | Kernel resources are not acquired |
| Missing/unreadable eBPF object | Returned during object resolution; process exits | Failure occurs before module creation |
| Invalid policy or detector definition | Returned during runner construction; process exits | The runtime never starts with a partially accepted rule set |
| Module creation, relocation, verifier, load, map, perf-buffer, or attach failure | Returned from runtime initialisation; process exits and deferred cleanup runs | A partially initialised runtime is not allowed to enter the event loop |
| Unknown event ID or malformed binary record | Logged with zap and dropped | One invalid record does not terminate monitoring |
| Event excluded by selection, command filter, or policy | Silently dropped in normal operation; debug diagnostics may describe the reason | Filtering is part of normal dispatch, not an error |
| Perf-buffer lost-event notification | Logged as a warning | Data loss is observable but cannot be reconstructed |
| Raw-event or alert rendering failure | Logged and processing continues | Presentation failure does not stop collection |
| Detector evaluation error | Logged and processing continues | A detector-local failure does not terminate the runtime |
| Context cancellation | Normal run completion | Supports signal-driven shutdown |
| Cleanup error | Aggregated by `Project.Close`, but currently not propagated by the runner | Cleanup observability should be improved in implementation work |

Malformed-event handling is intentionally fail-closed at the record boundary:
the runtime does not pass a partially decoded event to policies, detectors, or
printers. This supports a strong Chapter 3 claim about isolation of malformed
records, but not a claim that all event loss is recovered or persisted.

## 5. Event selection, probe attachment, and later dispatch

### 5.1 Selection before loading

`probes.Select` resolves public event names. When no inclusion list is supplied,
the default is the supported public event set. Explicit exclusions take
priority. The result contains:

- the physical probe specifications needed for the selected logical events;
- a set of selected public event names used later by dispatch.

A probe can imply support probes. Such dependencies are attached because the
selected public event requires them, but they are not necessarily user-facing
events and should not appear in `--list-events`.

`ConfigureAutoload` disables registered eBPF programs outside this resolved
probe set before object loading. This reduces verifier and load work and avoids
attaching irrelevant programs.

### 5.2 Dispatch after transport

Attachment selection does not eliminate the need for a later event check.
Several physical probes may contribute to a logical event, support probes may
emit intermediate records, and selected cgroup programs can produce
protocol-specific records whose names are not standalone public probe names.

The event gate therefore:

- accepts explicitly selected public events;
- rejects known public events that were not selected;
- permits names that are not standalone public probe-registry entries when
  they arise from selected protocol-aware producers.

After that gate, command filtering and policy checks can narrow the stream
further. Only accepted events reach raw presentation and detector evaluation.

### 5.3 Policy and detector influence

Policy selection can affect attachment indirectly by narrowing the effective
event set before `Project.New`. User-space policy rules also evaluate each
decoded event later in the dispatch path.

Detector definitions do not currently request or attach missing probes. They
are consumers of the event scope established by CLI and policy configuration.
This should be stated clearly in Chapter 3 to avoid implying automatic detector
dependency resolution.

### 5.4 Current worktree warning

The uncommitted version of `pkg/ebpf/probes/probes.go` removes the `internal`
classification from some networking support probes. The registry invariant
test then fails because at least `sock_alloc_file` becomes public without a
corresponding decoder specification. This is not a stable architectural
change. Until the registry and decoder contract are updated together, those
support programs must remain internal dependencies rather than advertised
events.

## 6. Proposed end-to-end sequence diagram

The Chapter 3 diagram should use the following participants:

- Operator;
- CLI and Configuration;
- Policy/Detector Preparation;
- Go Runtime;
- Probe Registry;
- libbpf/libbpfgo;
- Linux Kernel/eBPF Programs;
- Perf Buffer;
- Decoder and Event Registry;
- Policy Manager;
- Raw Event Output;
- Detector Engine;
- Alert Output;
- Operational Logger.

The sequence can be described as follows:

1. The operator starts the executable with event, policy, detector, output, and
   logging options.
2. CLI and Configuration normalise and validate the request.
3. The object resolver selects the eBPF object and BTF source.
4. Policy/Detector Preparation loads definitions, validates event references,
   and optionally narrows the effective event scope.
5. The Go Runtime asks the Probe Registry to resolve selected logical events
   into physical programs and support dependencies.
6. The Go Runtime creates the libbpf module and asks the Probe Registry to
   disable autoload for unselected registered programs.
7. libbpf relocates, verifies, and loads the remaining programs.
8. The Go Runtime configures the optional kernel UID filter, starts the perf
   buffer, and attaches selected programs.
9. A kernel operation reaches an attached hook.
10. The eBPF program extracts bounded context and arguments, optionally applies
    the configured UID filter, and submits a binary record.
11. The Perf Buffer delivers the record to the Go Runtime; loss notifications
    take a separate path to the Operational Logger.
12. The Decoder reads the common context, resolves the event ID through the
    Event Registry, and decodes typed indexed arguments.
13. If decoding fails, the Go Runtime logs and drops the record.
14. The Go Runtime checks event selection and command filtering.
15. The Policy Manager decides whether the event belongs to the active
    user-space scope.
16. If raw presentation is enabled, the accepted event is sent to Raw Event
    Output.
17. The same accepted event is sent to the Detector Engine.
18. If no detector matches, processing ends for that event.
19. If one or more detectors match, each alert and its supporting evidence are
    sent to Alert Output.
20. Output or detector errors are sent to the Operational Logger without
    terminating the event loop.
21. On cancellation, the Go Runtime destroys links, closes transport and
    module resources, and synchronises logging.

The diagram should show two explicit alternatives:

- `alerts-only`: omit step 16, but retain steps 17-19;
- malformed record: take step 13 and do not enter policy or detector
  processing.

It should also show operational logging as a side channel, not as a stage that
serialises raw events or alerts.

## 7. Claim-to-code matrix for Sections 3.2 and 3.5

### Section 3.2: Overall System Architecture

| Proposed design claim | Primary implementation evidence | Safe Chapter 3 formulation |
|---|---|---|
| Kernel and user space have separate responsibilities | `pkg/ebpf/c/project.bpf.c`; `pkg/ebpf/project.go`; `pkg/bufferdecoder`; `pkg/policy`; `pkg/detectors` | Kernel programs collect and serialise bounded observations; the Go runtime owns lifecycle, decoding, policy evaluation, detection, and presentation |
| The runtime is selective rather than attaching every probe | `pkg/ebpf/probes.Select`; `pkg/ebpf/probes.ConfigureAutoload`; `pkg/ebpf/project.go:New` and `Init` | Requested logical events are resolved before loading, and unrelated registered programs are excluded from autoload and attachment |
| Binary events cross a perf-buffer boundary | `pkg/ebpf/project.go:Init`; `pkg/bufferdecoder/eventsreader.go:DecodeEvent` | The current transport uses perf buffers and treats loss and malformed records as explicit failure boundaries |
| Policy scope can influence selected events | `pkg/policy/manager.go:EffectiveEventSelection`; `pkg/cmd/project.go:newRuntimeExtensions` | Explicit policy event scopes can narrow the effective collection domain before attachment |
| Detectors operate over normalised event objects, not output text | `pkg/detectors/engine.go:ProcessEvent`; adapters in `pkg/ebpf/project.go`; `pkg/output` | Accepted events are passed as typed internal observations to the detector engine independently of their presentation format |
| Operational logging is separate from event and alert data | `pkg/logging/logger.go`; `pkg/output.Printer`; `pkg/ebpf/project.go:handleRawEvent` | Runtime diagnostics use a separate structured logging path, while telemetry and alerts use dedicated output paths |
| The design is observation and detection, not enforcement | No deny or response path in `pkg/ebpf`, `pkg/policy`, or `pkg/detectors` | The prototype observes activity and emits detections; it does not block kernel operations |

### Section 3.5: User-Space Runtime Design

| Proposed design claim | Primary implementation evidence | Safe Chapter 3 formulation |
|---|---|---|
| Startup validates configuration before kernel setup | `cmd/project/cmd/root.go:RunE`; `pkg/config`; `pkg/cmd/initialize.BPFObject` | Configuration and rule definitions are validated before object loading and attachment |
| The runtime supports signal-aware lifecycle control | `cmd/project/cmd/root.go`; `pkg/cmd/project.go:ProjectRunner.Run`; `pkg/ebpf/project.go:Run` | A cancellation context coordinates normal event-loop termination and resource cleanup |
| Go and libbpfgo divide application and kernel-facing work | `pkg/cmd`; `pkg/ebpf/project.go`; local `libbpfgo` dependency in `go.mod` | Go structures application control and typed processing, while libbpfgo exposes libbpf's object, map, program, link, and perf-buffer operations |
| Event IDs and schemas are registry-backed | `pkg/events/spec.go:GetSpec`, `Exists`, `IDByName`, `EventName`; `pkg/bufferdecoder/eventsreader.go` | The decoder resolves each event through an explicit registry of IDs, names, and typed argument schemas |
| Unknown or malformed records are isolated | `pkg/bufferdecoder`; `pkg/ebpf/project.go:handleRawEvent`; decoder tests | Structurally invalid records are logged and dropped before policy, detector, or output processing |
| Event selection is enforced both before attachment and after decoding | `pkg/ebpf/probes.Select`; `ConfigureAutoload`; `pkg/ebpf/project.go:eventEnabled` and `handleRawEvent` | Early program selection reduces the loaded scope, while a later gate protects logical dispatch boundaries |
| Raw output and alert output are independently configurable | `pkg/config`; `pkg/output.Printer`; `pkg/ebpf/project.go:New` | Telemetry and detections use separate presentation paths and may use different formats |
| Alerts-only is a presentation mode | `pkg/ebpf/project.go:handleRawEvent` | Suppressing raw display does not disable collection, decoding, policies, or detector evaluation |
| Zap is only operational logging | `pkg/logging/logger.go`; logger calls in `pkg/ebpf/project.go`; output printers | Structured diagnostics are emitted separately and do not serialise events or alerts |
| Event-local failures do not necessarily terminate monitoring | `pkg/ebpf/project.go:handleRawEvent` | Decode, detector, and rendering failures are isolated and reported while the event loop continues |
| Setup failures are fatal | `pkg/ebpf/project.go:Init`; `pkg/cmd/project.go:Run` | Object, verifier, transport, and attachment failures prevent the runtime from entering normal operation |

## 8. Details reserved for Chapter 4

The following facts are valuable evidence but are too implementation-specific
for the Chapter 3 design narrative:

- package and function walkthroughs, including the exact Cobra `RunE`,
  `ProjectRunner`, and `Project` call chain;
- environment-variable and CLI precedence for object and BTF resolution;
- concrete channel capacities, perf-buffer page count, and callback signatures;
- the exact 136-byte event-context layout and individual field offsets;
- the 32,000-byte argument-buffer capacity and the bit-mask-to-bounds-check
  correction from commit `163ad72`;
- the exact argument-record index and length encodings;
- detailed decoder branches for credentials, arrays, socket addresses, and
  byte buffers;
- concrete event IDs, registry entries, probe section names, and attachment
  symbols;
- the `impliedBy` dependency representation and public/internal registry flags;
- cgroup mount discovery and cleanup mechanics;
- BPF map names and keys used for kernel UID filtering;
- output field names and the exact table or JSON serialisation format;
- zap encoder construction, libbpf message-level mapping, and concrete logging
  flags;
- the current ignored return values from deferred cleanup and logger
  synchronisation;
- the absence of an explicit detector flush during shutdown;
- the current temporary configuration-map test write performed during
  initialisation;
- regression-test implementation and command-level reproduction procedures.

Chapter 3 should instead explain the responsibility boundaries and their
rationale: validation before acquisition, selective loading, stable typed
decoding, separate telemetry/detection/diagnostic paths, bounded local failure,
and ordered cleanup.

## Documentation and implementation discrepancies

The audit found the following points that should be corrected or qualified in
future documentation and thesis drafting:

1. `documentation/report.md` states that the Go runtime calls
   `rlimit.RemoveMemlock()`. No such call exists in the audited current source.
   This must not be claimed unless it is reintroduced and verified.
2. Some simplified diagrams place detector evaluation before all output. The
   implementation prints the raw event first in combined-output mode, then
   evaluates detectors and prints alerts.
3. Structured logging does not cover every startup error. Logger creation
   follows configuration and extension preparation, so earlier failures are
   returned and printed plainly by `main`.
4. Cleanup is attempted and internally aggregates errors, but the runner
   currently ignores the returned cleanup error. “Orderly cleanup” is accurate;
   “all cleanup failures are propagated” is not.
5. Detector definitions are validated against the event registry but do not
   automatically request missing probes. Policy and CLI event scope remain the
   source of attachment selection.
6. The uncommitted networking probe-registry edits break the tested invariant
   that every public probe has a decoder specification. They should not be used
   as evidence of the stable architecture.

## Validation performed

Focused Go tests for configuration, command wiring, decoding, event registry,
output, logging, detector engine, and detector YAML handling passed when run
with the repository's libbpf build environment.

The dedicated probe-registry test failed against the dirty worktree because the
uncommitted networking edits make support probes public without decoder
specifications. The policy package also observed the untracked
`rules/policies/test_net.yaml`, changing the preset count expected by an
existing test. These failures are caused by local uncommitted state and should
be resolved before using the worktree as a reproducible thesis baseline.

