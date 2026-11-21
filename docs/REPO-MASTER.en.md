# ExtremeKernel_noksu — Master guide (d2xks)

> One page to understand the repo: what it is, what it is made of, how to build
> it, how it behaves at runtime, and what not to touch. For maintainers.
>
> 🌐 Tiếng Việt: [REPO-MASTER.md](REPO-MASTER.md)

Updated: 2026-11-21 · Dev branch: `ccr-48d3cc47-snwzwj`

---

## 1. Identity

| Item | Value |
|---|---|
| Repo | `izsdeaman97/ExtremeKernel_noksu` |
| Dev branch | `ccr-48d3cc47-snwzwj` |
| Kernel | Linux **4.14.356**, Samsung Galaxy **Note10+ (d2xks)**, Exynos 9825 |
| Lineage | ExtremeKernel (JackA1ltman) — Exynos 9820/9825 family (beyond*/d*) |
| Toolchain | **Neutron clang 18** (antman tag `05012024`), ThinLTO on |
| Root | SukiSU-Nova (KernelSU), built in |

The same source tree builds 9 models: `beyond0lte`, `beyond1lte`, `beyond2lte`,
`beyondx` (9820) and `d1`, `d1xks`, `d2s`, `d2x`, `d2xks` (9825). The whole
feature stack below has been **verified booting on real d2xks hardware**; the
other models are only confirmed to build cleanly.

---

## 2. Component lineage

| Component | Upstream | Version | Notes |
|---|---|---|---|
| Host kernel | Samsung `SM-N975_10_Opensource` (ExtremeKernel) | 4.14.356 | — |
| KernelSU | **SukiSU-Nova** `builtin` @ `4bec532d` | KSU_VERSION **30007** | vendored into `KernelSU/`, + 5 x 4.14 edits |
| SUSFS | NonGKI 4.14 port (`patches/susfs_patch_to_4.14.patch`) | **v2.4.0** | from SM-A226B-SukiSU-Ultra-Kernel |
| NoMount | `maxsteeel/nomount` @ `7914458` | **v2.3.0** (base 23) | `fs/nomount` symlink |
| SELinux query filter | selinux_hook 1.1.7 (Admire), 4.14 "legacy" half | — | `security/selinux/selinux_query_filter.{c,h}` |
| selinux_hook_kpm | selinux_hook 1.1.7 KPM, built-in port | — | `security/selinux/selinux_hook_kpm/` (4531 lines) |

All SUSFS / NoMount / SELinux pieces are **ported from the reference repo
`SM-A226B-SukiSU-Ultra-Kernel`** (4.14.186) and adapted to 4.14.356 + ThinLTO.

---

## 3. Feature stack & config switches

Enabled via `arch/arm64/configs/ksu.config` (merged when building with `-k Y`):

```
CONFIG_KSU=y                                 # SukiSU-Nova (root)
CONFIG_KSU_ALLOWLIST_WORKAROUND=y
CONFIG_KSU_SUSFS=y                           # SUSFS v2.4.0 ...
CONFIG_KSU_SUSFS_SUS_PATH=y
CONFIG_KSU_SUSFS_SUS_MOUNT=y
CONFIG_KSU_SUSFS_SUS_KSTAT=y
CONFIG_KSU_SUSFS_SPOOF_UNAME=y
CONFIG_KSU_SUSFS_ENABLE_LOG=y
CONFIG_KSU_SUSFS_HIDE_KSU_SUSFS_SYMBOLS=y
CONFIG_KSU_SUSFS_SPOOF_CMDLINE_OR_BOOTCONFIG=y
CONFIG_KSU_SUSFS_OPEN_REDIRECT=y
CONFIG_KSU_SUSFS_SUS_MAP=y
CONFIG_KSU_SUSFS_PRIORITY_UID_FILTER=y
CONFIG_NOMOUNT=y                             # NoMount v2.3.0
CONFIG_SECURITY_SELINUX_QUERY_FILTER=y       # DirtySepolicy probe filter
CONFIG_SECURITY_SELINUX_HOOK_KPM=y           # built-in selinux_hook (needs KALLSYMS_ALL)
```

| Layer | Mechanism | Entry |
|---|---|---|
| SukiSU-Nova | SUSFS-style manual host hooks (exec/open/stat/read/reboot/setresuid/input) | `KernelSU/kernel/` + host |
| SUSFS 2.4.0 | `susfs_*` + ioctl via reboot magic | `fs/susfs.c` + ~16 host |
| NoMount 2.3.0 | `fs_initcall`, "nomount" key-type | `NoMount/kernel/src/nomount.c` |
| SELinux query filter | filter `/sys/fs/selinux/{context,access}`, freeze status seqno | `selinux_query_filter.c` + selinuxfs/hooks/status |
| selinux_hook_kpm | clean policy snapshot, hide `/policy` reads, clean AV | `selinux_hook_kpm/` + 12 trampolines |

### The 12 selinux_hook_kpm trampolines (host side)

Each wrapped function is guarded by `#ifdef CONFIG_SECURITY_SELINUX_HOOK_KPM`
(with an `#else` keeping the original when off). Mechanism: a built-in KPM
cannot patch text, so the original function is renamed `__kp_<name>` and a
same-named trampoline calls `kp_hook_before/after`. `selinux_hook.c` resolves
symbols via **kallsyms at `late_initcall`, handling ThinLTO suffixes**
(`context_struct_compute_av.NN`).

| File | Functions |
|---|---|
| `fs/libfs.c` | `simple_read_from_buffer` |
| `security/security.c` | `security_setprocattr` |
| `security/selinux/ss/services.c` | `context_struct_compute_av`, `string_to_context_struct`, `security_read_policy` |
| `security/selinux/ss/status.c` | `selinux_status_update_policyload` |
| `security/selinux/hooks.c` | `selinux_setprocattr`, `selinux_complete_init` |
| `security/selinux/selinuxfs.c` | `sel_read_handle_status`, `sel_mmap_handle_status`, `sel_write_context`, `sel_write_access` |

---

## 4. Build

### Toolchain
`workflows.sh` fetches **Neutron clang 18** (antman `-S=05012024`) into
`toolchain/neutron_18/` if missing. On CI the cache must include both
`toolchain/neutron_18` **and** `~/.neutron-tc` (antman `--patch=glibc` installs
a glibc there; without it clang fails with `posix_spawn`).

### Local
```bash
bash ./workflows.sh -m d2xks -k Y -r N
# output: build/out/d2xks/ExtremeKRNL-Nexus-d2xks.zip (AnyKernel3)
```
`MAKE_ARGS`: `LLVM=1 LLVM_IAS=1 ARCH=arm64 O=out`. With `-k Y`, `workflows.sh`
pins the KSU version (otherwise it queries the GitHub API at build time):
```
KSU_VERSION=30007 KSU_VERSION_FULL=v3.0.0-4bec532d@builtin
```

### GitHub Actions
`.github/workflows/build.yml`: runs on `ubuntu-24.04`, a 5-model 9825 matrix
(model/ksu selectable via Run workflow), caches Neutron. Fires on push to
`ccr-48d3cc47-snwzwj`; run manually for other branches via Actions → Run workflow.

> **ThinLTO note**: Neutron's `ld.lld` needs `libxml2.so.2`; the runner must
> install `libxml2`. Without it `ld-name` falls back to `bfd` and the vDSO link
> fails with `-EL: not found`.

---

## 5. Branch map

Each branch is a milestone **booting on real d2xks**, cumulatively:

| Branch | Content |
|---|---|
| `d2xks` | SukiSU-Nova 30007 |
| `d2xks-susfs` | + SUSFS v2.4.0 |
| `d2xks-susfs-nomount` | + NoMount v2.3.0 |
| `d2xks-susfs-nomount-selinux` | + SELinux query filter |
| `d2xks-susfs-nomount-selinuxhook` | + selinux_hook_kpm (**full**) |
| `ccr-48d3cc47-snwzwj` | dev branch (= the full build) |

---

## 6. Runtime verification (root)

```sh
# NoMount (direct sysfs read, always reliable)
su -c 'cat /sys/module/nomount/version'          # 23  (= v2.3.0)

# SukiSU-Nova + SUSFS + selinux_hook: printed early at boot, capture RIGHT after reboot
su -c 'dmesg | grep -iE "SukiSU-Nova version|SUSFS_VERSION|selinux_hook"'
#   -- SukiSU-Nova version: 30007 [v3.0.0-4bec532d@builtin]
#   -- SUSFS_VERSION: v2.4.0
#   [selinux_hook] symbol cache resolved ... found=N
#   [selinux_hook] CLEAN policy snapshot saved ...

# Or use the SukiSU manager app (KSU version, SUSFS section)
```

The kernel log ring is only 2 MB, so boot-time lines are overwritten after a few
minutes — not seeing them does **not** mean failure. Previous-boot log:
`/sys/fs/pstore/console-ramoops-0`.

selinux_hook emits very chatty per-event tracing (CLEAN/AUDIT/STATUS on every
app spawn) by design — see §8.

---

## 7. Excluded (with reasons)

| A226B component | Why excluded |
|---|---|
| **PathMask** (kretprobe LKM) | Under **ThinLTO**, kretprobes on `__inode_permission`/`vfs_getattr_nosec` do not fire for inlined call sites → only partial path hiding (the author acknowledges this in the code). It also needs `CONFIG_KPROBES` and an out-of-tree, vermagic-bound `.ko`. Not worth it. |
| **Anti-detection** | The two core pieces are device-specific: `a226b_hide_mount_line()` hardcodes A226B stock mounts + detector UIDs 10296/10272; `init/main.c` injects A226B's own vbmeta digest `d259db70…`. Porting blind would hide the wrong mounts / inject a wrong digest. The main surface is already covered by SUSFS + selinux_hook. |

---

## 8. Invariants & lessons (read before editing)

1. **Keep SukiSU-Nova's 5 x 4.14 compat edits** (`KernelSU/kernel/`): KPM gate
   below 5.10, `static_call.h` guard, `ksu_kvcalloc`, `strncpy_from_user_nofault`
   alias, `feature/selinux_hide_compat.c`. Preserve them when updating KSU.
2. **Toolchain is Neutron clang 18 (ThinLTO)**, NOT A226B's clang-r383902: that
   clang 11 builds fine but **hangs at the d2xks logo** (clang 11 integrated
   assembler is unsupported for arm64 4.14). Tried both integrated-as and GNU as.
3. **KSU version is pinned to 30007**; unpinned, offline/CI builds query the
   GitHub API and the version "collapses" to the repo commit count.
4. **selinux_hook_kpm: KEEP the verbose logging.** Lowering it (commit
   `4093f2d9b`, `no_printk`) caused a **bootloop**. The diff was logging-only and
   all log args are read-only ⇒ the frequent `pr_info` calls on the hot SELinux
   paths (setprocattr runs very early/often at boot) were **masking a latent
   race**; removing them exposes it. Reverted in `d21425afa`. To reduce the log,
   keep printk on the hot path (e.g. ratelimited) or fix the race first.
5. **Every SUSFS/selinux feature is behind `#ifdef`**; a build without
   `ksu.config` is byte-identical to the base branch.
6. selinux_hook resolves symbols via kallsyms — requires `CONFIG_KALLSYMS_ALL=y`.
7. Before flashing: **back up boot/dtb/dtbo**. On a logo hang: use the previous
   archive branch. If a manager **module** (not the kernel) bootloops: enter
   **safe mode** — power off, power on, press **Volume Up 3–5 times** so SukiSU
   disables all modules.

---

## 9. Directory map

```
KernelSU/                               SukiSU-Nova builtin@4bec532d (vendored) + 5 x 4.14 edits
drivers/kernelsu -> ../KernelSU/kernel  symlink
NoMount/                                NoMount v2.3.0 (maxsteeel/nomount@7914458)
fs/nomount -> ../NoMount/kernel/src     symlink
fs/susfs.c, include/linux/susfs*.h      SUSFS 2.4.0
security/selinux/selinux_query_filter.* SELinux hiding layer 1
security/selinux/selinux_hook_kpm/      SELinux hiding layer 2 (built-in KPM)
include/linux/selinux_hook_kpm.h        kp_hook ABI
arch/arm64/configs/ksu.config           feature switches (merged with -k Y)
workflows.sh                            build script (Neutron, pins KSU 30007)
.github/workflows/build.yml             CI (ubuntu-24.04, 5-model matrix)
docs/REPO-MASTER.md / .en.md            this guide (VI / EN)
```

---

## 10. Feature commit history (since base `317720047`)

```
c7b6d60a9  drivers: vendor SukiSU-Nova builtin@4bec532d (KSU_VERSION 30007)
2e21518a9  kernelsu: carry the SM-A226B 4.14 compatibility edits (5 edits)
3f64cec08  kernelsu: add SukiSU-Nova manual hooks (non-SUSFS mode)
b2db1d246  ci: build SukiSU-Nova kernels on GitHub Actions
d7b1d3ff7  build: use the SM-A226B toolchain  (later replaced by Neutron)
f0546dc3d  ci: also build on push to the SukiSU-Nova branch
fbe1b0571  ci: make the toolchain selectable (aosp / neutron)
b71aba99e  build(aosp): assemble with GNU as from GCC 4.9  (later dropped)
990560110  ci: run the build job on ubuntu-24.04
93322fa95  build: default to the Neutron clang 18 toolchain
f9e573bd2  build: drop the clang-r383902 (aosp) toolchain, use Neutron only
aef1d2501  fs: integrate SUSFS v2.4.0 (4.14 non-GKI port)
76da0954c  kernelsu: switch to SUSFS inline hooks and enable SUSFS v2.4.0
713aa8cc1  ci: cache ~/.neutron-tc together with the Neutron toolchain
d38c456e3  fs: integrate NoMount v2.3.0
4344ac848  selinux: add native query filter to hide dirty policy probes
d3a332d47  selinux: vendor the built-in selinux_hook KPM port (part 1: subtree)
d355e77a2  selinux: wire selinux_hook_kpm trampolines into the host (part 2)
4093f2d9b  selinux_hook: silence per-event tracing by default   [REVERTED]
d21425afa  Revert "selinux_hook: silence per-event tracing by default"
```
