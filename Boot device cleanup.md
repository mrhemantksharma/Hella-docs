# RUNBOOK: Linux Kernel Cleanup & /boot Space Management

## Purpose
This runbook describes the **safe, production-approved procedure** to:
- Identify the running Linux kernel
- Remove unused / old kernel versions
- Free disk space in `/boot`
- Configure automatic kernel retention
- Recover safely in case of boot or kernel issues

Applicable to **RPM-based Linux distributions**:
- RHEL
- Oracle Linux
- Rocky Linux
- AlmaLinux
- CentOS Stream

---

## ⚠️ Risk Statement

Improper kernel removal can result in:
- System boot failure
- Missing GRUB entries
- Loss of recovery options

**Follow this runbook exactly.**  
**Never delete kernel files manually.**

---

## 1. Pre-Checks (MANDATORY)

### 1.1 Identify Running Kernel
```bash
uname -r
```

✔ Identifies the active kernel  
❌ This kernel **must not be removed**

---

### 1.2 Check `/boot` Usage
```bash
df -h /boot
```

Purpose:
- Confirm disk pressure
- Establish pre-change baseline

---

### 1.3 List Installed Kernel Packages
```bash
rpm -qa | grep kernel
```

Purpose:
- Identify all installed kernel versions
- Identify kernel types (standard / UEK)

---

## 2. Kernel File Types (Reference)

Each kernel version stores the following files in `/boot`:

| File | Description |
|----|----|
| `vmlinuz-*` | Compressed Linux kernel loaded by GRUB |
| `initramfs-*` | Initial RAM filesystem used during early boot |
| `initramfs-*kdump.img` | Kernel crash dump image |
| `System.map-*` | Kernel symbol-to-address map |
| `symvers-*` | Symbol version table for kernel modules |
| `config-*` | Kernel build configuration |
| `vmlinuz-0-rescue-*` | Emergency rescue kernel |
| `initramfs-0-rescue-*` | Rescue initramfs image |

---

## 3. Rescue Kernel (DO NOT REMOVE)

Files:
```text
vmlinuz-0-rescue-*
initramfs-0-rescue-*
```

Purpose:
- Emergency recovery
- Boot when normal kernels fail
- Filesystem repair

⚠️ **Never remove rescue kernel files**

---

## 4. Kernel Package Structure (RPM)

Kernel versions are split into multiple RPMs:

| Package | Purpose |
|----|----|
| `kernel` / `kernel-uek` | Meta package |
| `kernel-core` | Kernel binary |
| `kernel-modules` | Device drivers |
| `kernel-headers` | Kernel headers |
| `kernel-tools*` | Performance and debugging tools |

Removing the main kernel package automatically removes all related subpackages.

---

## 5. Safe Kernel Retention Policy

✔ Keep:
- Current running kernel
- One fallback kernel
- Rescue kernel

❌ Remove:
- All older unused kernels

Target state:
```text
Maximum installed kernels = 2 (+ rescue)
```

---

## 6. Kernel Removal Procedures (PRODUCTION SAFE)

### 6.1 Manual Kernel Cleanup (Targeted)

```bash
dnf remove 'kernel*-<old-version>*'
```

Example:
```bash
dnf remove 'kernel*-4.18.0-*'
```

Notes:
- Wildcards **must be quoted**
- Prevents shell expansion
- Allows DNF to handle dependencies safely

---

### 6.2 Automatic Old Kernel Cleanup (RECOMMENDED)

```bash
dnf remove --oldinstallonly --setopt installonly_limit=2 kernel kernel-uek
```

Explanation:
- `--oldinstallonly` → removes only old install-only packages
- `installonly_limit=2` → keeps current + one fallback kernel
- `kernel kernel-uek` → applies to both standard and UEK kernels

✔ Running kernel protected  
✔ Rescue kernel unaffected  
✔ GRUB updated automatically  

---

## 7. Verification Steps

### 7.1 Verify Kernel Packages
```bash
rpm -qa | grep kernel
```

Expected:
- Only two kernel versions present

---

### 7.2 Verify `/boot` Space
```bash
df -h /boot
```

Expected:
- Increased free space

---

### 7.3 Verify Kernel Files
```bash
ls /boot | grep vmlinuz
```

Expected:
- Active kernel
- One fallback kernel
- Rescue kernel

---

## 8. Preventive Configuration (MANDATORY)

Persist kernel retention limit:

```bash
vi /etc/dnf/dnf.conf
```

Add or update:
```ini
installonly_limit=2
```

---

## 9. Rollback Procedures (CRITICAL)

### 9.1 Boot-Time Rollback (GRUB)

If system fails to boot after cleanup:

1. Reboot system
2. At GRUB menu, select:
   - Previous kernel version (fallback), or
   - Rescue kernel
3. Boot into system

---

### 9.2 Reinstall Kernel Package (From Running System)

List available kernel versions:
```bash
dnf list kernel kernel-uek --showduplicates
```

Install a specific kernel version:
```bash
dnf install kernel-<version>
# OR
dnf install kernel-uek-<version>
```

---

### 9.3 Rebuild Initramfs (If Boot Issues Persist)

```bash
dracut -f
```

For a specific kernel:
```bash
dracut -f /boot/initramfs-<kernel-version>.img <kernel-version>
```

---

### 9.4 Rebuild GRUB Configuration

BIOS systems:
```bash
grub2-mkconfig -o /boot/grub2/grub.cfg
```

UEFI systems:
```bash
grub2-mkconfig -o /boot/efi/EFI/*/grub.cfg
```

---

### 9.5 Rescue Mode Recovery (Last Resort)

Boot into rescue kernel and run:
```bash
mount -o remount,rw /
dnf reinstall kernel kernel-uek
```

---

## 10. Prohibited Actions

❌ Do not delete kernel files manually  
❌ Do not remove running kernel  
❌ Do not remove rescue kernel  
❌ Do not reboot without verification  
❌ Do not use unquoted wildcards  

---

## 11. Post-Change Notes

- Kernel cleanup does **not require reboot**
- Changes are immediate
- Future kernel installs respect retention limits

---

## 12. Summary

This runbook provides a **repeatable, auditable, and production-safe** approach to:
- Kernel lifecycle management
- `/boot` disk space recovery
- Automated kernel retention
- Reliable rollback and recovery

---

## 13. Change Management Checklist

- [ ] Running kernel verified
- [ ] Cleanup method selected
- [ ] Removal list reviewed
- [ ] `/boot` space validated
- [ ] Retention limit configured
- [ ] Rollback path confirmed

---

## License

MIT License
