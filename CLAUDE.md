# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

---

## Repository Layout

```
project/
├── demo_project/   ← "Vesuvius" — the tool being built (active development)
└── tracee/         ← Aqua Security's Tracee (reference implementation, read-only)
```

All active development happens in `demo_project/`. Tracee is used only as reference for understanding eBPF patterns and data structures.

---

## Vesuvius — demo_project/

An eBPF-based runtime security monitoring tool. Target environment: **Rocky Linux 8.10**, kernel `4.18.0-553.109.1.el8_10.x86_64`.

Go module: `github.com/V3suv1us/demo_project`  
Key dependency: `github.com/cilium/ebpf` (pure Go — **no CGO**, unlike Tracee's libbpfgo)

### Build & Run

```bash
# Compile the eBPF C program (run from pkg/ebpf/c/ or adjust include paths)
clang -O2 -target bpf -c vesuvius.bpf.c -o vesuvius.bpf.o

# Build the Go userspace binary
go build ./cmd/vesuvius/

# Run (requires root or CAP_BPF + CAP_PERFMON)
sudo ./vesuvius [path/to/vesuvius.bpf.o]
# Defaults to vesuvius.bpf.o in the current directory if no path given
```

### Package Architecture

```
cmd/vesuvius/main.go   → wires loader → attach → ReadEvents loop
pkg/loader/            → loads .bpf.o, attaches hooks, drives ring buffer read loop
pkg/decoder/           → deserializes raw ring buffer bytes into typed Event structs
pkg/output/            → prints Event as a single JSON line (JSONL) to stdout
pkg/ebpf/c/            → eBPF C headers (types.h, maps.h, vmlinux*.h, common/)
```

Data flow: `ring buffer → loader.ReadEvents() → decoder.DecodeEvent() → output.JSONPrinter`

### Critical Constraint: Struct Layout Parity

`decoder/types.go` declares Go structs (`TaskContext`, `EventContext`) that must be **byte-for-byte identical** to the C structs in `pkg/ebpf/c/types.h`. The decoder uses `encoding/binary.Read` with `binary.LittleEndian` — any size or order mismatch silently corrupts every decoded event. If `types.h` changes, `decoder/types.go` must be updated to match.

`EventContext` is 128 bytes: `[u64 Ts][TaskContext 104B][EventID u32][Syscall s32][StackID u32][ProcessorID u16][Pad u16]`

### Hook Attachment Strategy

The loader (`pkg/loader/loader.go`) reads the ELF section name of each eBPF program and routes it to the correct attach method — no hardcoded program list:

| Section prefix | Attach method |
|---|---|
| `raw_tracepoint/<name>` | `link.AttachRawTracepoint` |
| `tracepoint/<group>/<name>` | `link.Tracepoint` |
| `lsm/<name>` | `link.AttachLSM` |

### eBPF Maps

Defined in `pkg/ebpf/c/maps.h`:

| Map | Type | Purpose |
|---|---|---|
| `events` | `RINGBUF` (256 KB) | Primary kernel→userspace event channel |
| `scratch_map` | `PERCPU_ARRAY` (2 slots) | Per-CPU scratch to avoid large eBPF stack usage |
| `task_info_map` | `LRU_HASH` (10 240 entries) | Per-TID persistent task metadata across hook invocations |
| `containers_map` | `HASH` | Cgroup→container state, populated by userspace |

### Adding a New Event Type

1. Add the event ID constant in `pkg/ebpf/c/types.h` (`event_id_e`) and the matching `const` in `decoder/types.go`
2. Add the `case` in `decoder.EventName()`
3. Add the eBPF C hook with `SEC("raw_tracepoint/<name>")` or `SEC("lsm/<name>")`
4. No changes needed to loader, decoder arg parsing, or printer

### Work Division

- **Simone**: process hooks (`sched_process_fork`, `sched_process_exec`, `sched_process_exit`)
- **Giuseppe**: network hooks (`security_socket_connect`, `security_socket_accept`, `security_socket_bind`)

Both the `vesuvius.bpf.c` source file and the final detection engine are still in progress.
