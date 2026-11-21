# ExtremeKernel_noksu — Tài liệu tổng hợp (d2xks)

> Một trang nắm toàn bộ: repo là gì, gồm những gì, build thế nào, chạy ra sao,
> và chỗ nào không được đụng. Dành cho người tiếp nhận bảo trì.

Cập nhật: 2026-11-21 · Nhánh phát triển: `ccr-48d3cc47-snwzwj`

---

## 1. Thân phận

| Mục | Giá trị |
|---|---|
| Repo | `izsdeaman97/ExtremeKernel_noksu` |
| Nhánh phát triển | `ccr-48d3cc47-snwzwj` |
| Kernel | Linux **4.14.356**, Samsung Galaxy **Note10+ (d2xks)**, Exynos 9825 |
| Nguồn gốc | ExtremeKernel (JackA1ltman) — họ Exynos 9820/9825 (beyond*/d*) |
| Toolchain | **Neutron clang 18** (antman tag `05012024`), ThinLTO bật |
| Root | SukiSU-Nova (KernelSU) nhúng built-in |

Kernel này build được cho 9 model trong cùng cây nguồn: `beyond0lte`,
`beyond1lte`, `beyond2lte`, `beyondx` (9820) và `d1`, `d1xks`, `d2s`, `d2x`,
`d2xks` (9825). Toàn bộ tính năng dưới đây đã được kiểm chứng **boot thật trên
d2xks**; các model khác chỉ mới xác nhận build sạch.

---

## 2. Nguồn gốc các thành phần

| Thành phần | Thượng nguồn | Phiên bản | Ghi chú |
|---|---|---|---|
| Kernel chủ | Samsung `SM-N975_10_Opensource` (ExtremeKernel) | 4.14.356 | — |
| KernelSU | **SukiSU-Nova** `builtin` @ `4bec532d` | KSU_VERSION **30007** | vendor vào `KernelSU/`, + 5 vá 4.14 |
| SUSFS | NonGKI 4.14 port (`patches/susfs_patch_to_4.14.patch`) | **v2.4.0** | lấy từ SM-A226B-SukiSU-Ultra-Kernel |
| NoMount | `maxsteeel/nomount` @ `7914458` | **v2.3.0** (base 23) | symlink `fs/nomount` |
| SELinux query filter | selinux_hook 1.1.7 (Admire), nửa "legacy" 4.14 | — | `security/selinux/selinux_query_filter.{c,h}` |
| selinux_hook_kpm | selinux_hook 1.1.7 KPM, bản built-in | — | `security/selinux/selinux_hook_kpm/` (4531 dòng) |

Tất cả phần SUSFS / NoMount / SELinux đều **đối chiếu từ repo tham chiếu
`SM-A226B-SukiSU-Ultra-Kernel`** (4.14.186), rồi thích ứng cho 4.14.356 + ThinLTO.

---

## 3. Chồng tính năng & công tắc cấu hình

Bật qua `arch/arm64/configs/ksu.config` (merge khi build với `-k Y`):

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
CONFIG_SECURITY_SELINUX_QUERY_FILTER=y       # Lọc probe DirtySepolicy
CONFIG_SECURITY_SELINUX_HOOK_KPM=y           # selinux_hook built-in (cần KALLSYMS_ALL)
```

| Lớp | Cơ chế | Điểm vào |
|---|---|---|
| SukiSU-Nova | hook thủ công kiểu SUSFS trong host (exec/open/stat/read/reboot/setresuid/input) | `KernelSU/kernel/` + host |
| SUSFS 2.4.0 | `susfs_*` + ioctl qua reboot magic | `fs/susfs.c` + ~16 host |
| NoMount 2.3.0 | `fs_initcall`, key-type "nomount" | `NoMount/kernel/src/nomount.c` |
| SELinux query filter | lọc `/sys/fs/selinux/{context,access}`, đóng băng status seqno | `selinux_query_filter.c` + selinuxfs/hooks/status |
| selinux_hook_kpm | ảnh policy sạch, ẩn đọc `/policy`, AV sạch | `selinux_hook_kpm/` + 12 trampoline |

### 12 trampoline của selinux_hook_kpm (host)

Mỗi hàm bọc dưới `#ifdef CONFIG_SECURITY_SELINUX_HOOK_KPM` (có `#else` giữ
nguyên bản gốc khi tắt). Cơ chế: KPM không vá text được khi built-in, nên hàm
gốc đổi tên thành `__kp_<tên>` và một trampoline cùng tên gọi
`kp_hook_before/after`. `selinux_hook.c` phân giải symbol qua **kallsyms lúc
`late_initcall`, có xử lý hậu tố ThinLTO** (`context_struct_compute_av.NN`).

| File | Hàm |
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
`workflows.sh` tự tải **Neutron clang 18** (antman `-S=05012024`) vào
`toolchain/neutron_18/` khi chưa có. Trên runner CI, cache gồm cả
`toolchain/neutron_18` **và** `~/.neutron-tc` (antman `--patch=glibc` đặt glibc
ở đó; thiếu nó thì clang hỏng `posix_spawn`).

### Cục bộ
```bash
bash ./workflows.sh -m d2xks -k Y -r N
# sản phẩm: build/out/d2xks/ExtremeKRNL-Nexus-d2xks.zip (AnyKernel3)
```
`MAKE_ARGS`: `LLVM=1 LLVM_IAS=1 ARCH=arm64 O=out`. Khi `-k Y`, `workflows.sh`
ghim phiên bản KSU (nếu không sẽ hỏi GitHub API lúc build):
```
KSU_VERSION=30007 KSU_VERSION_FULL=v3.0.0-4bec532d@builtin
```

### GitHub Actions
`.github/workflows/build.yml`: chạy trên `ubuntu-24.04`, matrix 5 model 9825
(có thể chọn model/ksu khi Run workflow), cache Neutron. Tự chạy khi push vào
`ccr-48d3cc47-snwzwj`; chạy tay cho nhánh khác qua tab Actions → Run workflow.

> **Lưu ý ThinLTO**: `ld.lld` của Neutron cần `libxml2.so.2`; runner phải cài
> `libxml2`. Nếu thiếu, `ld-name` tụt về `bfd` và vDSO link lỗi `-EL: not found`.

---

## 5. Bản đồ nhánh

Mỗi nhánh là một mốc **đã boot thật trên d2xks**, cộng dồn tính năng:

| Nhánh | Thành phần |
|---|---|
| `d2xks` | SukiSU-Nova 30007 |
| `d2xks-susfs` | + SUSFS v2.4.0 |
| `d2xks-susfs-nomount` | + NoMount v2.3.0 |
| `d2xks-susfs-nomount-selinux` | + SELinux query filter |
| `d2xks-susfs-nomount-selinuxhook` | + selinux_hook_kpm (**đầy đủ**) |
| `ccr-48d3cc47-snwzwj` | nhánh phát triển (= bản đầy đủ) |

---

## 6. Kiểm tra lúc chạy (root)

```sh
# NoMount (đọc sysfs trực tiếp, luôn tin cậy)
su -c 'cat /sys/module/nomount/version'          # 23  (= v2.3.0)

# SukiSU-Nova + SUSFS + selinux_hook: in lúc boot sớm, chụp NGAY sau reboot
su -c 'dmesg | grep -iE "SukiSU-Nova version|SUSFS_VERSION|selinux_hook"'
#   -- SukiSU-Nova version: 30007 [v3.0.0-4bec532d@builtin]
#   -- SUSFS_VERSION: v2.4.0
#   [selinux_hook] symbol cache resolved ... found=N
#   [selinux_hook] CLEAN policy snapshot saved ...

# Hoặc xem app manager SukiSU (version KSU, mục SUSFS)
```

Bộ đệm log kernel chỉ 2 MB nên các dòng boot bị ghi đè sau vài phút — không
thấy chúng **không** có nghĩa là lỗi. Log lần boot trước: `/sys/fs/pstore/console-ramoops-0`.

selinux_hook mặc định in trace rất dày (CLEAN/AUDIT/STATUS mỗi lần app spawn) —
đây là **chủ ý giữ nguyên**, xem §8.

---

## 7. Phần đã loại (có lý do)

| Thành phần A226B | Vì sao loại |
|---|---|
| **PathMask** (LKM kretprobe) | Dưới **ThinLTO**, kretprobe trên `__inode_permission`/`vfs_getattr_nosec` không fire cho call site bị inline → chỉ ẩn một phần (chính tác giả ghi nhận trong code). Còn cần bật `CONFIG_KPROBES` và build .ko ngoài cây buộc vermagic. Không đáng. |
| **Anti-detection** | Hai mảnh lõi riêng thiết bị: `a226b_hide_mount_line()` hardcode mount gốc A226B + UID detector 10296/10272; `init/main.c` chèn vbmeta digest `d259db70…` của riêng A226B. Port mù sẽ ẩn nhầm / chèn digest sai. Mặt chính đã được SUSFS + selinux_hook phủ. |

---

## 8. Bất biến & bài học (đọc trước khi sửa)

1. **Giữ 5 vá tương thích 4.14 của SukiSU-Nova** (`KernelSU/kernel/`): cổng KPM
   dưới 5.10, chặn `static_call.h`, `ksu_kvcalloc`, alias
   `strncpy_from_user_nofault`, `feature/selinux_hide_compat.c`. Khi cập nhật
   KSU phải giữ lại.
2. **Toolchain là Neutron clang 18 (ThinLTO)**, KHÔNG dùng clang-r383902 của
   A226B: bản clang 11 đó build xong nhưng **treo logo d2xks** (assembler clang
   11 không được hỗ trợ cho arm64 4.14). Đã thử cả integrated-as lẫn GNU as.
3. **Phiên bản KSU ghim = 30007**; không ghim thì build offline/CI bị hỏi GitHub
   API và số version "sụp" về số commit repo.
4. **selinux_hook_kpm: GIỮ log dày.** Đã thử hạ log (commit `4093f2d9b`,
   `no_printk`) → **bootloop**. Diff chỉ có log, đối số đều chỉ-đọc ⇒ các lời
   gọi `pr_info` dày trên đường nóng SELinux (setprocattr chạy rất sớm/lớn lúc
   boot) đang **che một race tiềm ẩn**; bỏ chúng làm lộ race. Đã revert
   (`d21425afa`). Muốn hạ log phải giữ printk trên đường nóng (vd. ratelimited)
   hoặc sửa race trước.
5. **Mọi tính năng SUSFS/selinux đặt dưới `#ifdef`**; build không bật
   `ksu.config` không đổi gì (byte-identical với nhánh gốc).
6. selinux_hook phân giải symbol qua kallsyms — cần `CONFIG_KALLSYMS_ALL=y`.
7. Trước khi flash: **sao lưu boot/dtb/dtbo**. Nếu treo logo: vào nhánh lưu trữ
   trước đó. Nếu một **module** manager gây bootloop (không phải kernel): vào
   **safe mode** — tắt máy, bật lại, nhấn **Volume Up 3–5 lần** để SukiSU tắt
   toàn bộ module.

---

## 9. Sơ đồ thư mục

```
KernelSU/                               SukiSU-Nova builtin@4bec532d (vendor) + 5 vá 4.14
drivers/kernelsu -> ../KernelSU/kernel  symlink
NoMount/                                NoMount v2.3.0 (maxsteeel/nomount@7914458)
fs/nomount -> ../NoMount/kernel/src     symlink
fs/susfs.c, include/linux/susfs*.h      SUSFS 2.4.0
security/selinux/selinux_query_filter.* Lớp 1 ẩn SELinux
security/selinux/selinux_hook_kpm/      Lớp 2 ẩn SELinux (built-in KPM)
include/linux/selinux_hook_kpm.h        ABI kp_hook
arch/arm64/configs/ksu.config           công tắc tính năng (merge khi -k Y)
workflows.sh                            build script (Neutron, ghim KSU 30007)
.github/workflows/build.yml             CI (ubuntu-24.04, matrix 5 model)
docs/REPO-MASTER.md                     tài liệu này
```

---

## 10. Lịch sử commit tính năng (từ base `317720047`)

```
c7b6d60a9  drivers: vendor SukiSU-Nova builtin@4bec532d (KSU_VERSION 30007)
2e21518a9  kernelsu: carry the SM-A226B 4.14 compatibility edits (5 vá)
3f64cec08  kernelsu: add SukiSU-Nova manual hooks (non-SUSFS mode)
b2db1d246  ci: build SukiSU-Nova kernels on GitHub Actions
d7b1d3ff7  build: use the SM-A226B toolchain  (sau bị thay bằng Neutron)
f0546dc3d  ci: also build on push to the SukiSU-Nova branch
fbe1b0571  ci: make the toolchain selectable (aosp / neutron)
b71aba99e  build(aosp): assemble with GNU as from GCC 4.9  (bị loại sau)
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
4093f2d9b  selinux_hook: silence per-event tracing by default   [đã REVERT]
d21425afa  Revert "selinux_hook: silence per-event tracing by default"
```
