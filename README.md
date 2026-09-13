# PCIe Peer-to-Peer between two AMD MI50s on an Intel X99 host: a saga in three gates

A case study in "the driver says P2P works, and the data is garbage." If you run an AMD
GPU (Vega 20 / gfx906) on an Intel X99 box and want two cards talking to each other over
PCIe instead of round-tripping through system RAM, read this before you burn a weekend.

---

## Why I even cared

I run a small local LLM box: two AMD MI50 cards (Vega 20, 32 GiB each) on an older Intel
X99 platform (Broadwell-E). That's 64 GiB of VRAM total — plenty for a big model — but only
if both cards can actually cooperate.

The way you normally split a model across two GPUs is *layer split*: card A holds the first
half of the layers, card B holds the second half, and every token hops through host memory
between them. It works, but it's slow — that host round-trip is the bottleneck on every
decode step.

What I **wanted** was *tensor split*: both cards work on the same layers together, and a
fast all-reduce stitches the halves together at every layer. Tensor split is the strategy
that actually wins — but it only wins if the two cards can move data directly between
themselves over PCIe, without dragging the CPU and system RAM into it. That direct
card-to-card link is **Peer-to-Peer (P2P)**, and it's what this whole story is about.

So: the goal was never "check a box that says P2P enabled." The goal was **fast, correct
tensor-split token generation.** P2P was just the road. I kept losing sight of that — which
is exactly how I wasted most of my time.

---

## The short version

If you take one thing from this: **on Intel X99, don't lift the GPU's DMA address mask to
meet its BARs — move the BARs down into the GPU's native address window. The "forced P2P"
path is fast but silently corrupt.** Crank it up, and P2P lights up green... and hands you
garbage.

Also: the scary "hardware read bug" I chased for hours was my own test harness. Both of
those sentences are the whole story in microcosm — *I kept fighting the machine instead of
listening to it.*

---

## What unfolded

AMD's driver only calls a device "peer-accessible" when several things are all true at the
same time. Sounds trivial. On my machine, every single one of them was wrong, one after
another — so what looked like one maddening mystery bug was really three stacked problems
wearing a trench coat.

### Gate 1: the kernel didn't even allow this path

The kernel has a whitelist of Intel host bridges it considers safe for P2P. My
Broadwell-EP bridge (`8086:6f00`) isn't on it — the list starts at Skylake-E and newer. From
a stock kernel, `hipDeviceCanAccessPeer` said **0**. I wasn't stuck at step three; I was
dead before I reached the start.

*Fix: build a kernel with the bridge whitelisted.*

### Gate 2: the BARs lived above the GPU's head

Whitelist added, rebuilt — still no P2P. Here's the subtle part. These MI50s natively use a
**44-bit DMA mask** (AMD only uses 48 bits on newer silicon). But this board's firmware put
the 32 GiB GPU BARs up around **56 TiB** — miles above bit 44. So the "address fit" check
failed. Distance check passed, mask check failed.

### Gate 3: the trap — "P2P enabled" but quietly corrupt

The obvious next move: force the mask up to 48 bits. Do that, and the HIP layer lights up
**P2P ENABLED**. Victory lap time? No. Now I had a *much* worse problem: the data was
**silently wrong**. GPU 0 → 1 writes landed as garbage/bit-flipped artifacts, and the other
direction only behaved when forced through host memory.

P2P was "on," and it was handing me corrupted numbers with a smile. That's the state that
cost me the most time, because it's the one that makes you doubt your cards.

---

## The dead ends (so you don't walk them)

- **Stock kernel:** `hipDeviceCanAccessPeer = 0`. Bridge not whitelisted.
- **Whitelist alone:** still no P2P. BARs above the 44-bit mask.
- **Whitelist + forced 48-bit mask:** "P2P enabled," silently corrupt. The X99 fabric has
  **no safe direct path at high MMIO**. This was the real poison pill.
- **Writing the PCI config registers directly** to move the MMIO window: **locked** on this
  board. The firmware call said success; the register never moved.
- **DSDT/ACPI override at boot:** landed in initramfs emergency mode, fighting the
  firmware's own tables. Abandoned.
- **RCCL for the all-reduce:** my llama build ships with NCCL compiled out, and the
  standalone 2-GPU RCCL all-reduce deadlocks here. Dead end.
- **Various speedup forks** (speculative decoding, community vLLM builds): none fit this
  model on this chip. Rolled back.

None of these moved the needle. The fix came from **stopping the fight**.

---

## What actually fixed it

### 1. Move the BARs *down*, don't lift the mask

The native 44-bit mask was **right**; I'd had the wrong goal the whole time. So instead of
pushing the mask up to meet the BARs, I pushed the **BARs down** into the 44-bit window.

Direct register writes are locked — but AMI's reference code exposes the MMIO controls as a
**writable UEFI policy variable**, so the firmware programs the window before the OS ever
sees it. On my MSI board that's the `IntelSetup` variable (two DWORDs: MMIO high base and
size). I dropped the window from ~56 TiB to a **1 TiB base / 1024 GiB window**, comfortably
under the 44-bit ceiling.

### 2. A kernel that gets out of its own way

The kernel I settled on combines three things:

1. whitelist entries for the `6f00`/`6f01` bridge IDs → clears Gate 1;
2. a gfx906-scoped rule that only keeps GPU↔GPU direct attach when the kernel's own P2PDMA
   distance check passes;
3. **stock `gmc_v9_0.c`** — the native 44-bit mask, no 48-bit hack. That last point is the
   load-bearing invariant: **don't touch the mask once the BARs are low.**

After boot, both cards report full 32 GiB BARs at `0x10000000000` and `0x11000000000` —
inside the mask — with KFD peer links present.

### 3. Use the fork's custom all-reduce, not RCCL

With NCCL compiled out, tensor split finally reduced through the fork's **custom
AllReduce** (broadcast + two-shot, peer-write, size-adaptive) — and it beats layer split.

---

## The "read bug" was me, not the hardware

Halfway through I was *convinced* there was directional CU peer-**read** corruption.
There wasn't, and I'm glad I kept drilling.

My throughput test was reusing **bitwise-complement** patterns (`0xa5a55a5a` /
`0x5a5aa5a5`) across directions without re-seeding the source buffer first. So it "verified"
correct data against the *wrong* expected pattern — and reported `16777216/16777216 bad`,
`got = ~want`. The machine was right; my test was lying.

Once I re-seed the source buffer before every read:

- compute-read + compute-write, **both directions**: 0 bad;
- DMA (`hipMemcpyPeer`): clean both ways;
- no more `0x3333`/garbage once on the low-MMIO kernel.

**P2P data is byte-clean in both directions. The hardware was innocent the entire time.**

---

## Why did I want P2P working? Because of this.

Here are the numbers that made the whole slog worthwhile — same workload, each config
measured alone, cold:

| Metric | Tensor split + custom AR | Layer split | Delta |
|---|---|---|---|
| Decode (token gen) | ~92–98 tok/s | ~75–78 tok/s | **+22–26%** |
| Prefill | ~1995 tok/s | ~1840 tok/s | +8–14% |
| End-to-end latency | — | — | **−13–20%** |

Correctness held across the full sweep (deterministic greedy math/reasoning, zero garbage),
and the tensor config wins decode at **every** context length — the widest margin, ~+25%,
at long context.

So the whole point of P2P was never the checkbox. It's that token generation went from
~75–78 to ~92–98 tokens per second, and prefill from ~1840 to ~1995 — because the two cards
finally talk directly instead of dragging the CPU and RAM through every step.

---

## Credit where it's due

A big part of the reason this didn't eat weeks of my life is **Assistmeister**, who handed me a build I could build upon to get to my fix. If you happen across this post with the same hardware, his work is a good place to start from — and mine is really a small delta on top of it.

---

## The takeaways — hopefully these save you a day

1. **`hipDeviceCanAccessPeer = 1` is not proof of a working data path.** Prove it with
   bidirectional, byte-correct transfers before you trust a single number from it.
2. **On Intel X99 with an AMD GPU: move the BARs down, never raise the mask.** The forced
   48-bit path is corrupt; the 44-bit-native + low-MMIO path is clean. This is the fix.
3. **This "one bug" was really three stacked gates** (whitelist → mask-fit → fabric
   correctness). There was never a single "enable P2P" switch — each gate needed its own,
   different fix.
4. **If a config-register write is locked, hunt for a writable UEFI policy variable.** The
   firmware often exposes the same control through a second, working door.
5. **Suspect your test before you suspect your hardware.** Re-seed buffers, never reuse
   complement patterns across directions. The infamous "read bug" was a harness bug, and
   nearly sent me on a mission to RMA perfectly good cards.

---

*Hardware-specific offsets, GUIDs, and kernel build recipes are intentionally omitted here
— the *approach* and the invariants above are the portable part. If you've got a dual-GPU
X99 build, I hope this gets you to clean P2P faster than it got me.*
