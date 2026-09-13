# P2P Fix Recipe: from "enabled but corrupt" to byte-clean tensor split

A step by step recipe for getting real, byte-correct PCIe Peer-to-Peer between two AMD
MI50 (Vega 20 / gfx906) cards on an Intel X99 host. Read the main write-up first for the
context and the three gates. This page is the "how to actually do it" version.

It is written to be portable. Where a value depends on your specific board or BIOS, the
recipe tells you how to discover it on your own hardware instead of hard-coding one.
Nothing here is tied to a particular person's build.

## What you are trying to achieve

Two things, in order:

1. Make the kernel and driver *allow* a P2P path between the two cards.
2. Make the GPU BARs sit low enough that the peer's memory is addressable by the local
   GPU's 44-bit DMA mask, at an address range the X99 fabric handles correctly.

If you get only step 1, P2P reports enabled and corrupts data. That is the trap. The fix
is almost always moving the BARs down, not raising the DMA mask.

## Step 0. Identify the hardware

Run `lspci -vv` and find both GPUs and the host bridge they hang off. Note the bridge's
vendor:device IDs. On Broadwell-E/X99 the host bridge devices are `8086:6f00` and
`8086:6f01` (those are generic Intel PCI IDs for that platform, not machine specific).
Your own output may differ, so use what `lspci` reports.

## Step 1. Whitelist the host bridge in the kernel

The kernel only allows P2P over host bridges it trusts. See
`drivers/pci/p2pdma.c`, the array `pci_p2pdma_whitelist[]`, whose entries look like:

```c
struct pci_p2pdma_whitelist_entry {
	unsigned short vendor;
	unsigned short device;
	u8 flags;
};
```

Add one entry per host bridge you identified in step 0, for example:

```c
{ PCI_VENDOR_ID_INTEL, 0x6f00, 0 },
{ PCI_VENDOR_ID_INTEL, 0x6f01, 0 },
```

Use the actual vendor/device you found in step 0, not necessarily these. Rebuild and
install that kernel. Afterwards `hipDeviceCanAccessPeer` should no longer return 0 for
these cards (you can confirm with `dmesg | grep -i p2p`).

## Step 2. Check where the BARs actually are

With the whitelist in place, P2P still will not work if the BAR addresses are too high.
Look at the GPU BARs:

```
lspci -v -s <gpu-bus:device.function>
```

You want the large (32 GiB) prefetchable BAR entries. Read their addresses. gfx906 uses a
44-bit DMA mask, so the peer BAR address must be below 2^44 = 16 TiB. If your BARs live
above that (for example near 56 TiB), P2P will be reported as not addressable, or if you
force the mask up, silently corrupt. Move on to step 3.

## Step 3. Move the BARs down into the 44-bit window

Do **not** raise the GPU DMA mask as a shortcut. The reliable approach is to make the
firmware place the MMIO/BAR window low.

On these boards the MMIO window is normally controlled by a firmware policy variable,
because direct PCI config register writes are locked. The exact variable name, its GUID,
and the bit layout are specific to your board's BIOS, so you must discover them rather
than copy a value.

How to find them on your board:

- Boot into the UEFI shell (or use a tool that can read firmware variables) and list the
  variables with something like `dmpstore` or `dmpstore -all`. Look for an AMI/Intel
  `Setup` style variable that carries MMIO or memory-window policy.
- In the BIOS setup UI, find the setting that controls the 64-bit MMIO high base size or
  the PCI/memory window for the GPUs. The variable in firmware almost always mirrors what
  that menu exposes.
- Read the current values, change them to a low MMIO high base, and confirm the change
  actually landed in the register (if it does not, you found a locked/lost path and need
  the variable route).

Target: place the 64-bit prefetchable window so both 32 GiB GPU BARs sit below 16 TiB,
with room for the full VRAM. A low base (around 1 TiB) with a window large enough to
cover both cards works, but adjust to your own VRAM and other devices. Reconfigure the
BIOS, then recheck `lspci` until the BAR addresses are below 16 TiB.

## Step 4. Keep the native 44-bit DMA mask

Do not patch the mask. On gfx906 the driver uses a 44-bit mask, and AMD only raises to
48 bits on newer silicon, so leave the mask file (`drivers/gpu/drm/amd/amdgpu/gmc_v9_0.c`)
stock. Once the BARs are low, the native mask is correct and the forced-mask path is
the one that corrupts.

## Step 5. Restrict direct GPU attach to the valid path

To get the correct data path (not the corrupt one), make GPU to GPU direct attach valid
only when the kernel's own P2P distance check passes. Add a gfx906 scoped rule (in the
AMD GPU driver, around where DMA-buf peer attachments are accepted) that gates the direct
attach on the result of the kernel P2PDMA distance check. If the distance check fails, fall
back to the host-staged path. This is the guard that turns "enabled but corrupt" into
"enabled and clean."

The exact hook point differs by kernel version, so locate the function that decides whether
a peer DMA-buf attachment is direct, and add the distance-check condition there.

## Step 6. Verify the data path is actually clean

Before trusting any throughput number, prove P2P is byte-correct in both directions:

1. `hipDeviceCanAccessPeer` returns 1 in both directions.
2. BAR addresses are below 16 TiB (step 2).
3. A bidirectional byte-exact transfer passes.

For the transfer, copy from GPU 0 to GPU 1 and from GPU 1 to GPU 0, compare every byte,
and **re-seed the source buffer before each direction**. Reusing a buffer with
bitwise-complement patterns and no re-seed will falsely fail on the second direction.
When both directions are clean, P2P is real.

## Step 7. Use the P2P in tensor split

For two GPUs to share one model with tensor parallelism, you need an all-reduce that uses
the peer path. The key is that it must be lossless and use peer writes. A lightweight
in-tree all-reduce (broadcast plus a two-shot reduce, peer-write, size adaptive to the
tensor being reduced) works well and avoids a heavyweight dependency like RCCL. Verify
greedy inference output is deterministic across runs, so any corruption shows up as a
changed token rather than a silent wrong number.

## Checklist summary

- [ ] Host bridge whitelisted in `pci_p2pdma_whitelist[]`
- [ ] `hipDeviceCanAccessPeer` = 1 both directions
- [ ] Both GPU BARs below 16 TiB in `lspci`
- [ ] Native 44-bit mask left untouched
- [ ] Direct GPU attach gated on the P2P distance check
- [ ] Bidirectional byte-exact transfer passes (re-seed between directions)
- [ ] Tensor-split inference output is deterministic
