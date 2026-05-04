Rivedere gli hook implementati in project.bpf.c, in particolare:
- task_rename
- cap_capable
- security_task_setrlimit
- security_settime64
- security_task_prctl

Crea un branch solo per quello. I file che sono stati modificati sono:
- project.bpf.c
- types.h(nuovi event ID)
- types.go (nuovi mapper)
- loader.go (supporto attach kprobe/kretprobe)