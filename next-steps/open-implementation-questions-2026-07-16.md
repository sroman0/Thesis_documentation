# Dubbi aperti per la prossima call - 2026-07-16

Questo file raccoglie i punti da chiarire prima di continuare con nuove
ottimizzazioni o nuovi hook.

## 1. Quale benchmark conta davvero?

Finora usiamo come riferimento:

```text
CPU media userspace < 5% di un core
```

Da chiarire:

- Il 5% deve valere per il caso realistico o anche per il caso peggiore?
- Usiamo `avg_cpu`, `p95_cpu` o entrambi?
- Una run da 120 secondi basta come benchmark ufficiale?

Proposta: usare il 5% sul profilo realistico/policy-driven, mentre gli stress
test vanno documentati separatamente.

## 2. Quale profilo rappresenta la demo?

Abbiamo questi profili:

```bash
make benchmark-profile PROFILE=raw
make benchmark-profile PROFILE=point
make benchmark-profile PROFILE=collective
KERNEL_FILTER_UID=1000 make benchmark-profile PROFILE=kernel-filter-uid
```

Da chiarire:

- Quale profilo mostriamo in azienda?
- Quale profilo e' piu' vicino al futuro uso in Kubernetes?
- Il profilo `collective` deve includere anche detector puntuali come
  `root-exec`, o solo detector stateful?

Proposta: creare un profilo `production-like` separato da demo e stress test.

## 3. Il detector `root-exec` e' troppo rumoroso?

Durante i test produce alert su processi legittimi come:

```text
getconf
ip
systemd-cgroups-agent
```

Da chiarire:

- Lo teniamo come detector generico?
- Escludiamo helper noti?
- Lo trasformiamo in qualcosa di piu' specifico, ad esempio root shell,
  interpreti, path temporanei o exec dopo privilege change?

Proposta: non usare `root-exec` generico nei benchmark collective.

## 4. Policy e detector: chi decide cosa?

Da chiarire:

- La policy deve abilitare solo eventi o anche detector specifici?
- Se abilito un detector, gli eventi necessari devono essere abilitati
  automaticamente?
- In una run detector-focused, l'output deve essere `alerts-only` di default?

Proposta: mantenere eventi e alert separati, ma rendere esplicito nella policy
se vogliamo raw events, alert o entrambi.

## 5. Kernel-side filtering: quanto spingere?

Abbiamo aggiunto il filtro UID nel kernel:

```bash
--kernel-filter-uid-enabled
--kernel-filter-uid <uid>
```

Da chiarire:

- E' una feature utente o solo uno strumento di benchmark/debug?
- Serve supportare piu' UID?
- Nel contesto Kubernetes conviene filtrare per UID, namespace, cgroup o
  container?
- Ha senso aggiungere anche `comm` kernel-side?

Proposta: tenere il filtro kernel-side minimale per ora; valutare namespace o
cgroup prima di `comm`.

## 6. Continuiamo con hook o stabilizziamo?

Il tool ha gia' molti hook process/security. Aggiungerne altri aumenta copertura
ma anche rumore e costo.

Da chiarire:

- Prima completiamo benchmark/policy/detector o aggiungiamo altri hook?
- Quali hook sono davvero necessari per le policy che vogliamo mostrare?
- Dobbiamo classificare gli hook in core, demo e stress?

Proposta: prima stabilizzare benchmark e detector, poi aggiungere hook mirati.

## 7. Cosa significa "pronto" per la prossima demo?

Da chiarire:

- Basta mostrare alert corretti?
- Serve anche stare sotto il 5%?
- La demo deve privilegiare chiarezza o copertura ampia?

Proposta: demo breve, pochi detector selettivi, output chiaro e benchmark
completo su un profilo realistico.
