# Experimental Ventura GFX9 HwInfo fix

This branch contains an opt-in proof of concept for Raven-family APUs running
macOS Ventura 13.7.8 (22H730). It was developed and runtime-tested on a Ryzen
7 5800H system with Green Sardine integrated graphics.

## What it fixes

Ventura's `AMDRadeonX5000GLDriver` and `AMDRadeonX5000MTLDriver` initialise
their GFX9 hardware information from Vega discrete-GPU records. On the tested
APU this gives the user-space drivers incorrect family, topology, cache and
capability values and can result in Chromium/Electron rendering corruption,
stalls, SharedImage allocation failures and GFX channel resets.

The patch corrects both the OpenGL and Metal copies of the hardware record:

- family: Raven instead of AI/Vega;
- shader engines and shader arrays: 1/1;
- TCC blocks: 4;
- GS waves per VGT: 16;
- off-chip LDS buffers per shader engine: 128;
- parameter-cache lines: 1024;
- common flags: RBPlus, CMask, FMask, mobile, APU and valid;
- chip flags: Raven2, GFX9, ASTC and programmable near-Z.

The binary signatures are deliberately tied to Ventura 13.7.8 (22H730).
Other macOS builds are not expected to match and therefore remain unpatched
rather than receiving unverified offsets.

## Companion Lilu patch

Lilu normally disables UserPatcher on Big Sur and newer. The companion patch
in `Patches/Lilu-Ventura-DirectMemory.patch` adds a narrowly scoped,
opt-in DirectMemory path for this NootedRed experiment. It hooks Ventura's
`cs_validate_range` and `cs_validate_page` paths early enough to patch the
standalone AMD user-space driver pages.

After cloning with submodules, apply it before building:

```sh
git -C Lilu apply ../Patches/Lilu-Ventura-DirectMemory.patch
```

The GitHub Actions workflow applies the patch automatically.

## Enabling the experiment

Install the matching locally built `Lilu.kext` and `NootedRed.kext`, then add
both boot arguments:

```text
-NRedGLFull -lilunreduser
```

`-NRedGLTopo` enables the topology-only variant. `-NRedGLFull` enables the
complete tested record correction.

Keep a bootable EFI backup before enabling the experiment. To roll back,
restore the previous Lilu and NootedRed kexts and remove both boot arguments.

## Validation status

The full patch was confirmed at runtime on Ventura 22H730 in both the OpenGL
and Metal drivers. Chromium 150 stopped producing the repeated GFX resets and
rendering stalls seen with the unmodified Vega records. No new GPU restart
report was generated during the validation run.

This experiment only corrects GFX rendering hardware information. It does not
enable the VCN engine or VideoToolbox H.264/HEVC hardware decoding.
