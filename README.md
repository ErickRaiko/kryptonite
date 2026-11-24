![Header](./Resources/Header.png)

# Kryptonite (macOS Tahoe / OCLP 3.0.0 Fork)

An updated fork of [mayankk2308/kryptonite](https://github.com/mayankk2308/kryptonite) that restores and extends **Thunderbolt 3 device, dock, and eGPU support on Thunderbolt 1 & 2 Macs** running modern macOS versions up to **macOS 26 (Tahoe)** alongside **OpenCore Legacy Patcher (OCLP) 3.0.0**.

Like the upstream project, Kryptonite is a [Lilu](https://github.com/acidanthera/Lilu) kernel extension that patches macOS **in-memory** through OpenCore—leaving system files untouched on disk and keeping your installation cleanly recoverable via a backup EFI.

---

## Tested Configuration

- **Host:** Mac Pro (Late 2013, `MacPro6,1` — Thunderbolt 2, Dual AMD FirePro)
- **macOS:** macOS 26.7.2 Tahoe (Build `25G332`, Darwin `25.6.0`)
- **Bootloader / Patcher:** OpenCore Legacy Patcher `3.0.0` (OpenCore `1.0.6`, `Lilu.kext` `1.7.2`, `AMD Legacy GCN` + `Tahoe Graphics` root patches)
- **Verified Thunderbolt 3 Device:** HP Thunderbolt 3 Dock (`8086:1578`, Alpine Ridge) via Apple Thunderbolt 3 (USB-C) to Thunderbolt 2 Adapter — previously blocked as `- Unsupported`, now fully enumerated and functional

---

## What Changed in This Fork

Building on upstream `mayankk2308/kryptonite` and Sequoia-era experiments, this fork adds and fixes:

1. **Native macOS 26 (Tahoe / Darwin 25) support**
   - Updated `PluginConfiguration` max kernel version from `KernelVersion::Sequoia` (24) to `KernelVersion::Tahoe` (25).
2. **`AppleGPUWrangler` Thunderbolt threshold patch for Sequoia & Tahoe**
   - Fixed `unblockLegacyThunderbolt` to match `getKernelVersion() >= KernelVersion::Sequoia` (verified against macOS 26.7.2 `AppleGPUWrangler`).
3. **`IOPCIFamily` `IOPCITunnelCompatible` bypass for Tahoe + future-proof fallback**
   - Added the verified macOS 26.7.2 jump offset (`0xb5`) inside `IOPCIBridge::matchNubWithPropertyTable` (`nop; jmp +0xb5`).
   - Added a symbol-scoped dynamic fallback inside `__ZN11IOPCIBridge25matchNubWithPropertyTableEP9IOServiceP12OSDictionaryPi` so minor Tahoe updates (e.g. `26.7.3+`) continue to patch cleanly even if the relative jump displacement shifts.
4. **Fixed Thunderbolt 3 enumeration hook (`- Unsupported` fix)**
   - Restored upstream Kryptonite behavior in `routeThunderboltEnumeration` and `thunderboltShouldSkipEnumeration`: `IOThunderboltSwitchType3::shouldSkipEnumeration()` is always routed (unless `kryskiptbroute=1`) and returns `0`, preventing macOS from marking TB3 docks/eGPUs as `- Unsupported` on TB1/TB2 hosts.
5. **Modern AMD eGPU driver coverage**
   - Added `AMDRadeonX5000` (Vega) and `AMDRadeonX6000` (Navi) to `IOPCITunnelled` patching alongside `AMDRadeonX4000` (Polaris).
6. **Stability & OCLP safety fixes**
   - Fixed out-of-bounds `pathNum` entries in `kern_targetkexts.hpp`.
   - Fixed stack-buffer lifetime issues in lookup patch definitions in `kern_patches.cpp` and moved duplicated file-scope statics into `NVRAMArgs`.
   - Hardened `Installer/hybrid_install.sh` so it **never overwrites an existing compiled `Lilu.kext`** on an OCLP EFI.

---

## Building from Source

Requires **Xcode** (tested with Xcode 26.5), [MacKernelSDK](https://github.com/acidanthera/MacKernelSDK), and [Lilu](https://github.com/acidanthera/Lilu) headers:

```bash
# 1. Fetch SDK dependencies (once)
git clone https://github.com/acidanthera/MacKernelSDK Kryptonite/MacKernelSDK
git clone https://github.com/acidanthera/Lilu Kryptonite/Lilu.kext
mkdir -p Kryptonite/Lilu.kext/Contents
ln -s ../Lilu Kryptonite/Lilu.kext/Contents/Resources

# 2. Build Kryptonite.kext
xcodebuild \
  -project Kryptonite/Kryptonite.xcodeproj \
  -scheme Kryptonite \
  -configuration Release \
  -derivedDataPath /tmp/kryptonite-build \
  build \
  CODE_SIGN_IDENTITY="-" \
  CODE_SIGNING_REQUIRED=NO
```

The compiled kext will be output at:
```
/tmp/kryptonite-build/Build/Products/Release/Kryptonite.kext
```

---

## Installing with OpenCore Legacy Patcher (OCLP 3.0.0)

> **Warning:** Always back up your working OCLP EFI to a **FAT32 GPT USB flash drive** before modifying your main EFI. If anything goes wrong, hold **Option (⌥)** at startup and boot from the USB backup EFI.

> **Important:** Do **not** replace OCLP's `EFI/OC/Kexts/Lilu.kext` with the `Kryptonite/Lilu.kext` folder from this repository (it contains Lilu source/headers only, not a compiled binary). Also do **not** run legacy Sequoia root-overlay scripts in `Installer/` on a Tahoe OCLP installation.

### 1. Copy `Kryptonite.kext` into your OCLP EFI

Mount your OCLP EFI partition and copy only the built `Kryptonite.kext` into `EFI/OC/Kexts/`:

```bash
ditto /tmp/kryptonite-build/Build/Products/Release/Kryptonite.kext /Volumes/EFI/EFI/OC/Kexts/Kryptonite.kext
```

### 2. Register `Kryptonite.kext` in `EFI/OC/config.plist`

Under `Kernel -> Add`, add an entry **after** `Lilu.kext`:

| Key | Value |
| --- | --- |
| `Arch` | `x86_64` |
| `BundlePath` | `Kryptonite.kext` |
| `Comment` | `Kryptonite eGPU / TB3 Enabler` |
| `Enabled` | `true` |
| `ExecutablePath` | `Contents/MacOS/Kryptonite` |
| `MaxKernel` | *(empty)* |
| `MinKernel` | `17.5.0` |
| `PlistPath` | `Contents/Info.plist` |

### 3. Add Boot Arguments

Under `NVRAM -> Add -> 7C436110-AB2A-4BBB-A880-FE41995C9F82 -> boot-args`, **keep all existing OCLP boot-args** and append the Kryptonite flags for your Mac.

**Example for `MacPro6,1` (Thunderbolt 2 + AMD):**
```text
krygpu=AMD krytbtv=2 kryskipagdp=1 kryskipmux=1
```

> **Critical for `MacPro6,1`:** Include **`kryskipagdp=1`**. On `MacPro6,1`, `AppleGraphicsDevicePolicy` must attach `AGDPClientControl` to the display-driving internal GPU (`GFX2`). Patching AGDP breaks `AGDPClientControl` and results in a black screen / offline displays.

---

## Boot Arguments Reference

| Boot-arg | Description |
| --- | --- |
| `krygpu=AMD` / `krygpu=NVDA` | Target GPU vendor for graphics/tunnel patches |
| `krytbtv=1` / `krytbtv=2` | Host Mac Thunderbolt version (`1` for TB1, `2` for TB2 — **required** on Sequoia/Tahoe) |
| `kryskipagdp=1` | Skip `AppleGraphicsDevicePolicy` patch (**required on `MacPro6,1`**) |
| `kryskipmux=1` | Skip `AppleMuxControl` NVRAM patch (recommended on desktops like `MacPro6,1`) |
| `kryskiptbroute=1` | Skip `IOThunderboltSwitchType3::shouldSkipEnumeration` hook |
| `kryskipagdc=1` | Skip `AppleGPUWrangler` legacy Thunderbolt threshold patch |
| `kryskipiogfx=1` | Skip `IOPCITunnelled` bypass patches |
| `kryskipiopci=1` | Skip `IOPCIFamily` `IOPCITunnelCompatible` bypass patch |
| `kryprobe=1` | Dry-run mode: load kext without applying patches |
| `-krydisable` | Completely disable Kryptonite at boot |
| `-krydbg` | Enable debug logging for Kryptonite |
| `-krybeta` | Force loading on newer/untested macOS kernel versions |

---

## Verification

After rebooting into your updated OCLP EFI:

1. Check **System Information → Thunderbolt/USB4** — your Thunderbolt 3 dock/eGPU should appear **without** `- Unsupported`.
2. Check kernel logs to verify `Kryptonite` loaded and Thunderbolt enumerated cleanly:
   ```bash
   /usr/bin/log show --last boot --style compact --predicate 'senderImagePath CONTAINS "IOThunderboltFamily" OR eventMessage CONTAINS[c] "Kryptonite"'
   ```
3. Verify your `config.plist` syntax anytime before rebooting:
   ```bash
   plutil -lint /Volumes/EFI/EFI/OC/config.plist
   ```

---

## Credits & License

- Original project by **Mayank Kumar** — [mayankk2308/kryptonite](https://github.com/mayankk2308/kryptonite)
- [Acidanthera](https://github.com/acidanthera) for **OpenCore**, **Lilu**, and **MacKernelSDK**
- [Dortania](https://github.com/dortania/OpenCore-Legacy-Patcher) for **OpenCore Legacy Patcher**

Licensed under the **GNU General Public License v3.0**. See [LICENSE.md](./LICENSE.md).
