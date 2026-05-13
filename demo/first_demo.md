# Project demo notes

## Goal

This demo shows the current end-to-end pipeline:

- selected kernel hooks are attached from Go;
- events are sent through ring buffer or perf buffer;
- userspace decodes raw bytes into structured events;
- filters reduce noise;
- output is printed as `table` or `json`.

This is not yet a detection engine. The goal of the demo is to show that the
event collection pipeline works and that the architecture is ready for
enrichment and detection logic.

## Current Pipeline

```mermaid
flowchart TD
    A[CLI config] --> B[Resolve embedded or external eBPF object]
    B --> C[Load eBPF module with libbpfgo]
    C --> D[Select probes with --events and --drop-events]
    D --> E[Attach raw tracepoints and kprobes]
    E --> F1[Ring buffer: events_ringbuf]
    E --> F2[Perf buffer: events]
    F1 --> G[handleRawEvent]
    F2 --> G
    G --> H[Decode event context and args]
    H --> I[Filter event name]
    I --> J[Filter comm with --comms]
    J --> K[Output printer]
    K --> L[table or json]
```

## What Works Today

- eBPF object build through `make bpf`.
- Go userspace runtime based on `libbpfgo`.
- Probe registry in `pkg/ebpf/probes`.
- Event selection with `--events` and `--drop-events`.
- Command-name filtering with `--comms`.
- Ring buffer reader for process/security events.
- Perf buffer reader for Tracee-like/networking events.
- Output layer with `json` and `table`.

## Event Groups

```mermaid
mindmap
  root((Supported events))
    Process lifecycle
      sched_process_fork
      sched_process_exec
      sched_process_exit
      task_rename
    Security
      cap_capable
      security_task_setrlimit
      security_settime64
      security_task_prctl
    Networking
      security_socket_create
      security_socket_listen
      security_socket_connect
      security_socket_accept
      security_socket_bind
      security_socket_setsockopt
      security_socket_recvmsg
      security_socket_sendmsg
```

## Demo Flow

### 1. Build the Tool

```bash
cd demo_project
make build
```

Expected result:

```text
dist/project.bpf.o
dist/project
```

### 2. Show Process Execution Events

Terminal 1:

```bash
sudo ./dist/project \
  --events task_rename,sched_process_exec,sched_process_exit \
  --comms ls,whoami \
  --output table
```

Terminal 2:

```bash
ls
whoami
```

Expected output shape:

```text
event=task_rename pid=... tid=... uid=1000 comm=bash args=old_name=bash,new_name=ls
event=sched_process_exec pid=... tid=... uid=1000 comm=ls args=filename=/usr/bin/ls
event=sched_process_exit pid=... tid=... uid=1000 comm=ls args=exit_code=0,group_dead=1
```

What to explain:

- `task_rename` shows the task `comm` transition.
- `sched_process_exec` shows the executed binary path.
- `sched_process_exit` shows process termination and exit code.
- `--comms` keeps the output focused on the commands used in the demo.

### 3. Show Networking Event Path

Terminal 1:

```bash
make filtered
```

Terminal 2:

```bash
python3 -m http.server 18080 --bind 127.0.0.1
```

Terminal 3:

```bash
curl http://127.0.0.1:18080
```

What to explain:

- `security_socket_connect` is attached as a kprobe.
- Networking hooks currently follow the perf-buffer path.
- The runtime now reads both ring buffer and perf buffer.

### 4. Optional: Show a Sensitive Security Hook

Terminal 1:

```bash
sudo ./dist/project --events security_settime64 --output table
```

Terminal 2:

```bash
sudo date -s "$(date '+%Y-%m-%d %H:%M:%S')"
```

What to explain:

- `security_settime64` is triggered when a process tries to modify system time.
- It requires privileges such as `CAP_SYS_TIME`.
- Use this only on a test VM.

## Architecture Talking Points

```mermaid
sequenceDiagram
    participant User as User CLI
    participant Go as Go runtime
    participant BPF as eBPF program
    participant Kernel as Kernel hook
    participant Out as Output

    User->>Go: project --events ... --comms ...
    Go->>BPF: load object and attach selected probes
    Kernel->>BPF: hook is triggered
    BPF->>Go: submit raw event through ring/perf buffer
    Go->>Go: decode context and args
    Go->>Go: apply event and comm filters
    Go->>Out: print table/json event
```

Key points:

- The tool does not attach every probe blindly when `--events` is used.
- The decoder is intentionally separated from output formatting.
- The output layer can evolve without changing the eBPF runtime.
- The current design is close to Tracee conceptually, but smaller and easier to
  reason about for the thesis.

## Known Limits

- Some arguments are still raw numeric values.
- `security_task_prctl` needs symbolic mapping for options like `PR_SET_VMA`.
- Socket addresses and socket constants need better human-readable formatting.
- `--comms` is a userspace filter, so it improves output readability but does
  not reduce kernel-side work.
- The final transport strategy still needs consolidation: keep both ring/perf
  buffers, or converge to perf buffer in a more Tracee-like design.
- Detection logic and MITRE ATT&CK mapping are future work.

## Short Closing

The current version demonstrates the hard part of the project: a working
kernel-to-userspace event pipeline on the target Rocky Linux kernel. The next
step is not loading eBPF anymore, but improving event semantics: better
argument mapping, stronger filters and eventually detection logic.
