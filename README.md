# INFO: task hung in bdev_open() — Linux 7.4-rc2 syzkaller reproducer

Artifacts for a reproducible hang found by syzkaller fuzzing on
**7.4.0-rc2-kernsight-kasan #1** (base `v7.4-rc2`, commit `7475ea8b5`, unmodified
block layer): two tasks (a udev worker and a syzkaller executor) were blocked
for >143 s each, both waiting on the same gendisk `open_mutex` inside
`bdev_open()` (block/bdev.c:990) while opening block device nodes via `openat()`.

The minimized reproducer was verified **2/2 on freshly replayed VMs** by the
state-primed verification flow, and the hang was observed twice during
concurrent fuzzing.

## Reproduce

On a KASAN-enabled 7.4-rc2 kernel (config: `config-7.4.0-rc2-kernsight-kasan`):

```bash
# single minimized program
syz-execprog -repeat=1 repro/repro.prog

# with the state-reconstruction recipe prefix (recommended; see repro/repro.README.txt)
syz-execprog -repeat=1 repro/repro.kernsight.prog
```

The guest VM should run a systemd-based userspace (a udev worker participates
in the hang) with the same kernel command line used by the fuzzing setup
(`panic_on_warn=1 hung_task_panic=0` etc.; the hang itself only triggers
hung-task reporting, the VM keeps running).

## Artifacts

| file | content |
|---|---|
| `logs/log0`, `logs/log1` | raw syzkaller crash logs (executing-program context) |
| `logs/report0`, `logs/report1` | syzkaller crash reports (hung-task dumps) |
| `logs/machineInfo*`, `logs/pre_state*` | guest info / 539-D pre-crash state vectors |
| `logs/crash-excerpt.txt` | tail of the verification-run report |
| `repro/repro.prog` | minimized single-program reproducer |
| `repro/repro.kernsight.prog` | state-reconstruction recipe + crashing program, execution order |
| `repro/repro.priming.prog` | recipe only (run it first, then `repro.prog` on the same VM) |
| `repro/repro.report`, `repro/repro.log` | verification-run report and log |
| `repro/repro.state.json`, `repro/repro.context.json` | measured 539-D state, reproduction context |
| `repro/repro.attempts.json`, `repro/repro.outcomes.jsonl` | repro attempt ledger |
| `config-7.4.0-rc2-kernsight-kasan` | full kernel configuration |

`REPORT-DRAFT.md` is the upstream-report email draft for this issue.
