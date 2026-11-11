# Gigabyte X99-UD4-CF (BIOS F22): MMIO high base and size variables

A concrete, verified example for Step 3 of [P2P-FIX-RECIPE.md](../P2P-FIX-RECIPE.md). The
recipe asks you to discover the firmware policy variable that controls the MMIO window on
your own board. This page shows one board's answer end to end, including how it was found,
so you can repeat the same method and check your work.

Everything here is specific to this board and BIOS revision. Do not copy the values to a
different board without repeating the discovery step. The method is portable; the numbers
are not.

| item | value |
|---|---|
| board | Gigabyte GA-X99-UD4-CF |
| BIOS | F22 (2016-06-13), AMI Aptio V |
| CPU | Xeon E5-2687W v4 (Broadwell-E) |
| host bridge | `8086:6f00` at `00:00.0` (plus `8086:6f01`) |
| GPUs | 2x MI50 / Vega 20 (gfx906), 32 GiB VRAM, native 44-bit DMA mask |

## What changed

The firmware has one visible prerequisite and two hidden MMIO window controls.

| purpose | variable | data offset | width | value | meaning |
|---|---|---|---|---|---|
| Above 4G decoding | `Setup` | `0x489` | u8 | `1` | Enabled |
| MMIO high base | `IntelSetup` | `0x10D6` | u32 | `7` | 1 TiB |
| MMIO high size | `IntelSetup` | `0x10DA` | u32 | `3` | 1024 GiB |

The two MMIO values are enum selectors, not raw addresses:

```
MMIOHBase      56T=0 40T=1 24T=2 16T=3 12T=4 4T=5 2T=6 1T=7
MMIO High Size 256G=0 128G=1 512G=2 1024G=3
```

Both variables use the same GUID, but they are distinct UEFI variables:

| variable | GUID | size |
|---|---|---|
| `Setup` | `EC87D643-EBA4-4BB5-A1E5-3F3E36B20DA9` | `0x4DF` |
| `IntelSetup` | `EC87D643-EBA4-4BB5-A1E5-3F3E36B20DA9` | `0x17CD` |

Offsets are relative to the start of the variable data. The efivarfs file is the 4-byte EFI
attributes followed by that data, so data offset `N` is file offset `4 + N`.

## Before and after

Defaults on this board are `MMIOHBase=0` (56 TiB) and `MMIOHighSize=0` (256 GiB). With
Above 4G decoding enabled and those defaults, the firmware placed the GPUs here:

```
host bridge window [mem 0x380000000000-0x383fffffffff]      # 56 TiB, 256 GiB
pci 04:00.0: BAR 0 [mem 0x383fe0000000-0x383fefffffff]     # 256 MiB
pci 07:00.0: BAR 0 [mem 0x383fc0000000-0x383fcfffffff]     # 256 MiB
```

That is above the 44-bit mask, and the BARs are still the default 256 MiB, not the full
32 GiB VRAM. After writing `MMIOHBase=1T` and `MMIOHighSize=1024G` and rebooting:

```
host bridge window [mem 0x10000000000-0x13fffffffff]       # 1 TiB
amdgpu 04:00.0: BAR 0 [mem 0x10000000000-0x107ffffffff]    # 32 GiB
amdgpu 07:00.0: BAR 0 [mem 0x11000000000-0x117ffffffff]    # 32 GiB
```

Both BARs are below 2^44 (16 TiB). The kernel only resizes the BAR once the firmware
advertises a high window; see the kernel command line note below.

## How these were found

1. Extract and decompress the firmware image, for example with `UEFIExtract image.bin all`.
2. Search the decompressed tree for the setup strings, for example `MMIOH`, `MMIO High`,
   `MMIO Low`, `MMCFG`. On this image they appear only in the `Platform` DXE driver
   (`ABBCE13D-E25A-4D9F-A1F9-2F7710786892`).
3. Decode that driver's HII/IFR with
   [IFRExtractor-RS](https://github.com/LongSoft/IFRExtractor-RS):

   ```
   ifrextractor "<...>/Platform/.../unc_data.bin" list
   ifrextractor "<...>/Platform/.../unc_data.bin" single 0 0
   grep -nE "MMIOH|MMIO High" <output>.txt
   ```

4. The form is `Common RefCode Configuration` (FormId `0x17`). It is not referenced by any
   other form, so it is hidden from the setup menus, but its questions still bind to the
   varstore. On this image:

   - `MMIOHBase` QuestionId `0x190`, VarStoreId `1`, VarOffset `0x10D6`
   - `MMIO High Size` QuestionId `0x191`, VarStoreId `1`, VarOffset `0x10DA`

5. The same IFR defines VarStoreId `1` as `IntelSetup`, GUID
   `EC87D643-EBA4-4BB5-A1E5-3F3E36B20DA9`, size `0x17CD`. The live efivar
   `IntelSetup-ec87d643-...` is `0x17D1` bytes, which is `0x17CD` plus the 4-byte
   attributes, confirming the mapping.

The raw IFR opcodes confirm the binding independently of the text parser. The
`EFI_IFR_VARSTORE` opcode carries the GUID, id, and size; each `EFI_IFR_ONE_OF` opcode
carries QuestionId, VarStoreId, and VarOffset. The values in the option lists are what the
variable stores.

## Writing it

On this board the visible `Above 4G decoding` can be set from the setup UI. The two MMIO
window values are hidden, so they have to be written to the variable directly. From Linux
with efivarfs:

- The efivarfs entry is immutable by default, so clear that first (`chattr -i`).
- efivarfs accepts only a whole-variable write at offset 0, so read the whole file, patch
  the bytes, and write it back unchanged in length.
- Back up the variable first, and read it back afterwards to confirm the write landed.
- Restore the immutable flag when done.

```python
import struct, shutil, subprocess
V = "/sys/firmware/efi/efivars/IntelSetup-ec87d643-eba4-4bb5-a1e5-3f3e36b20da9"
ATTR, BASE_OFF, SIZE_OFF = 4, 0x10D6, 0x10DA
d = bytearray(open(V, "rb").read())
shutil.copyfile(V, "/tmp/IntelSetup.bak")
struct.pack_into("<I", d, ATTR + BASE_OFF, 7)   # 1 TiB
struct.pack_into("<I", d, ATTR + SIZE_OFF, 3)   # 1024 GiB
subprocess.run(["chattr", "-i", V])
open(V, "wb").write(d)
subprocess.run(["chattr", "+i", V])
```

If the write is rejected, the firmware NVRAM whitelist is blocking it; use a UEFI shell
`setup_var` against the `IntelSetup` varstore instead. An earlier attempt on this machine
that used another board's offsets silently did nothing, which is why the read-back matters.

## Kernel command line

Remove `pci=nocrs`. With `pci=nocrs` the kernel ignores the firmware's host bridge windows
and uses a 0-based span, so the AMD driver's `amdgpu_device_resize_fb_bar()` check for a
root-bus memory window starting above 4 GiB fails and the BAR stays at 256 MiB. Once the
firmware advertises the high window and the kernel uses it, the driver resizes the BAR to
the full 32 GiB at boot. `pci=realloc` is optional in this setup.

## Notes and caveats

- Above 4G decoding is a separate prerequisite. With it disabled this board reserves no
  MMIO region above RAM at all, so the 32 GiB BARs cannot be placed no matter what the
  MMIO high base and size are set to.
- The MMIO window values are enum selectors. Writing a raw address into those DWORDs does
  nothing useful.
- These offsets are for BIOS F22. Other revisions may move them; repeat the discovery step.
- On this board CSM was left enabled and the MMIO window and 32 GiB BARs still worked. The
  ReBarUEFI documentation recommends disabling CSM, so treat that separately.
