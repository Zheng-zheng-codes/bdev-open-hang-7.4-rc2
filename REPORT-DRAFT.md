Subject: INFO: task hung in bdev_open() while opening block devices on Linux 7.4-rc2 (syzkaller, reproducible)

Hi block maintainers,

I encountered an `INFO: task hung in bdev_open()` while running syzkaller on
Linux 7.4-rc2. Two tasks were blocked for more than 143 seconds each, both
waiting on the same gendisk `open_mutex` inside `bdev_open()`
(block/bdev.c:990) while opening block device nodes via `openat()`. The hang
reproduces with a single minimized syzkaller program on fresh VMs.

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

**Reproduction:**

- Minimized single-program reproducer: `repro/repro.prog` (program text) —
  run `syz-execprog -repeat=1 repro.prog` on a fresh VM with the same kernel
  image; `repro/repro.kernsight.prog` additionally prepends the
  state-reconstruction recipe that was recorded before the hang.
- Reproduction results: 2 successful / 2 attempts on freshly replayed VMs
  (kernel uptime at the hang report: ~4 minutes).

**Reproduction and logs:**

- Full kernel log: see `logs/log0`, `logs/log1` and `repro/repro.log`
- Syzkaller reproducer: `repro/repro.prog` (minimized) and
  `repro/repro.kernsight.prog` (recipe + program)
- Kernel configuration: `config-7.4.0-rc2-kernsight-kasan`
- Reproduction results: 2/2 verified on fresh VMs, observed twice during
  fuzzing

Could you please advise whether this corresponds to a known issue (for
example an open_mutex holder waiting on a device that never becomes ready),
or whether additional diagnostics from the VM would be useful?

Thanks,
Zheng
