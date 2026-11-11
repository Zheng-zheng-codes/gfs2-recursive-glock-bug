# kernel BUG in gfs2_glock_nq (fs/gfs2/glock.c:1476) via recursive glock holder

Syzkaller-found bug on Linux v7.4-rc2 (also observed on v7.4-rc1),
reproduced and verified on the same kernel build.

## Summary

Acquiring glock holders for resource groups from the statfs slow path trips
GFS2's recursive-holder detection:

```
------------[ cut here ]------------
kernel BUG at fs/gfs2/glock.c:1476!
Oops: invalid opcode: 0000 [#1] SMP KASAN NOPTI
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

The BUG is triggered in `add_to_queue()` while `gfs2_statfs_slow()` is
acquiring glock holders for resource groups. The recursive-holder check
detects that the same task is attempting to queue another holder for a
glock that already has a holder owned by that task.

The console diagnostics captured during the verified run support this
reading (see `logs/crash-serial-excerpt.txt`): the dumped glock is a
resource-group glock (`G: s:SH n:3/22 ... t:SH`), both the "original" and
the "new" holders belong to the same task (pid 17923,
`statfs_by_dentry+0x13d/0x220`, lock type 3 = resource group, requested
state 3 = shared).

The resource-group iteration in `gfs2_statfs_slow()` may revisit a resource
group before its previous holder has been released, causing the recursive
acquisition. This is a possible explanation based on the source code; the
exact cause in our reproduction has not yet been confirmed.

## How to reproduce (stable)

Environment: Debian trixie x86_64 guest (6 GiB RAM, 4 vCPU, KVM), kernel
built from v7.4-rc2 with CONFIG_KASAN=y, CONFIG_PROVE_LOCKING=y,
CONFIG_GFS2_FS=y, booted with `oops=panic` and `panic_on_warn=1`. See
`environment.txt` for the exact QEMU command line and `kernel.config` for
the configuration. The syzkaller build used is commit
`853b104bb454141a5b0fc0e4218ec78e9f34ffb1`.

The crash depends on glock-holder state accumulated by the preceding
programs of the crash log, so the reproducer contains 107 programs stored
in their original order (1731 lines) and replayed concurrently using
-procs=8 and -repeat=0:

```sh
# in the guest, as root:
syz-execprog -procs=8 -threaded -repeat=0 -executor ./syz-executor repro.kernsight.prog
```

`reproducer/repro.kernsight.prog` first replays the state-building tail
(mounting a gfs2 image and performing lock-heavy operations) and then
triggers `statfs()`. Run it as-is in a loop (`-repeat=0`).

Expected time to hit: roughly **3-5 minutes**. In the standalone
verification run the kernel BUG fired at 290.6 s after guest boot, which is
~250 s (~4 min 10 s) after syz-execprog started (guest boot completed
~40 s before the reproducer). The fuzzing pipeline's state-primed
reproduction sessions reproduced the same crash repeatedly on freshly
replayed VMs, typically within 4-10 minutes of session start. With
`-repeat=0` the program set keeps looping until the crash fires. All
fresh-VM attempts so far have hit; the stable success rate over many runs
has not been formally measured.

`reproducer/repro.prog` is a syzkaller-extracted single-call program
(`bpf$PROG_LOAD`); it does **not** reproduce the GFS2 crash independently —
use `repro.kernsight.prog`. It is included only as original evidence of
what the pipeline minimiser extracted for this crash title.

## Files

- `kernel.config` — kernel configuration used (also required to interpret addresses)
- `environment.txt` — kernel image info (sha256), QEMU command line, syzkaller commit, reproducer command
- `reproducer/repro.kernsight.prog` — pipeline-verified reproducer (107-program ordered set: state tail + triggering statfs)
- `reproducer/repro.prog` — syzkaller-extracted single-call program (bpf$PROG_LOAD); does not reproduce the GFS2 crash independently
- `reproducer/repro.priming.prog` — state-priming program set (pipeline repro recipe)
- `logs/log-*.txt`, `logs/report-*.txt` — original syzkaller execution logs and crash reports from the fuzzing pipeline
- `logs/machineInfo-*.txt`, `logs/title-stat.json` — guest environment info and title statistics
- `logs/crash-serial-excerpt.txt` — serial console excerpt of the verified crash, including the glock holder diagnostics (original/new holders, lock type) and the BUG
- `logs/verification-serial.log` — full serial console of an independent verification run on a fresh VM (BUG at 290.6 s after boot)

## Notes for maintainers

- What is confirmed: during the statfs slow path, the same task ended up
  queueing a second shared holder for a resource-group glock that already
  had a holder owned by that task, and `add_to_queue()` BUGs on it. The
  diagnostics at the top of `logs/crash-serial-excerpt.txt` show both the
  "original" and the "new" holder (`statfs_by_dentry`, pid 17923) and the
  glock dump (`n:3/22`, lock type 3).
- Suspected mechanism (based on the source, not yet confirmed): the
  resource-group iteration in `gfs2_statfs_slow()`
  (`gfs2_rgrpd_get_first()`/`gfs2_rgrpd_get_next()`) wraps around at the
  end of the resource-group tree, so a resource group can be revisited
  before its previous holder is released; with up to 64 asynchronous
  holder slots, a file system with a small number of resource groups could
  re-queue the same glock within one pass. The number of resource groups
  in the reproduced image has not yet been checked against this theory.
- We can provide the kernel image, additional crash instances or raw
  artifacts on request.
