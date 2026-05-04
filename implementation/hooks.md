# Hook implementati

## Process lifecycle

### `sched_process_fork`

Tipo attach:

```text
raw_tracepoint/sched_process_fork
```

Scopo:

- intercettare creazione di processi/thread;
- raccogliere parent PID/TID;
- raccogliere child PID/TID;
- raccogliere start time del child.

### `sched_process_exec`

Tipo attach:

```text
raw_tracepoint/sched_process_exec
```

Scopo:

- intercettare exec;
- leggere `linux_binprm->filename`;
- inviare il path allo userspace.

### `sched_process_exit`

Tipo attach:

```text
raw_tracepoint/sched_process_exit
```

Scopo:

- intercettare uscita processo/thread;
- raccogliere exit code;
- capire se sta terminando il thread group.

## Security-related hooks

### `task_rename`

Tipo attach:

```text
raw_tracepoint/task_rename
```

Scopo:

- intercettare cambio nome task;
- salvare vecchio nome e nuovo nome.

### `cap_capable`

Tipo attach:

```text
kprobe/cap_capable
```

Scopo:

- intercettare controlli di capability;
- salvare capability richiesta.

### `security_task_setrlimit`

Tipo attach:

```text
kprobe/security_task_setrlimit
```

Scopo:

- intercettare modifiche ai resource limits;
- salvare target PID, resource, soft limit e hard limit.

### `security_settime64`

Tipo attach:

```text
kprobe/security_settime64
```

Scopo:

- intercettare tentativi di modificare il tempo di sistema.

### `security_task_prctl`

Tipo attach:

```text
kprobe/security_task_prctl
```

Scopo:

- intercettare chiamate `prctl` passate dal security hook;
- utile per future detection su manipolazioni di processo.

## Collegamenti

- [Timeline](../timeline.md)
- [Overview implementazione](overview.md)
- [Debugging verifier](../debugging/ebpf-verifier.md)
