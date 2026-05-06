# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

---

## Repository Layout

```
project/
├── demo_project/   ← Project — the tool being built (active development)
└── tracee/         ← Aqua Security's Tracee (reference implementation, read-only)
```

All active development happens in `demo_project/`. Tracee is used only as reference for understanding eBPF patterns and data structures.

---

## Project — demo_project/

An eBPF-based runtime security monitoring tool. Target environment: **Rocky Linux 8.10**, kernel `4.18.0-553.109.1.el8_10.x86_64`.

Go module: `github.com/V3suv1us/demo_project`  
Key dependency: `github.com/cilium/ebpf` (pure Go — **no CGO**, unlike Tracee's libbpfgo)

### Build & Run

```bash
# Compile the eBPF C program
make bpf

# Build the Go userspace binary
go build -o /tmp/project ./cmd/project

# Run (requires root or CAP_BPF + CAP_PERFMON)
sudo /tmp/project --bpf-object pkg/ebpf/c/project.bpf.o
```

### Package Architecture

```
cmd/project/main.go           → CLI entrypoint
cmd/project/cmd/root.go       → Cobra command, config validation, signal handling
pkg/cmd/initialize/           → loads .bpf.o bytes into runtime config
pkg/ebpf/                     → loads collection, selects probes, attaches hooks, reads ring buffer
pkg/bufferdecoder/            → deserializes raw ring buffer bytes into typed Event structs
pkg/ebpf/c/                   → eBPF C program and headers (types.h, maps.h, common/)
```

Data flow: `ring buffer → project.Run() → bufferdecoder.DecodeEvent() → JSON stdout`

### Critical Constraint: Struct Layout Parity

`pkg/bufferdecoder/protocol.go` declares Go structs that must match the C structs in `pkg/ebpf/c/types.h`. The decoder uses little-endian binary reads, so any size or order mismatch corrupts decoded events. If `types.h` changes, `protocol.go` must be updated to match.

`EventContext` is 128 bytes: `[u64 Ts][TaskContext 104B][EventID u32][Syscall s32][StackID u32][ProcessorID u16][Pad u16]`

### Hook Attachment Strategy

The runtime uses `pkg/ebpf/probes.go` as a Tracee-light probe registry. Each entry maps a decoded event name to an eBPF program, kernel hook, and attach method. The CLI can select events with `--events` and `--drop-events`.

| Probe kind | Attach method |
|---|---|
| raw tracepoint | `link.AttachRawTracepoint` |
| kprobe | `link.Kprobe` |

### eBPF Maps

Defined in `pkg/ebpf/c/maps.h`:

| Map | Type | Purpose |
|---|---|---|
| `events` | `RINGBUF` (256 KB) | Primary kernel→userspace event channel |
| `args_bufs` | `PERCPU_ARRAY` | Per-CPU scratch buffer for event arguments |

### Adding a New Event Type

1. Add the event ID constant in `pkg/ebpf/c/types.h`.
2. Add the matching event ID and event schema in `pkg/bufferdecoder/protocol.go`.
3. Add the eBPF C hook in `pkg/ebpf/c/project.bpf.c`.
4. Add the event/program/hook mapping in `pkg/ebpf/probes.go`.

### Work Division

- **Simone**: process hooks (`sched_process_fork`, `sched_process_exec`, `sched_process_exit`)
- **Giuseppe**: network hooks (`security_socket_connect`, `security_socket_accept`, `security_socket_bind`)

The final detection engine is still in progress.
