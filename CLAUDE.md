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
Key dependency: `github.com/aquasecurity/libbpfgo`, with CGO and the bundled
`libbpf` submodule.

### Build & Run

```bash
# Compile the eBPF C program
make bpf

# Build the eBPF object and Go userspace binary
make build

# Run with the embedded eBPF object
sudo ./dist/project --output table
```

### Package Architecture

```
cmd/project/main.go           → CLI entrypoint
cmd/project/cmd/root.go       → Cobra command, config validation, signal handling
pkg/cmd/initialize/           → loads .bpf.o bytes into runtime config
pkg/ebpf/                     → loads module, selects probes, attaches hooks, reads event buffers
pkg/ebpf/probes/              → event/program/hook registry
pkg/bufferdecoder/            → deserializes raw event bytes into typed Event structs
pkg/events/                   → event IDs and argument schemas
pkg/output/                   → normalized table and JSON output
pkg/ebpf/c/                   → eBPF C program and headers (types.h, maps.h, common/)
```

Data flow:
`perf buffer → project.Run() → bufferdecoder.DecodeEvent() → output printer`.
Ring buffer support remains available, but active hooks currently submit
through the perf buffer.

### Critical Constraint: Struct Layout Parity

`pkg/bufferdecoder/protocol.go` declares Go structs that must match the C
structs in `pkg/ebpf/c/types.h`. Event IDs must match
`pkg/events/ids.go`, while argument indexes and types must match
`pkg/events/spec.go`.

`EventContext` is 136 bytes: `[u64 Ts][TaskContext 104B][EventID u32][Syscall s32][StackID u32][ProcessorID u16][PoliciesVersion u16][MatchedPolicies u64]`

### Hook Attachment Strategy

The runtime uses `pkg/ebpf/probes/probes.go` as its probe registry. Each entry
maps a decoded event name to an eBPF program, kernel hook, and attach method.
The CLI can select events with `--events` and `--drop-events`.

| Probe kind | Attach method |
|---|---|
| raw tracepoint | `AttachRawTracepoint` |
| tracepoint | `AttachTracepoint` |
| kprobe | `AttachKprobe` |
| kretprobe | `AttachKretprobe` |

### eBPF Maps

Defined in `pkg/ebpf/c/maps.h`:

| Map | Type | Purpose |
|---|---|---|
| `events` | `PERF_EVENT_ARRAY` | Primary kernel→userspace event channel |
| `events_ringbuf` | `RINGBUF` | Retained alternative event channel |
| `args_bufs` | `PERCPU_ARRAY` | Per-CPU scratch buffer for event arguments |
| `args_map` | `HASH` | Correlates syscall entry arguments with syscall exit |

### Adding a New Event Type

1. Add the event ID constant in `pkg/ebpf/c/types.h`.
2. Add the matching Go ID in `pkg/events/ids.go`.
3. Add the argument schema in `pkg/events/spec.go`.
4. Add the eBPF C hook in `pkg/ebpf/c/project.bpf.c`.
5. Add the event/program/hook mapping in `pkg/ebpf/probes/probes.go`.
6. Add probe-selection, decoder and output tests as needed.

### Current Hook Coverage

The tool now covers several macro-areas:

- process lifecycle and exec: `sched_process_*`, `task_rename`, `execve`,
  `execveat`, `process_execute_failed`, `fork`, `vfork`, `clone`;
- credentials and privilege changes: `setuid`, `setgid`, `setreuid`,
  `setregid`, `setresuid`, `setresgid`, `setfsuid`, `setfsgid`,
  `security_task_fix_setuid`, `commit_creds`;
- files, memory and namespaces: `open`, `chmod`, `chown`, `mmap`,
  `mprotect`, `pkey_mprotect`, `memfd_create`, `process_vm_readv`,
  `process_vm_writev`, `setns`, `unshare`, `switch_task_ns`;
- security/VFS hooks: `security_file_open`, `security_file_permission`,
  `security_file_ioctl`, `security_inode_*`, `security_sb_*`;
- kernel hardening: `security_bpf*`, `security_kernel_*_read_file`,
  `module_load`, `module_free`, `do_init_module`, `proc_create`,
  `register_kprobe`, `kallsyms_lookup_name`;
- cgroup, signal and networking hooks.

The final detection engine is still in progress.
