# Llaminar MI50/PEX P2P: tested configuration and findings

Research checkpoint: **2026-11-07**. Contributed from
[Llaminar's investigation record][research], pinned to commit
`a5d3ee32afdf4a3c5de3a5c243191b2842871995`. This case study also records the
public PEX module source and the subsequent Llaminar peer-channel correction;
the pinned investigation predates that correction.

**Result:** BIOS **MMIO High Base = 4T**, unchanged **1024G** granularity,
**identity DMA (`intel_iommu=on iommu=pt`)**, and the kernel/driver/module stack
below passed all **20 directed native AMD peer reads**. Each direction verified
a varying 64 KiB payload, completed cleanup, and retired with no driver errors
in its observed window.

This is a separate Intel VT-d/PEX topology from the two-card X99 machine in the
[main write-up](../README.md). It has four MI50s, a 7900 XTX, and two PEX880xx
switch trees (`1000:c010`). Two RTX 3090s were also installed, but NVIDIA peer
access was outside this experiment. The successful AMD directions comprise all
12 MI50-to-MI50 pairs and all eight MI50/XTX pairs, including paths across
switches/roots. The four MI50s share the `85:00/86:00` upstream tree and IOMMU
group 36; the nearest pair, `97:00.0` and `9a:00.0`, is under `93:00.0`. The XTX
at `49:00.0` is under the other root. These are this host's PCI addresses.

**The initial captured peer-channel mismatch was resolved in Llaminar.**
[Corrected replays](#llaminar-captured-peer-channel-correction) pass on the
original MI50 pair in both device orders. Lowering MMIO and selecting identity
DMA were tested together, so neither change is independently proven causal.
The native reads and corrected captured-channel replays have separate scopes;
they do not by themselves establish every collective/model path or throughput.

## BIOS and effective Linux configuration

The recorded firmware is **AMI Aptio Setup Utility (2025)**, PCI Bus Driver
**A5.01.18**. The only reported BIOS change during this experiment was
**MMIO High Base: 16T → 4T**. Firmware labels are reproduced literally; address
sizes and limits below use binary units.

| Setting | Successful experiment |
|---|---|
| MMIO High Base | **4T** |
| MMIO High Granularity Size | **1024G**, unchanged |
| Above 4G Decoding | Enabled |
| SR-IOV / ARI | Enabled / enabled |
| Intel VT-d / Interrupt Remapping | Enabled / enabled; Linux confirms IRQ remapping in x2APIC mode |
| BIOS ACS Control | Disabled; existing targeted Linux ACS-redirection configuration retained |
| BIOS PassThrough DMA / ATS | Enabled / enabled |
| Linux DMA-domain selection | `intel_iommu=on iommu=pt`; all five AMD GPUs actually report `identity` domains in sysfs |
| AMD P2P parameter | `/sys/module/amdgpu/parameters/pcie_p2p` reports `Y` |
| PCI resource arguments | Existing `pci=realloc,assign-busses` retained |
| PEX resource configuration | `apply=1 bar_gib=32 slot_gib=64 empty_min_width=8` |

The successful boot uses neither `pci=noats` nor `iommu=off`. Identity DMA
removes I/O address translation for these domains while interrupt remapping
remains enabled. A BIOS label such as “PassThrough DMA” does not establish the
effective Linux domain; the earlier failing boots still used translated
`DMA-FQ` domains. The inspected MI50 paths had ACS request/completion
redirection cleared; the XTX's other-root ACS controls differ.

The firmware offered MMIO High Base values `1T`, `4T`, `16T`, `24T`, `40T`,
and `56T`, and granularity values `1G`, `4G`, `16G`, `64G`, `256G`, and
`1024G`. Available values are not proof of device addressability. The
[full BIOS transcription][bios-record] preserves the other unchanged settings.
The table above is the relevant configuration record, not a complete replacement
kernel command line or a portable set of ACS selectors for another board.

### Final VRAM BAR0 placement

Read the final resources **after `pex_window` has applied its transaction**.
Early PCI enumeration can show different addresses from those imported later
by the GPU driver.

| GPU | Inclusive physical BAR0 range | Aperture size |
|---|---|---|
| 7900 XTX `49:00.0` | `0x64000000000–0x647ffffffff` | 32 GiB |
| MI50 `8c:00.0` | `0x90000000000–0x907ffffffff` | 32 GiB |
| MI50 `91:00.0` | `0x91000000000–0x917ffffffff` | 32 GiB |
| MI50 `97:00.0` | `0x93000000000–0x937ffffffff` | 32 GiB |
| MI50 `9a:00.0` | `0x94000000000–0x947ffffffff` | 32 GiB |

All five complete ranges fit below `0x100000000000`, the exclusive **44-bit /
16 TiB** limit. The highest observed System RAM address is `0xc03fffffff`, so
these windows also remain above installed RAM. A 32 GiB aperture on the XTX
does not turn its 24 GiB of physical VRAM into 32 GiB.

With the original 16T setting, all eight high host windows began at or above
16 TiB; post-PEX MI50 BARs began at 21 TiB and above. The relevant low host
window was only 88 MiB. Changing BAR/slot sizes within those host apertures
could not create an in-range 32 GiB window. Lowering the firmware base supplied
usable lower host apertures without widening the MI50's native DMA mask.

## Exact tested software stack

These identities were checked on the running stack after reboot. Installed
packages alone do not show which kernel or modules executed a test. Package
and build history is in the [host activation record][activation].

| Component | Tested identity |
|---|---|
| Running Linux kernel | `6.17.0-1000-generic` |
| Custom kernel package | `6.17.0-1000.42+llaminar2` |
| Ubuntu source base | `6.17.0-42.42`, tag `Ubuntu-hwe-6.17-6.17.0-42.42`, commit `7cb9b24dc95cd7faa58265648bd74bd17b03f55d` |
| Kernel build ID | `bc3a31fc96512be21f0f3ed43ee8727b0de6f1e1` |
| AMD driver / DKMS source | `7.1.9.31600000` / `7.1.9-2407772.24.04` |
| Loaded `amdgpu` srcversion | `BFEAE8D9287FE23B9907691` |
| AMD package | `1:7.1.9.31600000-2407772.24.04+local.dce1` |
| AMD firmware package | `1:31.60.0.0.31600000-2407772.24.04` |
| NVIDIA open driver / package | `595.99.02` / `595.99.02-0ubuntu0.24.04.1` (`nvidia-driver-595-open`, `nvidia-dkms-595-open`) |
| Loaded NVIDIA srcversion | `919309EF6954372EFADD77E` |
| PEX DKMS package / module | `pex-window/0.1.2` / `pex_window` |
| Loaded PEX srcversion | `7D4D55A110866DB75810F63` |
| Container ROCm SDK | **10.0.0** at `/opt/rocm-sdk` |
| Native allocation policy | `HSA_USERPTR_FOR_PAGED_MEM=0`, `HSA_USE_SVM=0`; `HSA_OVERRIDE_GFX_VERSION` unset |

The host ROCm/CUDA SDKs had been removed while host driver/runtime integration
was retained; native test compilation/runtime used the container SDK. NVIDIA
versions document the co-installed stack, not a demonstrated requirement for
AMD peer reads.

### Kernel and AMD changes

The custom kernel retains the Ubuntu generic configuration, including Rust,
BTF, and DWARF, apart from its version signature. There is **no DMA-mask
widening or raw-BAR substitution** in this stack. The retained patches below
document the actual changes; the X99 recipe's bridge-whitelist/direct-attach
edits should not be assumed to be part of this separate configuration.

| Change | Retained patch and purpose | What it establishes |
|---|---|---|
| Intel queued-invalidation recovery | [0001][patch-qi], upstream `fe89277c9ceb0d6af0aa665bcf24a41d8b1b79cd`; `drivers/iommu/intel/dmar.c` aborts every fetched in-use entry after timeout, including even-indexed wait descriptors | Repairs abort bookkeeping; does not prevent the original device fault or bound every retry loop |
| IOMMU scatterlist overflow checks | [0002][patch-sg], upstream `1c8f6308e557f32412e9d4678b3f029f47f6d9aa`; `drivers/iommu/dma-iommu.c` rejects overflowing lengths/padding | Addresses a separate large mapped-RAM failure, not a demonstrated peer-VRAM routing fix |
| DRM input-segment bound | [0003][patch-drm]; `drivers/gpu/drm/drm_prime.c` applies `max_segment = min_t(size_t, max_segment, UINT_MAX / 2)` before building the page-array scatterlist | Splits large contiguous runs while preserving total allocation size and stricter device limits |
| Rust header package closure | [0004][patch-rust]; `debian/rules.d/2-binary-arch.mk` passes `$(baserustpkg)` to `link-lib-rust` | Packaging correction; no DMA or inference arithmetic change |
| AMD legacy-DCE build correction | [0005][patch-dce]; `amd/dkms/dkms-config.sh` adds `export_macro_mk CONFIG_DRM_AMD_DC_DCE` | Preserves the legacy DCE configuration default in the external AMD build |

Patches 0001–0004 target the Ubuntu source identified above. Patch 0005 targets
the **separate AMD DKMS source** `7.1.9-2407772.24.04`. Matching the displayed
AMD upstream version is insufficient if the local DCE correction is absent.
The custom kernel and derived AMD package are local builds, not vendor-signed
release packages.

ABI 1000 retains the QI and packaging fixes from ABI 999 and adds the two
scatterlist changes. Source-function CPU checks covered 524,309 patched QI
cases and 16 DRM/scatterlist composition geometries. The source investigation
also reapplied all five committed patches without fuzz and checked the result
against the tested candidate sources. Those checks authenticate the changes;
they do not prove which physical scatterlist caused the original runtime
failure or qualify a new kernel build on another host.

## `pex_window`: resource layout, ReBAR, and load order

`pex-window/0.1.2` is an out-of-tree DKMS package, separate from AMD and
NVIDIA. It builds `pex_window` from `/usr/src/pex-window-0.1.2`, installs under
`updates/dkms`, and sets `AUTOINSTALL=yes`. Its implementation is in
`pex_window.c`, `plan.h`, and `transaction.h`. The original source required no
compatibility modification for the tested 6.17 kernels.

**Public source and installation instructions:**
[dbsanfte/pex-window-driver][pex-project]. [Release `v0.1.2`][pex-release]
corresponds to [source commit `f1fc1921fd1ef4759aacc0c18ab7173f21d4b11f`][pex-source].
The retained test sources `pex_window.c`, `plan.h`, and `transaction.h` are
byte-identical to those files in that public release commit.

The release provides the versioned DKMS source asset `pex-window-0.1.2.tar.xz`
and its SHA-256 checksum. Use matching headers/build files for the custom
`6.17.0-1000-generic` kernel; the release's prebuilt module archives target
other exact kernel ABIs. The repository documents building, installation,
initramfs integration, validation, and rollback.

The module plans the complete 64-bit prefetchable resource hierarchy of the
PEX880xx trees, updating root/downstream bridge windows and GPU BARs together
while preserving bus numbers and unrelated MMIO. ReBAR enlargement is subject
to the endpoint's actual capability and supported-size mask. An already large
enough BAR is retained, including an existing BAR larger than the request.

Installed modprobe configuration:

```text
options pex_window apply=1 bar_gib=32 slot_gib=64 empty_min_width=8
softdep amdgpu pre: pex_window
softdep nvidia pre: pex_window
softdep nouveau pre: pex_window
softdep snd_hda_intel pre: pex_window
```

| Parameter | Meaning in 0.1.2 |
|---|---|
| `apply=1` | Apply the verified transaction; default `apply=0` only plans |
| `bar_gib=32` | Request at least 32 GiB of VRAM BAR aperture, subject to endpoint capability |
| `slot_gib=64` | Reserve 64 GiB of MMIO address space per occupied or qualifying empty slot; no VRAM allocation |
| `empty_min_width=8` | Reserve empty ports whose maximum link width is at least x8; zero disables empty-port reservation |
| `roots` unset | Discover all eligible PEX trees |
| `fail_step=0` | No transaction fault injection |

Both inspected trees reserve five 64 GiB slots, including empty-slot capacity.
Sizes must be powers of two, and each slot must accommodate its BAR. The
module retains the existing ACPI/firmware **host aperture**: it has no base
override and cannot extend the root complex's physical decoder. It changes
neither DMA masks nor IOMMU domains, ATS/ACS policy, or P2P routing. This is why
the same module could allocate suitable BARs after the BIOS change but could
not repair the original high host apertures by itself.

The initramfs hook `/etc/initramfs-tools/hooks/pex-window` includes the module;
`/etc/initramfs-tools/scripts/init-top/00-pex-window` loads it before udev
coldplug and ordinary module loading. The soft dependencies cover later/manual
probes. The early script logs a load failure; it does not itself guarantee that
GPU drivers cannot start after a failure. The separate diagnostic AMD loader
and Docker readiness gate are described in the [recovery record][recovery].

Applying the layout requires detached endpoint drivers. The module serializes
PCI rescan/removal, disables memory decoding and bus mastering during the
transaction, updates Linux resources and hardware registers, and checks writes.
Failed application attempts restoration. **Unloading the module does not undo
a successful layout**: the resources belong to the PCI devices. Explicit
rollback requires retained transaction state, detached endpoints, and unchanged
resource ownership; it is not a live-workload switch.

## Failed comparisons and the successful read

| BIOS MMIO base | Effective DMA domain | ATS control | Native remote-read result |
|---|---|---|---|
| 16T | Translated `DMA-FQ` | Enabled | Owner `97` → reader `9a` faults; GPU reset fails, followed by VT-d/NIC disruption and a user-observed watchdog reboot |
| 16T | Translated `DMA-FQ` | `pci=noats`, endpoint ATS confirmed disabled | Same remote-read fault and failed reset; no VT-d/NIC cascade within the observed 313.881 seconds, but TTM cleanup stalls |
| 4T | `identity` via `iommu=pt` | No `pci=noats` override | All 20 directed AMD reads pass, including full payload verification, cleanup, and clean driver/process retirement |

No 4T/translated-DMA or 16T/identity-DMA remote-read comparison was performed.
ATS-off therefore did not repair the peer read; its observed benefit was
limited to avoiding the secondary host-wide cascade in one observation window.
The QI recovery patch was already present during the failed reads.

The native test used two 64 KiB device allocations. The owner initialized its
buffer and completed its stream; the reader enabled access and ran a kernel
that directly loaded the owner's pointer. The original read kernel used
64-by-256 launch geometry and verified **all 16,384 varying 32-bit words**.
On the successful boot, the unchanged read binary passed in every direction.
Separate local-memory controls also passed on the nearest MI50 pair. Pairwise
success does not imply simultaneous execution of all pairs.

The failing translated-DMA trace is important: the driver did **not** simply
put the high raw BAR address into the planned peer mapping. `dma_map_resource`
returned an in-range I/O virtual address (IOVA). The access still failed.

| Address observation | Failed ATS-off read | Successful identity-DMA read |
|---|---|---|
| Owner `97` BAR0 base | `0x153000000000` | `0x93000000000` |
| Owner-local VRAM offset | `0x7fec00000` | `0x7fec00000` |
| Physical peer resource | `0x1537fec00000` | `0x937fec00000` |
| Importer `9a` DMA address | `0xf2200000` | `0x937fec00000` |
| Planned peer mapping destination | `0xf2200000` | `0x937fec00000` |
| Target retirement observed in trace | No matching target invalidation/unmap before capture ended | Target invalidations followed by matching resource unmap |

The successful boot ID is `2ef86b85-de24-4e81-8796-d5a21af56178`. Its actual
successful read was correlated separately from the preceding mapping-only
process. Local/peer planned flags were `0x4e1` / `0x4e7` in both comparisons.
These VM events describe **planned updates**, not final hardware PTE readback
or observed PCIe transactions. An in-range IOVA alone did not prove a working
route, and the traces do not identify the original faulty TLP. The evidence
supports the combined mitigation without justifying a raw-BAR bypass,
DMA-mask widening, or blanket IOMMU disable.

## Llaminar captured peer-channel correction

The first captured `PeerDevice` round trip on MI50s `97` and `9a` failed its
first live byte (`replay=0 bytes=1 offset=1 first_bad_byte=1`) after the native
reads passed. That historical failure, preserved in the pinned investigation,
was subsequently resolved in Llaminar. This channel is separate from RCCL;
its native-collective option was false.

The corrected implementation uses explicit peer-visible ROCm storage via
`hipExtMallocWithFlags(..., hipDeviceMallocUncached)` and places the 64-byte
producer publication header in the same peer-visible allocation as the
payload. The consumer observes that publication through its retained device
binding. The tested path also retains non-temporal peer reads, with the
publication pointer and storage geometry included in capture identity. These
are Llaminar memory-visibility/publication changes; the replay did not require
another BIOS, kernel, or module change.

The original production-path registration now has passing follow-up receipts:

| Test | MI50 device order | Result |
|---|---|---|
| `V2_Integration_CapturedPeerPayload_ROCm2_OriginalInventory` | Original `97`, `9a` order | Pass; exit 0, clean driver observation |
| Same registration | Reversed `9a`, `97` order | Pass; exit 0, clean driver observation |

Both ran on boot `2ef86b85-de24-4e81-8796-d5a21af56178`, the same boot as the
successful native reads, with the recorded HSA allocation policy. The local
follow-up evidence is under
`/tmp/qfn-peer-channel-fix-r913/colocated-original-inventory/` and
`/tmp/qfn-peer-channel-fix-r913/colocated-reverse-inventory/`: each retains
`command.json`, `summary.json`, and `tests.xml`. These receipts belong to the
corrected r709 source/build r710; they are later evidence than the linked
research checkpoint, not a claim that its original source already contained
the repair.

This closes the specific captured MI50 peer-channel mismatch recorded above.
Collective transport, broader topology/model coverage, and performance have
their own validation scopes. Performance requires quiet matched measurements;
durable diagnostic logging times are not link bandwidth measurements. The X99
performance numbers in the main write-up are not measurements of this
configuration.

## Rechecking configuration and preserving recovery

On the **host**, record the running state together so observations from
different boots or pre-/post-PEX layouts cannot be mixed:

```sh
uname -r
cat /proc/sys/kernel/random/boot_id
cat /proc/cmdline
cat /sys/module/amdgpu/parameters/pcie_p2p
cat /sys/module/amdgpu/version /sys/module/amdgpu/srcversion
cat /sys/module/pex_window/version /sys/module/pex_window/srcversion

# These are this report's BDFs; identify your own devices first.
for bdf in 0000:49:00.0 0000:8c:00.0 0000:91:00.0 0000:97:00.0 0000:9a:00.0; do
    printf '\n%s\n' "$bdf"
    cat "/sys/bus/pci/devices/$bdf/resource"
    cat "/sys/bus/pci/devices/$bdf/iommu_group/type"
done
```

The first `resource` row gives BAR0's inclusive start/end and flags. Check
the **whole range**, not just its start, against the 44-bit limit. Confirm
loaded module identities as well as installed packages; `modinfo` alone
describes an on-disk module. Retain actual ATS/ACS observations and the kernel
journal covering initialization, access, cleanup, and process retirement.
Capability queries and successful mapping-only probes did not catch the
failing remote load in this investigation.

The successful identity-DMA trial initially used a consumed one-shot GRUB
entry. It was subsequently made the normal default by explicit operator
choice: `GRUB_DEFAULT=llaminar-p2p-trial-identity`. That policy change performed
no reboot and provided no additional runtime qualification. It retained the
diagnostic delayed AMD-load service and Docker readiness gate.

The visible 10-second menu retains kernel `6.17.0-999-generic`
(`6.17.0-999.42+llaminar1`) with P2P off, a drivers-disabled recovery entry,
and earlier `6.14.0-37-generic` / `6.14.0-36-generic` kernels. After promotion,
an ordinary reset selects the identity-DMA/P2P-on default again; P2P-off
recovery requires manual selection. GRUB rollback **does not undo the BIOS
MMIO change**. The [recovery record][recovery] documents boot selection,
persistent logging, the existing hardware watchdog, and diagnostic load order.

The linked investigation retains detailed trace correlations and historical
local evidence paths. Those paths are provenance, not downloadable artifacts.
The committed kernel/AMD patches and public PEX source/release links below
provide the source references for this configuration. The later peer-channel
repair and its passing local receipts are recorded separately above.

[research]: https://github.com/Llaminar/llaminar/blob/a5d3ee32afdf4a3c5de3a5c243191b2842871995/docs/v2/projects/2026-11/ROCM_PEX_P2P_MMIO_IOMMU_RESEARCH_20261107.md
[bios-record]: https://github.com/Llaminar/llaminar/blob/a5d3ee32afdf4a3c5de3a5c243191b2842871995/docs/v2/projects/2026-11/ROCM_PEX_P2P_MMIO_IOMMU_RESEARCH_20261107.md#complete-user-supplied-bios-record
[activation]: https://github.com/Llaminar/llaminar/blob/a5d3ee32afdf4a3c5de3a5c243191b2842871995/docs/v2/projects/2026-11/QFN_HOST_KERNEL_DRIVER_ACTIVATION_20261107.md
[recovery]: https://github.com/Llaminar/llaminar/blob/a5d3ee32afdf4a3c5de3a5c243191b2842871995/docs/v2/projects/2026-11/ROCM_PEX_P2P_DIAGNOSTIC_RECOVERY_20261107.md
[pex-project]: https://github.com/dbsanfte/pex-window-driver
[pex-release]: https://github.com/dbsanfte/pex-window-driver/releases/tag/v0.1.2
[pex-source]: https://github.com/dbsanfte/pex-window-driver/tree/f1fc1921fd1ef4759aacc0c18ab7173f21d4b11f
[patch-qi]: https://github.com/Llaminar/llaminar/blob/a5d3ee32afdf4a3c5de3a5c243191b2842871995/docs/v2/projects/2026-11/patches/rocm-pex-p2p-20261107/0001-intel-iommu-qi-recovery.patch
[patch-sg]: https://github.com/Llaminar/llaminar/blob/a5d3ee32afdf4a3c5de3a5c243191b2842871995/docs/v2/projects/2026-11/patches/rocm-pex-p2p-20261107/0002-iommu-sg-overflow.patch
[patch-drm]: https://github.com/Llaminar/llaminar/blob/a5d3ee32afdf4a3c5de3a5c243191b2842871995/docs/v2/projects/2026-11/patches/rocm-pex-p2p-20261107/0003-drm-prime-segment-bound.patch
[patch-rust]: https://github.com/Llaminar/llaminar/blob/a5d3ee32afdf4a3c5de3a5c243191b2842871995/docs/v2/projects/2026-11/patches/rocm-pex-p2p-20261107/0004-rust-header-package-link.patch
[patch-dce]: https://github.com/Llaminar/llaminar/blob/a5d3ee32afdf4a3c5de3a5c243191b2842871995/docs/v2/projects/2026-11/patches/rocm-pex-p2p-20261107/0005-amdgpu-legacy-dce.patch
