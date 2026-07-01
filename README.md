# Nothing Phone 2 (Pong) — GKI Kernel with KernelSU-Next + SuSFS

Builds a patched GKI boot image for the Nothing Phone 2 (codename **Pong**, SM8475) that integrates:

- **KernelSU-Next** — next-generation kernel-based root with SuSFS hooks
- **SuSFS** (susfs4ksu) — kernel-level file/path hiding to evade root detection

Based on the [DroidBasement GKI tutorial](https://droidbasement.com/db-blog/tutorial-kernelsu-next-with-susfs-integrated-in-to-a-gki-generic-kernel-image/) adapted for the 5.15 kernel used by the Nothing Phone 2.

---

## Before you start

### Check your exact kernel version

```bash
adb shell uname -r
```

| Output starts with | Kernel branch to pick |
|---|---|
| `5.10.x-android12-…` | **`android12-5.10`** ← Nothing Phone 2 (Pong) default |
| `5.15.x-android13-…` | `android13-5.15` |
| `5.15.x-android14-…` | `android14-5.15` |

> The Nothing Phone 2 (Pong / SM8475) ships with kernel **5.10** (`android12-5.10` GKI branch). This does not change with Nothing OS OTAs — only the minor version and security patch level change.

### Prerequisites

- Bootloader unlocked (you already have this — you're on KernelSU LKM)
- ADB + Fastboot installed on your PC
- USB debugging enabled
- A backup of your current `boot.img` (optional but recommended):

```bash
adb reboot bootloader
fastboot getvar current-slot          # note which slot (a or b)
fastboot --slot all getvar current-slot
# grab the current boot partition to back it up:
adb shell dd if=/dev/block/by-name/boot of=/sdcard/boot-backup.img
adb pull /sdcard/boot-backup.img .
```

---

## Step 1 — Trigger the GitHub Actions build

1. Go to the **Actions** tab of this repository
2. Select **"Build Nothing Phone 2 (Pong) — GKI Kernel + KernelSU-Next + SuSFS"**
3. Click **"Run workflow"** and choose:
   - **Kernel branch** — see table above (`android14-5.15` for most users)
   - **LTO mode** — `thin` (recommended; ~30 min faster than `full`)
4. The build takes approximately **1–3 hours** depending on GitHub runner load

When it finishes, download **`pong-boot-<branch>-ksu-susfs`** from the Artifacts section.

---

## Step 2 — Flash the boot image

> Your device stays in fastboot from the backup step, or reboot into it:
> ```bash
> adb reboot bootloader
> ```

```bash
# Flash to the current inactive slot (fastboot picks this automatically)
fastboot flash boot boot.img

# Reboot to check it boots correctly (does NOT permanently set the slot yet)
fastboot reboot
```

If the device boots normally, you're good. If it bootloops, hold **Power + Vol Down** to go back to fastboot and run:

```bash
fastboot flash boot boot-backup.img   # restore your backup
fastboot reboot
```

---

## Step 3 — Install KernelSU-Next Manager

Download the latest **KernelSU-Next** APK from the official releases:

https://github.com/rifsxd/KernelSU-Next/releases/latest

Install it via ADB or sideload:

```bash
adb install KernelSU-Next-*.apk
```

Open the app. If the kernel shows **"KernelSU-Next is installed"** with a valid kernel version, the boot image worked correctly.

---

## Step 4 — Install the SuSFS module

In **KernelSU-Next Manager**:

1. Tap the **Modules** tab
2. Tap the **+** button
3. Download and flash the latest `susfs4ksu-*.zip` module from:

   https://gitlab.com/simonpunk/susfs4ksu/-/releases

4. Reboot when prompted

After reboot, SuSFS is active. Root-detection apps (e.g. Play Integrity, banking apps) will see a clean environment when you configure hide lists.

---

## Step 5 — Verify everything works

```bash
# Should show KernelSU-Next version
adb shell su -c 'ksud version'

# Should show SuSFS is active
adb shell su -c 'cat /proc/susfs'
```

In KernelSU-Next Manager, the **SuperUser** tab should list apps that have requested root.

---

## Troubleshooting

| Problem | Fix |
|---|---|
| Bootloop after flashing | Restore backup from fastboot: `fastboot flash boot boot-backup.img` |
| KSU Manager shows "not installed" | Double-check the kernel branch — try `android13-5.15` if you used `android14-5.15` |
| Wi-Fi / Bluetooth broken | The workflow already removes `protected_exports_list`; if still broken, try `--lto=none` build |
| Patch fails in CI | Check the Actions log — the SuSFS patch may not apply cleanly on a new kernel minor version; open an issue |
| Play Integrity still failing | Ensure the SuSFS module is installed and the app's package is in the deny list in KSU Manager |

---

## What the build does

```
GKI kernel source (android.googlesource.com)
  └── simonpunk/susfs4ksu patches applied
        └── pershoot/KernelSU-Next (next-susfs) integrated
              └── Bazel build → boot.img
```

The resulting `boot.img` replaces only the **generic kernel** partition. The `vendor_boot.img` (which contains all Nothing Phone 2 hardware drivers) is never touched.

---

## Credits

- [pershoot](https://droidbasement.com/db-blog/tutorial-kernelsu-next-with-susfs-integrated-in-to-a-gki-generic-kernel-image/) — original GKI + KernelSU-Next + SuSFS tutorial
- [simonpunk/susfs4ksu](https://gitlab.com/simonpunk/susfs4ksu) — SuSFS kernel patches
- [pershoot/KernelSU-Next](https://github.com/pershoot/KernelSU-Next) — KernelSU-Next with SuSFS hooks
- [rifsxd/KernelSU-Next](https://github.com/rifsxd/KernelSU-Next) — upstream KernelSU-Next
