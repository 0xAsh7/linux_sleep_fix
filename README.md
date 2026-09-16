# ASUS TUF Gaming F15 (FX507ZV4) — Linux s2idle Wake Fix via DSDT Override

## Overview

**Problem:**  
On CachyOS / Arch Linux with kernel 7.x, `systemctl suspend` enters s2idle correctly, but the laptop cannot be woken by any built-in input: keyboard, touchpad, power button, or lid. The only recovery is a 15+ second power-button hard shutdown.

**Root cause:**  
The ASUS firmware DSDT is missing a `_PRW` (Power Resources for Wake) object on the active `LID0` device, and the touchpad’s `GpioInt` is marked `Exclusive` instead of `ExclusiveAndWake`. Because of this, Linux never arms GPE 0x02 — the Embedded Controller interrupt line shared by the lid, power button, and internal keyboard — during s2idle.

**Solution:**  
Extract, patch, recompile, and install a DSDT override via the `acpi_override` mkinitcpio hook. Bump the OEM Revision so the kernel actually applies the override. Once loaded, GPE 0x02 is armed and all internal wake sources work.

**Confirmed working on:**  
ASUS TUF Gaming F15 FX507ZV4 / FX507ZV, BIOS 330, CachyOS, kernel 7.2.5, Hyprland/Wayland.

---

## 1. Environment

| Component | Detail |
|-----------|--------|
| Laptop | ASUS TUF Gaming F15 (FX507ZV4 / FX507ZV) |
| BIOS | FX507ZV4.330 |
| CPU | 12th Gen Intel Core i7-12700H |
| iGPU | Intel Iris Xe (`/dev/dri/card1`) |
| dGPU | NVIDIA RTX 4060 Laptop (`/dev/dri/card0`) |
| NVIDIA Driver | 615.71.09 (open kernel modules) |
| Kernel | Linux 7.2.5-1-cachyos |
| Distro / DE | CachyOS x86_64, Hyprland 0.56.2 (Wayland) |
| Bootloader | Limine |
| GPU Switcher | supergfxctl (Hybrid) |

---

## 2. Diagnostic Discovery

The hardware does **not** crash during suspend. This was proven with:

```bash
sudo rtcwake -m mem -s 15
```

The laptop slept and woke automatically after 15 seconds, restoring display, Hyprland, and all inputs. This confirms the kernel, Intel DRM, NVIDIA power transitions, and Wayland compositor all handle sleep/resume correctly.

**Conclusion:**  
The issue is strictly that built-in hardware interrupts (keyboard, power button, lid) fail to send an ACPI wake event / interrupt to the CPU while in Modern Standby (s2idle). The firmware is not arming the EC GPE.

Additional checks:
- `/proc/acpi/wakeup` showed `PNP0C0D:01` (lid) and `PNP0C0C:00` (power button) as `enabled`, but no wake events occurred.
- `gpe02` was enabled but not generating wake events.
- Forcing S3 (`mem_sleep_default=deep`) caused an immediate reboot (Alder Lake ASUS boards lack true S3 power rails).
- NVIDIA power management was verified and ruled out.
- Compositor and input class settings were verified and ruled out.

The final conclusion: the DSDT must be patched to add `_PRW` to the correct `LID0` and fix the touchpad `GpioInt` flag.

---

## 3. Prerequisites

- Secure Boot **disabled** (`mokutil --sb-state` → `SecureBoot disabled`).
- Kernel lockdown **not active** (`cat /sys/kernel/security/lockdown` → `[none]`).
- Kernel built with `CONFIG_ACPI_TABLE_UPGRADE=y`:
  ```bash
  zgrep CONFIG_ACPI_TABLE_UPGRADE /proc/config.gz
  ```
- Packages: `acpica`, `cpio`, `mkinitcpio` (usually already installed).

---

## 4. Step-by-Step Fix

### 4.1 Extract and decompile the DSDT

```bash
sudo acpidump -b
iasl -d dsdt.dat
```

This produces `dsdt.dsl`.

### 4.2 Identify the correct `LID0`

```bash
grep -n "PNP0C0D" dsdt.dsl
```

You will see two occurrences. Example from this laptop:

- Line 8547: `LID0` under `H_EC` — has `_STA` returning `Zero` (not present). **Do not touch.**
- Line 91243: `LID0` under `\_SB.PC00.LPCB` — this is the active device the kernel binds to. **Patch this one.**

You can confirm which is active by checking the kernel’s ACPI path:

```bash
ls /sys/bus/acpi/devices/ | grep PNP0C0D
cat /sys/bus/acpi/devices/PNP0C0D:*/path
```

Expected output:

```
PNP0C0D:00 -> .../PNP0C09:00/PNP0C0D:00
PNP0C0D:01 -> .../PNP0C0D:01
\_SB_.PC00.LPCB.H_EC.LID0
\_SB_.PC00.LPCB.LID0
```

The second path (`\_SB_.PC00.LPCB.LID0`) is the one to patch.

### 4.3 Patch the active `LID0`

Original block (before):

```asl
Device (LID0)
{
    Name (_HID, EisaId ("PNP0C0D"))
    Method (_LID, 0, NotSerialized)
    {
        Return (One)
    }
}
```

Patched block (after):

```asl
Device (LID0)
{
    Name (_HID, EisaId ("PNP0C0D"))
    Name (_PRW, Package (0x02) { 0x02, Zero })   // <-- ADD THIS
    Method (_LID, 0, NotSerialized)
    {
        If (ECOK)
        {
            If (LIDC)
            {
                ^^^GFX0.CLID = 0x80000000
                Return (Zero)
            }
        }

        ^^^GFX0.CLID = 0x80000003
        Return (One)
    }
}
```

**Notes:**
- `_PRW` points at GPE `0x02`, which is the EC interrupt line shared by lid, power button, and keyboard.
- `_LID` now reads real hardware via `ECOK` and `LIDC` instead of always returning `One`.
- Do **not** add `_STA` returning `Zero`. If a `_STA` method exists and returns `Zero`, remove it or change it to `Return (0x0F)`.
- Verify `ECOK` and `LIDC` are defined in your DSDT:
  ```bash
  grep -n "ECOK" dsdt.dsl | head
  grep -n "LIDC" dsdt.dsl | head
  ```

### 4.4 Patch the touchpad `GpioInt`

Find the touchpad device (usually `TPD0` under `I2C0`). Locate the `GpioInt` line inside its `_CRS` method.

Before:

```asl
GpioInt (Level, ActiveLow, Exclusive, PullDefault, 0x0000,
    "\\_SB.GPI0", 0x00, ResourceConsumer, ,
    )
    {   // Pin list
        0x0000
    }
```

After:

```asl
GpioInt (Level, ActiveLow, ExclusiveAndWake, PullDefault, 0x0000,
    "\\_SB.GPI0", 0x00, ResourceConsumer, ,
    )
    {   // Pin list
        0x0000
    }
```

Change only `Exclusive` → `ExclusiveAndWake`.

### 4.5 Fix compile errors (if any)

If `iasl` reports errors like:

```
Error 6163 - Object is created temporarily in another method and cannot be accessed
(_SB_.ATKD.WMNB.MDSA)
```

These come from auto-generated `External` declarations for method-local objects. Delete the offending lines (e.g., lines 62–63 in the decompiled file):

```bash
sed -i '62,63d' dsdt.dsl
```

Re-run the compile to check for remaining errors.

### 4.6 Bump the OEM Revision (critical)

The kernel only applies an ACPI table override if the new table has a **higher OEM Revision** than the firmware table.

Open `dsdt.dsl` and find the `DefinitionBlock` line near the top:

```asl
DefinitionBlock ("", "DSDT", 2, "_ASUS_", "Notebook", 0x01072009)
```

Change the last number to a higher value, e.g. `0x01072010`:

```asl
DefinitionBlock ("", "DSDT", 2, "_ASUS_", "Notebook", 0x01072010)
```

Without this, the kernel silently discards the override.

### 4.7 Recompile

```bash
iasl -tc dsdt.dsl
```

You should see:

```
Compilation successful. 0 Errors, ... Warnings, ... Remarks
AML Output: dsdt.aml
```

Warnings and remarks are normal.

### 4.8 Install the override

```bash
sudo mkdir -p /etc/initcpio/acpi_override
sudo cp dsdt.aml /etc/initcpio/acpi_override/dsdt.aml
```

> The `acpi_override` mkinitcpio hook reads from `/etc/initcpio/acpi_override/` and `/usr/lib/initcpio/acpi_override/`. It does **not** read from `/boot/acpi_override/`.

### 4.9 Add the `acpi_override` hook

Edit `/etc/mkinitcpio.conf` and add `acpi_override` to the `HOOKS` array. A safe placement is right after `systemd`:

```conf
HOOKS=(base systemd acpi_override autodetect microcode kms modconf block keyboard sd-vconsole plymouth filesystems sd-btrfs-overlayfs)
```

Save the file.

### 4.10 Rebuild the initramfs

```bash
sudo mkinitcpio -P
```

If you use Limine, it may prompt:

```
==> Would you like to run 'limine-mkinitcpio' now? [Y/n]:
```

Press **Enter** to accept.

The output should include:

```
-> Running build hook: [acpi_override]
```

and **no** `ERROR: mkinitcpio failed` line.

### 4.11 Verify the override is in the initramfs

With Limine, the initramfs path is not `/boot/initramfs-linux-cachyos.img`. Find the actual path from the mkinitcpio output, e.g.:

```bash
sudo lsinitcpio /boot/5b5d0010bbb1479aa480e113957718c5/linux-cachyos/initramfs | grep -i dsdt
```

Expected:

```
kernel/firmware/acpi/dsdt.aml
```

### 4.12 Ensure `acpi=force` is on the kernel cmdline

Check:

```bash
cat /proc/cmdline
```

If `acpi=force` is missing, add it to your Limine configuration (`/boot/limine.conf` or `/etc/default/limine`) and update Limine.

Example Limine entry:

```conf
cmdline: quiet nowatchdog splash acpi=force rw rootflags=subvol=/@ root=UUID=...
```

### 4.13 Reboot and verify the override loaded

```bash
sudo reboot
```

After reboot:

```bash
sudo dmesg | grep -iE "DSDT|override"
```

You must see a line like:

```
ACPI: Table Upgrade: override [DSDT-_ASUS_-Notebook]
```

and the new DSDT size should match your patched file:

```bash
sudo cat /sys/firmware/acpi/tables/DSDT | wc -c
```

Expected: `462198` (or whatever size your patched `dsdt.aml` has). If it still shows the original size (e.g., `466281`), the override was not applied — check Section 6.

### 4.14 Test suspend

```bash
systemctl suspend
```

Try waking, in order:

1. **Power button** — short press.
2. **Lid** — close, wait a few seconds, open.
3. **Keyboard** — press any key.

If all three work, the fix is complete.

---

## 5. What Was Tested and Failed (Historical)

These attempts did **not** solve the problem, but are documented so you don’t repeat them:

- Forcing S3 sleep (`mem_sleep_default=deep`) → immediate reboot (Alder Lake ASUS boards lack S3 rails).
- NVIDIA power management tuning (`NVreg_DynamicPowerManagement`, etc.) → no effect on wake.
- Disabling NVIDIA suspend/resume helper services → no effect.
- `SYSTEMD_SLEEP_FREEZE_USER_SESSIONS=false` → no effect.
- `AQ_DRM_DEVICES` pinning → no effect on wake.
- Setting input sysfs wake nodes (`/sys/class/input/*/power/wakeup`) → no effect.
- `atkbd.reset=1`, `i8042.reset=1`, `i8042.nopnp=1` → no effect.
- Masking `gpe02` → no effect.
- Adding `_PRW` without bumping OEM Revision → kernel ignored the override.
- Installing override in `/boot/acpi_override/` → mkinitcpio hook does not read there.

---

## 6. Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| `dmesg` shows “DSDT ACPI table found in initrd” but no “Table Upgrade: override” | OEM Revision not bumped | Edit `DefinitionBlock` line, increase revision, recompile, reinstall, rebuild initramfs |
| `mkinitcpio` reports “No ACPI table override files found!” | File in wrong directory | Place `dsdt.aml` in `/etc/initcpio/acpi_override/` |
| `lsinitcpio` shows no `dsdt.aml` | `acpi_override` hook missing from `HOOKS` | Add hook to `/etc/mkinitcpio.conf`, rebuild |
| Override still not applied after reboot | Secure Boot or kernel lockdown | Disable Secure Boot in BIOS; check `cat /sys/kernel/security/lockdown` |
| Override still not applied | `acpi=force` missing | Add `acpi=force` to kernel cmdline |
| Compile errors about method-local objects | Auto-generated `External` lines | Delete those lines (e.g., `sed -i '62,63d' dsdt.dsl`) |
| Power button wakes but lid/keyboard don’t | Different GPE or missing `_PRW` on other devices | Patch `PNP0C0C` (power button) or EC device similarly |
| Lid wakes but display doesn’t restore | Compositor/GPU resume issue | Separate problem; check Hyprland/NVIDIA logs |

---

## 7. Maintenance and Notes

- **BIOS updates invalidate the override.** If ASUS releases a new BIOS, re-extract, re-patch, and re-bump the revision.
- **Keep a backup** of your patched `dsdt.dsl` and `dsdt.aml` somewhere safe (e.g., `~/sleep/`).
- **Kernel updates:** `mkinitcpio -P` is usually run automatically on kernel updates; as long as the `acpi_override` hook is in `HOOKS` and the file is in `/etc/initcpio/acpi_override/`, the override will be included.
- **Limine initramfs paths** are machine-specific. Use `lsinitcpio` with the path from `mkinitcpio` output.
- **Alternative fallback:** If the mkinitcpio hook ever fails, you can manually create a CPIO archive and load it via Limine’s `module_path` directive. See Section 8.

---

## 8. Alternative Fallback: Manual CPIO Override

If the `acpi_override` hook does not work for any reason, you can load the DSDT directly via a CPIO archive.

```bash
mkdir -p /tmp/acpi_override/kernel/firmware/acpi
cp dsdt.aml /tmp/acpi_override/kernel/firmware/acpi/
cd /tmp/acpi_override
find . -print0 | cpio --null -ov --format=newc > /boot/acpi_override.cpio
```

Then edit `/boot/limine.conf` and add a second `module_path` line to your kernel entry:

```conf
module_path: boot():/5b5d0010bbb1479aa480e113957718c5/linux-cachyos/initramfs
module_path: boot():/acpi_override.cpio
```

Reboot and verify with `dmesg | grep -i override`.

---

## 9. Files and Locations

| Path | Purpose |
|------|---------|
| `/etc/initcpio/acpi_override/dsdt.aml` | Patched DSDT used by mkinitcpio hook |
| `/etc/mkinitcpio.conf` | Contains `acpi_override` in `HOOKS` |
| `/boot/<machine-id>/linux-cachyos/initramfs` | Limine initramfs path (example) |
| `/boot/limine.conf` | Limine boot entries and kernel cmdline |
| `~/sleep/dsdt.dsl` | Source patch (keep backup) |

---

## 10. References

- CachyOS forum thread: “ASUS TUF F15 FX507ZI4 suspend wake fix” (user with same model confirmed this DSDT patch).
- Linux kernel documentation: `Documentation/admin-guide/acpi/initrd_table_override.rst`.
- `acpi_override` mkinitcpio hook source: `/usr/lib/initcpio/install/acpi_override`.

---

**End of document.**