<img width="3000" height="4000" alt="20260827_204259" src="https://github.com/user-attachments/assets/ebc7af71-f473-4e97-bb99-d21d3864ebc2" />

# PCIe Peer-to-Peer between two AMD MI50s on an Intel X99 host

A write-up about getting two AMD MI50 (Vega 20 / gfx906, 32 GiB each) cards to talk to
each other over PCIe on an older Intel X99 (Broadwell-E) platform. The driver said peer
access was possible, but the data was garbage. Here is what was wrong and what fixed it.

## Why I wanted P2P

The setup is a small LLM box with two MI50s, 64 GiB VRAM total. To use both cards for one
model you can either:

- **Layer split**: card A gets the first half of the layers, card B the second half. Data
  moves through host memory between the two halves on every step. Works, but slow.
- **Tensor split**: both cards work on the same layers and an all-reduce joins the halves
  at each layer. This is faster, but only if the two cards can move data directly between
  each other over PCIe (Peer-to-Peer, P2P), without going through the CPU and system RAM.

So the goal was fast, correct tensor-split generation. P2P is the part that makes that
possible.

## How the driver decides a GPU is peer-accessible

AMD's driver only reports a device as peer-accessible when several conditions are all true:

- the kernel allows P2P on this host bridge
- resizable BARs are present (full visible VRAM)
- the p2p distance check passes (allowed on the same host bridge)
- the peer's BAR fits the GPU's DMA mask

On this machine each one was wrong in turn, which made it look like one bug when it was
really three separate problems.

### Gate 1: the host bridge was not whitelisted

The kernel keeps a whitelist of Intel host bridges it considers safe for P2P. The
Broadwell-EP bridge (`8086:6f00`) is not on it; the list covers Skylake-E and newer. So from
a stock kernel, `hipDeviceCanAccessPeer` returned 0.

Fix: build a kernel with that bridge whitelisted.

### Gate 2: the BARs were above the GPU's DMA mask

After the whitelist fix, still no P2P. These MI50s use a native 44-bit DMA mask (AMD only
uses 48 bits on newer silicon, GC 9.4.2 and up). This board's firmware placed the 32 GiB GPU
BARs at around 56 TiB, which is above bit 44. So the address-fit check failed even though
the distance check passed.

### Gate 3: forcing a 48-bit mask made P2P "work" but corrupt

The next obvious step was to force the mask up to 48 bits. That made the HIP layer report
P2P enabled, but with a worse problem: the data was silently wrong. Writes from GPU 0 to
GPU 1 landed as garbage or bit-flipped values, and the other direction only worked when the
data went through host memory. P2P was enabled and handing back corrupt data.

## Things that did not work

- Stock kernel with no patches: no P2P, bridge not whitelisted.
- Whitelist patch alone: still no P2P, BARs above the 44-bit mask.
- Whitelist plus forced 48-bit mask: P2P "enabled" but corrupt. The X99 fabric has no safe
  direct P2P path at high MMIO. This was the main dead end.
- Writing PCI config registers directly to move the MMIO window: locked on this board. The
  firmware call returned success but the register never changed.
- A DSDT/ACPI override at boot: ended in initramfs emergency mode and conflicted with the
  firmware's own tables, so it was dropped.
- RCCL for the all-reduce: the llama build in use has NCCL compiled out, and a standalone
  2-GPU RCCL all-reduce deadlocks here.
- Various speedup options (speculative decoding, a community vLLM fork): did not fit this
  model on this chip, rolled back.

None of these helped.

## What fixed it

### 1. Move the BARs down instead of raising the mask

The native 44-bit mask was correct. The mistake was trying to match the BARs to a higher
mask. The right move was to place the BARs inside the 44-bit window.

Direct register writes are locked, but AMI's reference code exposes the MMIO controls as a
writable UEFI policy variable, so the firmware programs the window before the OS reads it.
On this MSI board that is the `IntelSetup` variable (two DWORDs, MMIO high base and size).
The window was changed from around 56 TiB to a 1 TiB base with a 1024 GiB window, below the
44-bit ceiling.

### 2. The kernel that worked

This kernel does three things:

1. Whitelist entries for the `6f00` / `6f01` bridge IDs, which fixes Gate 1.
2. A gfx906-scoped rule that keeps GPU-to-GPU DMA-buf attachments direct only when the
   kernel's own P2PDMA distance check passes.
3. Stock `gmc_v9_0.c`, the native 44-bit mask with no 48-bit change. This is the important
   part: leave the mask alone once the BARs are low.

After boot both cards report full 32 GiB BARs at `0x10000000000` and `0x11000000000`,
inside the mask, with KFD peer links present.

### 3. Use the in-tree custom all-reduce, not RCCL

With NCCL compiled out, tensor split reduced through the fork's custom AllReduce (broadcast
plus two-shot, peer-write, size-adaptive). It works and is faster than layer split.

## The "read bug" was in the test, not the hardware

At one point it looked like there was directional peer-read corruption. There was not. The
throughput test reused bitwise-complement patterns (`0xa5a55a5a` / `0x5a5aa5a5`) without
re-seeding the source buffer between directions, so it checked the data against the wrong
expected pattern and reported all reads as bad with `got = ~want`.

After re-seeding the source buffer before every read:

- compute-read and compute-write, both directions: 0 bad
- DMA (`hipMemcpyPeer`): clean both ways
- no more `0x3333` / garbage values on the low-MMIO kernel

P2P data is byte-clean in both directions. The hardware was fine.

## Why P2P was worth setting up

Same workload, each config measured alone, cold:

| Metric | Tensor split + custom AR | Layer split | Delta |
|---|---|---|---|
| Decode (token gen) | ~92 to 98 tok/s | ~75 to 78 tok/s | +22 to 26% |
| Prefill | ~1995 tok/s | ~1840 tok/s | +8 to 14% |
| End-to-end latency | n/a | n/a | -13 to 20% |

Correctness held across the sweep (deterministic greedy math and reasoning, no garbage).
The tensor config wins decode at every context length, with the widest gap around +25% at
long context.

So the point of setting up P2P was that token generation went from about 75 to 78 tok/s up
to about 92 to 98 tok/s, and prefill from about 1840 up to about 1995, because the cards
talk directly instead of through the CPU and RAM on every step.

## How to check if P2P actually works

P2P can report "enabled" and still hand back garbage, so verify it in three steps, in order. If any step fails, the later ones don't matter.

### 1. Does the driver allow the peer path?

The first gate is `hipDeviceCanAccessPeer`. This small check prints whether each card can access the other:

```cpp
#include <hip/hip_runtime.h>
#include <cstdio>

int main() {
    int canAB = 0, canBA = 0;
    hipDeviceCanAccessPeer(&canAB, 0, 1);
    hipDeviceCanAccessPeer(&canBA, 1, 0);
    printf("canAccessPeer 0->1 = %d\n", canAB);
    printf("canAccessPeer 1->0 = %d\n", canBA);
    return (canAB && canBA) ? 0 : 1;
}
```

Build with `hipcc` and run it. A `0` means the kernel is not allowing the path, which points at the host bridge not being in the P2P whitelist (or the distance check failing). You can also confirm with `dmesg | grep -i p2p`.

### 2. Do the BARs sit inside the 44-bit mask?

The peer's VRAM has to be addressable by the local GPU. gfx906 uses a 44-bit DMA mask, so the peer's BAR address must be below 2^44 (16 TiB).

Check where the GPU BARs actually are:

```
lspci -v -s <bus:device.function>
```

Look for the 32 GiB entries, for example `Memory at <address> (64-bit, prefetchable) [size=32G]`. The address must be below 16 TiB. On my board the firmware put them around 56 TiB, which is why the mask check failed even after the whitelist fix.

### 3. Is the data actually correct, both ways?

The only real proof is a bidirectional, byte-exact transfer. Copy from 0 to 1 and from 1 to 0, compare the bytes, and re-seed the source buffer before each direction. Reusing the same buffer with bitwise-complement patterns without re-seeding will produce false failures (that was my "read bug").

A minimal HIP test that does direct peer copies in both directions, plus a re-seed before each run, will tell you if the fabric is clean. If both directions come back with 0 mismatched bytes, P2P is real and the speed numbers are trustworthy.

## Credit

**Assistmeister** provided a build that this fix builds on. If you are working with the
same hardware, that is a good place to start.

## Takeaways

1. `hipDeviceCanAccessPeer = 1` does not mean the data path works. Verify it with
   bidirectional byte-correct transfers before trusting it.
2. On Intel X99 with an AMD GPU: move the BARs down, do not raise the mask. The forced
   48-bit path is corrupt; the native 44-bit mask with low-MMIO BARs is clean.
3. This was three separate gates (whitelist, mask fit, fabric correctness). Each needed a
   different fix; there is no single "enable P2P" switch.
4. If a config register write is locked, look for a writable UEFI policy variable. The
   firmware may expose the same control another way.
5. Check the test before blaming the hardware. Re-seed buffers and do not reuse complement
   patterns across directions.

Hardware-specific offsets, GUIDs, and build recipes are left out here. The approach and the
rules above are the reusable part.
