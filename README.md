# kernel BUG in gfs2_glock_nq (fs/gfs2/glock.c:1476) via recursive glock holder

Syzkaller-found bug on Linux v7.4-rc2 (also observed on v7.4-rc1),
reproduced and verified on the same kernel build.

## Summary

Acquiring a glock for statfs trips GFS2's recursive-holder detection:

```
------------[ cut here ]------------
kernel BUG at fs/gfs2/glock.c:1476!
Oops: invalid opcode: 0000 [#1] SMP KASAN NOPTI
CPU: 1 UID: 0 PID: 27376 Comm: syz.7.24 Not tainted 7.4.0-rc1-kernsight-kasan+ #3
RIP: 0010:add_to_queue fs/gfs2/glock.c:1476 [inline]
RIP: 0010:gfs2_glock_nq+0x6ca/0x17b0 fs/gfs2/glock.c:1518
Call Trace:
 gfs2_glock_nq_init fs/gfs2/glock.h:253 [inline]
 gfs2_statfs_slow fs/gfs2/super.c:900 [inline]
 gfs2_statfs+0x45a/0xcd0 fs/gfs2/super.c:974
 statfs_by_dentry+0x13d/0x220 fs/statfs.c:66
 vfs_statfs fs/statfs.c:90 [inline]
 user_statfs+0xc3/0x190 fs/statfs.c:106
 __do_sys_statfs+0x79/0xf0 fs/statfs.c:194
```

`add_to_queue()` (fs/gfs2/glock.c) detects that the task is trying to queue
a holder for a glock it already holds in an incompatible state and calls
`BUG()` from `trap_recursive()` (glock.c:1476). Here the statfs slow path
(`gfs2_statfs_slow()`, fs/gfs2/super.c:900) acquires the same glock twice
within one task: it first grabs a holder (statfs glock via
`gfs2_glock_nq_init()`), and while that holder is still held requests the
same glock again, which the recursive-holder check rejects.

## How to reproduce (stable)

Environment: Debian trixie x86_64 guest (6 GiB RAM, 4 vCPU, KVM), kernel
built from v7.4-rc2 with CONFIG_KASAN=y, CONFIG_PROVE_LOCKING=y,
CONFIG_GFS2_FS=y, booted with `oops=panic` and `panic_on_warn=1`. See
`environment.txt` for the exact QEMU command line and `kernel.config` for
the configuration. The syzkaller build used is commit
`853b104bb454141a5b0fc0e4218ec78e9f34ffb1`.

The crash depends on glock-holder state accumulated by the preceding
programs of the crash log, so the reproducer is an ordered program set of
107 programs (1731 lines) rather than a single call:

```sh
# in the guest, as root:
syz-execprog -procs=8 -threaded -repeat=0 -executor ./syz-executor repro.kernsight.prog
```

`reproducer/repro.kernsight.prog` first replays the state-building tail
(mounting a gfs2 image and performing lock-heavy operations) and then
triggers `statfs()`. Run it as-is in a loop (`-repeat=0`); do not try to
replay only the tail, and do not expect the minimal testcase alone
(`repro.prog`, 2 calls) to work — it depends on the state above and did
not reproduce alone in testing.

Verification on fresh VMs (kernel BUG at fs/gfs2/glock.c:1476):

- 1/1 standalone fresh-VM run: BUG fired at 290.6 s after guest boot
  (~4 min wall including guest boot; full console in
  `logs/verification-serial.log`).
- The fuzzing pipeline's state-primed reproduction sessions reproduced the
  same crash repeatedly (crash-log replay reaching entry 107/107) on
  freshly replayed VMs.
- The stable success rate over many runs has not been formally measured;
  all fresh-VM attempts so far have hit.

## Files

- `kernel.config` — kernel configuration used (also required to interpret addresses)
- `environment.txt` — kernel image info (sha256), QEMU command line, syzkaller commit, reproducer command
- `reproducer/repro.kernsight.prog` — pipeline-verified reproducer (107-program ordered set: state tail + triggering statfs)
- `reproducer/repro.prog` — minimal triggering testcase (state-dependent; does not reproduce alone)
- `reproducer/repro.priming.prog` — state-priming program set (pipeline repro recipe)
- `logs/log-*.txt`, `logs/report-*.txt` — original syzkaller execution logs and crash reports from the fuzzing pipeline
- `logs/machineInfo-*.txt`, `logs/title-stat.json` — guest environment info and title statistics
- `logs/crash-serial-excerpt.txt` — serial console tail of a verified crash
- `logs/verification-serial.log` — full serial console of an independent verification run on a fresh VM (BUG at 290.6 s after boot)

## Notes for maintainers

- The trigger is `statfs()` on a gfs2 mount whose glock holders were left
  in a particular state by prior operations (`gfs2_statfs_slow()` loops
  acquiring/releasing the statfs glock and iopen/inode glocks).
- No C reproducer is included: syzkaller's reproducer pipeline could not
  produce one for this bug (the C form of the full recipe and the C form of
  the standalone testcase both failed verification 3/3 attempts), so the
  ordered syz program set above is the reproducer of record.
- We can provide the kernel image, additional crash instances or raw
  artifacts on request.
