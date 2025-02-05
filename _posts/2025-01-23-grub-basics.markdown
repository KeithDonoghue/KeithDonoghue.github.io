---
layout: post
title:  "Grub basics"
date:   2025-01-23 15:35:19 -0300
categories: linux bootloader
---

## Introduction
GRUB’s boot selection menu is an absolute mess I don't know why I care about this, but it drives me insane. Every OS you install dumps its own entries in there, mixing in recovery modes, old kernels, and random tools you never asked for. Instead of a clean, usable bootloader, you end up scrolling through a junk drawer of boot options.

I want:
- **One clean entry per OS**, loading the normal kernel with standard settings.
- **A submenu for advanced options**, containing recovery modes and older kernels.
- **A separate tools menu** for utilities like memtest and firmware updates.

To actually fix this mess, let’s first break down the **boot process**, which consists of:
1. **UEFI Firmware Phase** – The firmware initializes hardware and selects a bootloader.
2. **GRUB Phase** – GRUB loads the system configuration and presents the boot menu.
3. **Kernel Execution** – GRUB hands control over to the Linux kernel.

## Step 1: The UEFI Boot Order
UEFI firmware stores boot settings in **NVRAM (Non-Volatile RAM)**, which can be managed in Linux using `efibootmgr`. Below is my system’s current UEFI boot order:

### Checking Boot Configuration
```bash
efibootmgr
```
#### **Output:**
```html
BootCurrent: 0000
Timeout: 2 seconds
BootOrder: 0000, 0003, 0002
Boot0000* Ubuntu                          HD(1,GPT,8f756691-3c00-4eef-99e7-d5b15cadf461,0x800,0x9b4000)/EFI/ubuntu/shimx64.efi
Boot0002* Linux Firmware Updater         HD(1,GPT,8f756691-3c00-4eef-99e7-d5b15cadf461,0x800,0x9b4000)/EFI/fedora/fwupdx64.efi
Boot0003* Fedora                         HD(1,GPT,8f756691-3c00-4eef-99e7-d5b15cadf461,0x800,0x9b4000)/EFI/fedora/shimx64.efi
```

### **Key Takeaways**
- **BootCurrent:** The last booted OS (Ubuntu in this case).
- **Timeout:** The system waits 2 seconds before selecting the first boot entry.
- **Boot Order:** The system attempts to boot in this sequence:
  1. `0000` - Ubuntu
  2. `0003` - Fedora
  3. `0002` - Linux Firmware Updater

| Boot Number | Entry Name                | EFI File Path                   |
|-------------|---------------------------|---------------------------------|
| Boot0000*   | Ubuntu                    | `/EFI/ubuntu/shimx64.efi`       |
| Boot0002*   | Linux Firmware Updater    | `/EFI/fedora/fwupdx64.efi`      |
| Boot0003*   | Fedora                    | `/EFI/fedora/shimx64.efi`       |

## Step 2: How UEFI Chooses a Bootloader
Each UEFI boot entry points to an `.efi` file located on the **EFI System Partition (ESP)**. EFI firmware follows these steps:
1. **Identifies the boot partition** (Partition 1, formatted as FAT32, with PARTUUID `8f756691-3c00-4eef-99e7-d5b15cadf461`).
2. **Executes the `.efi` bootloader file** specified in the boot entry.
3. **Loads GRUB (via `shimx64.efi`)**, which then loads `grubx64.efi`.

#### **Why Use `shimx64.efi` Instead of `grubx64.efi`?**
- `shimx64.efi` is **signed by Microsoft**, allowing it to run even with Secure Boot enabled.
- It acts as a **thin chainloader**, passing control to `grubx64.efi`, which is not directly signed.

## Step 3: The GRUB Boot Process
Once `grubx64.efi` is executed, it loads a small **preliminary GRUB configuration file** stored on the ESP, usually just grub.cfg in the same directory as `grubx64.efi`. My understanding here is that it is necessary to invoke grub in order to be able to mount the `/` or `/boot` system in case they are non FAT32 file systems.

### Example: Initial GRUB Configuration Files
#### **Ubuntu GRUB Config (`/boot/efi/EFI/ubuntu/grub.cfg`)**
```bash
search.fs_uuid 1eb5e7ee-43ad-40f6-b862-7986c8464659 root
set prefix=($root)'/boot/grub'
configfile $prefix/grub.cfg
```

#### **Fedora GRUB Config (`/boot/efi/EFI/fedora/grub.cfg`)**
```bash
search --no-floppy --root-dev-only --fs-uuid --set=dev a7cdf71e-2219-423a-ac09-d6a020c22409
set prefix=($dev)/boot/grub2
export $prefix
configfile $prefix/grub.cfg
```

### **What’s Happening Here?**
- **`search.fs_uuid`** finds the correct root filesystem via UUID not PART_UUID.
- **`set prefix=($root)/boot/grub[2]`** defines where to load the full GRUB configuration.
- **`configfile $prefix/grub.cfg`** loads the full GRUB menu settings from the OS’s `/boot` partition.

The only meaningful difference here is the different directory they use `/boot/grub` for bunts and `/boot/grub2` for fez.


I tend not to keep `/boot` itself on a separate partition even if in theory it would be more efficient, I just don't trust the different OS' not to trample eachother. Check out the [official specification][boot-loader-specs] for more (in truth probably waaaay too much) information on how the various boot loaders are supposed to play nice with eachother.

[boot-loader-specs]: https://uapi-group.org/specifications/specs/boot_loader_specification/
