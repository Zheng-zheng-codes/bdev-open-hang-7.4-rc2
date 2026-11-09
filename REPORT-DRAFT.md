Subject: INFO: task hung in bdev_open() while opening block devices on Linux 7.4-rc2 (syzkaller, reproducible)

Hi block maintainers,

I encountered an `INFO: task hung in bdev_open()` while running syzkaller on
Linux 7.4-rc2. Two tasks were blocked for more than 143 seconds each, both
waiting on the same gendisk `open_mutex` inside `bdev_open()`
(block/bdev.c:990) while opening block device nodes via `openat()`. The hang
reproduces with a single minimized syzkaller program on fresh VMs.

**All raw logs, crash reports, the reproducer and the kernel/fuzzing
configurations are available at:**

  https://github.com/Zheng-zheng-codes/bdev-open-hang-7.4-rc2

Everything referenced below by relative path lives in that repository.

**Kernel information:**

- Kernel version: 7.4.0-rc2-kernsight-kasan #1
- Base: v7.4-rc2 (`7475ea8b5`); the block layer (`block/bdev.c`) is
  unmodified upstream code
- Architecture: x86_64
- Environment: QEMU Standard PC (i440FX + PIIX), KVM, systemd-based guest
  (systemd-udevd present and participating in the hang)
- Configuration: KASAN; `hung_task_panic=0`, so the machine kept running and
  the dump shows the blocked tasks; full .config is included in the repo
- Fuzzing framework: syzkaller (configuration including the guest image and
  the full kernel command line is included as `syzkaller.cfg`)

**Hang:**

```text
INFO: task (udev-worker):9713 blocked for more than 143 seconds.
      Not tainted 7.4.0-rc2-kernsight-kasan #1
task:(udev-worker)   state:D stack:25208 pid:9713  tgid:9713  ppid:4787
Call Trace:
 <TASK>
 __schedule+0x1425/0x6840 kernel/sched/core.c:7389
 schedule+0xec/0x2d0 kernel/sched/core.c:7481
 __mutex_lock_common kernel/locking/mutex.c:726 [inline]
 __mutex_lock+0xe0c/0x1dd0 kernel/locking/mutex.c:821
 bdev_open+0x3fa/0xe50 block/bdev.c:990
 blkdev_open+0x355/0x500 block/fops.c:686
 do_dentry_open+0x693/0x14f0 fs/open.c:1000
 vfs_open_consume+0xd1/0x460 fs/open.c:1131
 do_open fs/namei.c:4851 [inline]
 path_openat+0x1c1b/0x2240 fs/namei.c:5016
 do_file_open+0x21b/0x470 fs/namei.c:5045
 do_sys_openat2+0x13e/0x1d0 fs/open.c:1443
 __x64_sys_openat+0x144/0x200 fs/open.c:1460
 entry_SYSCALL_64_after_hwframe+0x77/0x7f
 </TASK>

INFO: task syz.2.29:12841 blocked for more than 143 seconds.
task:syz.2.29        state:D stack:27784 pid:12841 tgid:12818 ppid:10317
Call Trace:
 <TASK>
 (identical stack: __mutex_lock -> bdev_open block/bdev.c:990 -> blkdev_open
  -> do_dentry_open -> ... -> __x64_sys_openat)
 </TASK>
```

**Observations:**

- `bdev_open()` serializes device opening with `mutex_lock(&disk->open_mutex)`
  (block/bdev.c:990); both blocked tasks are in that mutex, opening block
  device nodes via `openat()`.
- The mutex holder is not visible in the hung-task dump. Given that one of the
  waiters is a udev worker reacting to device events and the other is a
  syzkaller executor process, we suspect a third opener held `open_mutex` for
  a very long time (for example a task stuck in a slow claim/holder setup or
  slow loop-device I/O while holding the mutex).
- The hang was observed twice during concurrent fuzzing, and the state-primed
  verification flow reproduced it 2/2 times on freshly replayed VMs with the
  minimized program, so this does not look like a one-off scheduling hiccup.

**Reproduction (step by step):**

1. Kernel: any `v7.4-rc2` x86_64 build with KASAN enabled reproduces; our
   exact configuration is `config-7.4.0-rc2-kernsight-kasan` in the repository (KASAN inline instrumentation, hung-task detector enabled with
   the default 120 s timeout, `CONFIG_LOCALVERSION=-kernsight-kasan`).
2. Guest: boot a systemd-based x86_64 root filesystem (we used a Debian
   trixie image; systemd-udevd must be present and active, since a udev
   worker is one of the two hung tasks) in QEMU, e.g.:
   `qemu-system-x86_64 -m 6144 -smp 4 -enable-kvm -cpu host -drive file=rootfs.img,if=ide -append "root=/dev/sda console=ttyS0 ..."`.
   The full command line we used is in `syzkaller.cfg` (`vm.cmdline`);
   `panic_on_warn=1` and `hung_task_panic=0` are set, so the hang shows up as
   hung-task INFO dumps and the VM keeps running.
3. Copy `syz-execprog` and `syz-executor` (built from any recent syzkaller
   checkout for linux/amd64) into the guest, together with
   `repro/repro.prog`.
4. Run in the guest:

   ```bash
   ./syz-execprog -procs=8 -threaded -repeat=1 -executor ./syz-executor ./repro.prog
   ```

   `repro/repro.kernsight.prog` is the same crashing program preceded by the
   recorded state-reconstruction recipe; it is the more robust variant if the
   bare program alone does not trigger on your setup
   (`./syz-execprog -repeat=1 ./repro.kernsight.prog`).

5. Expected result: within roughly 2-4 minutes of VM uptime, hung-task dumps
   appear on the console: two tasks (a `udev-worker` process and a
   syz-executor process) blocked for >120 s in `bdev_open()` at
   block/bdev.c:990, waiting on the same gendisk `open_mutex`. The guest keeps
   running (hung_task_panic=0).

- Reproduction results: 2 successful / 2 attempts on freshly replayed VMs
  (kernel uptime at the hang report: ~4 minutes); additionally observed twice
  during concurrent fuzzing.

**What is in the repository above:**

| file | content |
|---|---|
| `repro/repro.prog` | minimized single-program reproducer (plain syzkaller program text) |
| `repro/repro.kernsight.prog` | same crashing program preceded by the recorded state-reconstruction recipe, in execution order |
| `repro/repro.priming.prog` | the recipe alone (run it first, then `repro.prog` on the same VM) |
| `repro/repro.README.txt` | notes on how the reproducer was produced and how to run it |
| `repro/repro.report` | hung-task dump captured during the verification run |
| `repro/repro.log` | verification-run execution log |
| `repro/repro.state.json`, `repro/repro.context.json` | measured 539-D guest state right before the hang; reproduction context and metadata |
| `repro/repro.attempts.json`, `repro/repro.outcomes.jsonl` | automatic repro attempt ledger (1 attempt, verified) |
| `logs/log0`, `logs/log1` | raw syzkaller crash logs from the two fuzzing occurrences (with the executing-program context) |
| `logs/report0`, `logs/report1` | the corresponding hung-task crash reports |
| `logs/machineInfo0`, `logs/machineInfo1` | guest machine information |
| `logs/pre_state0`, `logs/pre_state1` | 539-D pre-crash guest state vectors captured by the fuzzing pipeline |
| `logs/crash-excerpt.txt` | tail of the verification-run report |
| `config-7.4.0-rc2-kernsight-kasan` | full kernel configuration |
| `syzkaller.cfg` | fuzzing configuration: guest image, full kernel command line, QEMU arguments, VM layout |

Could you please advise whether this corresponds to a known issue (for
example an open_mutex holder waiting on a device that never becomes ready),
or whether additional diagnostics from the VM would be useful?

Thanks,
Zheng
