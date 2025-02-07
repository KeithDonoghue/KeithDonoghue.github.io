---
layout: post
title: "Toward a better GRUB"
date:   2025-01-23 15:35:19 -0300
categories: linux bootloader
---

## The scene
GRUB's boot selection menu is an absolute mess I don't know why I care about this, but it drives me insane. Every OS you install dumps its own entries in there, mixing in recovery modes, old kernels, and random tools you never asked for. Instead of a clean, usable bootloader, you end up scrolling through a junk drawer of boot options.

I want:
- **One clean entry per OS**, loading the normal kernel with standard settings.
- **A submenu for advanced options**, containing recovery modes and older kernels.
- **A separate tools menu** for utilities like memtest and firmware updates.

To actually fix this mess, let's first break down the **boot process**, which consists of:
1. **UEFI Firmware Phase** – The firmware initializes hardware and selects a bootloader.
2. **GRUB Phase** – GRUB loads the system configuration and presents the boot menu.
3. **Kernel Execution** – GRUB hands control over to the Linux kernel.

## Step 1: The UEFI Boot Order
UEFI firmware stores boot settings in **NVRAM (Non-Volatile RAM)**, which can be managed in Linux using `efibootmgr`. Below is my system's current UEFI boot order:

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

### **What's Happening Here?**
- **`search.fs_uuid`** finds the correct root filesystem via UUID not PART_UUID.
- **`set prefix=($root)/boot/grub[2]`** defines where to load the full GRUB configuration.
- **`configfile $prefix/grub.cfg`** loads the full GRUB menu settings from the OS's `/boot` partition.

The only meaningful difference here is the different directory they use `/boot/grub` for bunts and `/boot/grub2` for fez.


I tend not to keep `/boot` itself on a separate partition even if in theory it would be more efficient, I just don't trust the different OS' not to trample eachother. Check out the [official specification][boot-loader-specs] for more (in truth probably waaaay too much) information on how the various boot loaders are supposed to play nice with eachother.



I'm going to go with the Fedora grub entry as the primary loader, so the file we need to change is:  `/boot/grub2/grub.cfg`

There are a set of scrips in ```/boot/grub.d``` which are invoked in priority order by grub2-mkconfig.

of import are 10_linux, loads the current OS' config
30_os_prober - finds other OS partitions and adds entries.
40_custom, where we can add our own stuff.


Our first problem is that fedora uses blscfg to load it's own boot options, these options come from the files in ```/boot/loader/entries``` with one file per boot option. Tbf, this is nice as it allows for each boot option to be delete/added/edited independent of any of the others. But for now it's gotta go.


So first things first, disabable blscfg.

```sudo grub2-editenv - unset blscfg```

From there we just run the normal grub2-mkconfig to generate the grub.cfg file.

```sudo grub2-mkconfig -o /boot/grub2/grub.cfg```

This gives us the set of options we need, from here we just re-arrange the entries to match that described above.

<details>
<summary>Click to view the complete GRUB configuration</summary>

```
# Main OS Entries
menuentry 'Fedora Linux 41 (Workstation Edition)' --class fedora --class gnu-linux --class gnu --class os --unrestricted $menuentry_id_option 'gnulinux-simple-a7cdf71e-2219-423a-ac09-d6a020c22409' {
    load_video
    set gfxpayload=keep
    insmod gzio
    insmod part_gpt
    insmod ext2
    search --no-floppy --fs-uuid --set=root a7cdf71e-2219-423a-ac09-d6a020c22409
    echo    'Loading Linux 6.12.11-200.fc41.x86_64 ...'
    linux   /boot/vmlinuz-6.12.11-200.fc41.x86_64 root=UUID=a7cdf71e-2219-423a-ac09-d6a020c22409 ro rhgb quiet 
    echo    'Loading initial ramdisk ...'
    initrd  /boot/initramfs-6.12.11-200.fc41.x86_64.img
}

menuentry 'Ubuntu 24.04.1 LTS' --class ubuntu --class gnu-linux --class gnu --class os $menuentry_id_option 'gnulinux-simple-f6be26a7-9f80-4334-8de3-3c80282de363' {
    insmod part_gpt
    insmod ext2
    search --no-floppy --fs-uuid --set=root f6be26a7-9f80-4334-8de3-3c80282de363
    linux /boot/vmlinuz-6.8.0-51-generic root=UUID=f6be26a7-9f80-4334-8de3-3c80282de363 ro quiet splash $vt_handoff
    initrd /boot/initrd.img-6.8.0-51-generic
}

menuentry 'Ubuntu 24.04' --class ubuntu --class gnu-linux --class gnu --class os $menuentry_id_option 'gnulinux-simple-1eb5e7ee-43ad-40f6-b862-7986c8464659' {
    insmod part_gpt
    insmod ext2
    search --no-floppy --fs-uuid --set=root 1eb5e7ee-43ad-40f6-b862-7986c8464659
    linux   /boot/vmlinuz-6.11.0-13-generic root=UUID=1eb5e7ee-43ad-40f6-b862-7986c8464659 ro quiet splash crashkernel=2G-4G:320M,4G-32G:512M,32G-64G:1024M,64G-128G:2048M,128G-:4096M $vt_handoff
    initrd  /boot/initrd.img-6.11.0-13-generic
}

menuentry 'Arch Linux' --class arch --class gnu-linux --class gnu --class os $menuentry_id_option 'osprober-gnulinux-simple-3d48a87e-0ec8-41e5-9df0-ceec7ec1c44f' {
    insmod part_gpt
    insmod ext2
    search --no-floppy --fs-uuid --set=root 3d48a87e-0ec8-41e5-9df0-ceec7ec1c44f
    linux /boot/vmlinuz-linux root=/dev/nvme0n1p11
    initrd /boot/initramfs-linux.img
}

menuentry 'Linux From Scratch' --class gnu-linux --class gnu --class os $menuentry_id_option 'osprober-gnulinux-simple-85204b3d-eeca-4e8f-8cbf-dc51e025bf78' {
    insmod part_gpt
    insmod ext2
    search --no-floppy --fs-uuid --set=root 85204b3d-eeca-4e8f-8cbf-dc51e025bf78
    linux /boot/vmlinuz-6.10.5-lfs-12.2 root=/dev/nvme0n1p10
}

# Advanced Options Submenu
submenu 'Advanced Options' {
    # Fedora section
    menuentry 'Fedora Linux (6.12.9-200.fc41.x86_64)' --class fedora --class gnu-linux --class gnu --class os --unrestricted $menuentry_id_option 'gnulinux-advanced-a7cdf71e-2219-423a-ac09-d6a020c22409' {
        load_video
        set gfxpayload=keep
        insmod gzio
        insmod part_gpt
        insmod ext2
        search --no-floppy --fs-uuid --set=root a7cdf71e-2219-423a-ac09-d6a020c22409
        echo    'Loading Linux 6.12.9-200.fc41.x86_64 ...'
        linux   /boot/vmlinuz-6.12.9-200.fc41.x86_64 root=UUID=a7cdf71e-2219-423a-ac09-d6a020c22409 ro rhgb quiet 
        initrd  /boot/initramfs-6.12.9-200.fc41.x86_64.img
    }
    
    menuentry 'Fedora Linux (6.12.5-200.fc41.x86_64)' --class fedora --class gnu-linux --class gnu --class os --unrestricted $menuentry_id_option 'gnulinux-advanced-a7cdf71e-2219-423a-ac09-d6a020c22409' {
        load_video
        set gfxpayload=keep
        insmod gzio
        insmod part_gpt
        insmod ext2
        search --no-floppy --fs-uuid --set=root a7cdf71e-2219-423a-ac09-d6a020c22409
        echo    'Loading Linux 6.12.5-200.fc41.x86_64 ...'
        linux   /boot/vmlinuz-6.12.5-200.fc41.x86_64 root=UUID=a7cdf71e-2219-423a-ac09-d6a020c22409 ro rhgb quiet 
        initrd  /boot/initramfs-6.12.5-200.fc41.x86_64.img
    }

    menuentry 'Fedora Linux (6.12.4-200.fc41.x86_64)' --class fedora --class gnu-linux --class gnu --class os --unrestricted $menuentry_id_option 'gnulinux-advanced-a7cdf71e-2219-423a-ac09-d6a020c22409' {
        load_video
        set gfxpayload=keep
        insmod gzio
        insmod part_gpt
        insmod ext2
        search --no-floppy --fs-uuid --set=root a7cdf71e-2219-423a-ac09-d6a020c22409
        echo    'Loading Linux 6.12.4-200.fc41.x86_64 ...'
        linux   /boot/vmlinuz-6.12.4-200.fc41.x86_64 root=UUID=a7cdf71e-2219-423a-ac09-d6a020c22409 ro rhgb quiet 
        initrd  /boot/initramfs-6.12.4-200.fc41.x86_64.img
    }

    menuentry 'Fedora Linux (Rescue)' --class fedora --class gnu-linux --class gnu --class os --unrestricted $menuentry_id_option 'gnulinux-rescue-a7cdf71e-2219-423a-ac09-d6a020c22409' {
        load_video
        insmod gzio
        insmod part_gpt
        insmod ext2
        search --no-floppy --fs-uuid --set=root a7cdf71e-2219-423a-ac09-d6a020c22409
        echo    'Loading Linux 0-rescue-fd247ce0ebe345f6b569283b6eda528b ...'
        linux   /boot/vmlinuz-0-rescue-fd247ce0ebe345f6b569283b6eda528b root=UUID=a7cdf71e-2219-423a-ac09-d6a020c22409 ro rhgb quiet 
        initrd  /boot/initramfs-0-rescue-fd247ce0ebe345f6b569283b6eda528b.img
    }

    # Ubuntu 24.04.1 LTS section
    menuentry 'Ubuntu 24.04.1 LTS (6.8.0-51-generic)' --class ubuntu --class gnu-linux --class gnu --class os $menuentry_id_option 'gnulinux-advanced-f6be26a7-9f80-4334-8de3-3c80282de363' {
        insmod part_gpt
        insmod ext2
        search --no-floppy --fs-uuid --set=root f6be26a7-9f80-4334-8de3-3c80282de363
        linux /boot/vmlinuz-6.8.0-51-generic root=UUID=f6be26a7-9f80-4334-8de3-3c80282de363 ro quiet splash $vt_handoff
        initrd /boot/initrd.img-6.8.0-51-generic
    }

    menuentry 'Ubuntu 24.04.1 LTS (Recovery mode)' --class ubuntu --class gnu-linux --class gnu --class os $menuentry_id_option 'gnulinux-advanced-recovery-f6be26a7-9f80-4334-8de3-3c80282de363' {
        insmod part_gpt
        insmod ext2
        search --no-floppy --fs-uuid --set=root f6be26a7-9f80-4334-8de3-3c80282de363
        linux /boot/vmlinuz-6.8.0-51-generic root=UUID=f6be26a7-9f80-4334-8de3-3c80282de363 ro recovery nomodeset dis_ucode_ldr
        initrd /boot/initrd.img-6.8.0-51-generic
    }

    # Ubuntu 24.04 section
    menuentry 'Ubuntu 24.04 (6.11.0-13-generic)' --class ubuntu --class gnu-linux --class gnu --class os $menuentry_id_option 'gnulinux-advanced-1eb5e7ee-43ad-40f6-b862-7986c8464659' {
        insmod part_gpt
        insmod ext2
        search --no-floppy --fs-uuid --set=root 1eb5e7ee-43ad-40f6-b862-7986c8464659
        linux /boot/vmlinuz-6.11.0-13-generic root=UUID=1eb5e7ee-43ad-40f6-b862-7986c8464659 ro quiet splash crashkernel=2G-4G:320M,4G-32G:512M,32G-64G:1024M,64G-128G:2048M,128G-:4096M $vt_handoff
        initrd /boot/initrd.img-6.11.0-13-generic
    }

    menuentry 'Ubuntu 24.04 (Recovery mode)' --class ubuntu --class gnu-linux --class gnu --class os $menuentry_id_option 'gnulinux-advanced-recovery-1eb5e7ee-43ad-40f6-b862-7986c8464659' {
        insmod part_gpt
        insmod ext2
        search --no-floppy --fs-uuid --set=root 1eb5e7ee-43ad-40f6-b862-7986c8464659
        linux /boot/vmlinuz-6.11.0-13-generic root=UUID=1eb5e7ee-43ad-40f6-b862-7986c8464659 ro recovery nomodeset dis_ucode_ldr
        initrd /boot/initrd.img-6.11.0-13-generic
    }
}

# System Tools Submenu
submenu 'System Tools' {
    menuentry 'Memory Test (memtest86+)' --class memtest $menuentry_id_option 'memtest86+' {
        insmod part_gpt
        insmod ext2
        search --no-floppy --fs-uuid --set=root f6be26a7-9f80-4334-8de3-3c80282de363
        linux /boot/memtest86+x64.efi
    }

    menuentry 'Memory Test (Serial Console)' --class memtest $menuentry_id_option 'memtest86+-serial' {
        insmod part_gpt
        insmod ext2
        search --no-floppy --fs-uuid --set=root f6be26a7-9f80-4334-8de3-3c80282de363
        linux /boot/memtest86+x64.efi console=ttyS0,115200
    }

    menuentry 'UEFI Firmware Settings' $menuentry_id_option 'uefi-firmware' {
        fwsetup
    }
}
```

</details>


Of course, this is a rather manual process, and it will need to be run whenever the OS is updated, but it'll do for now.

If I find something better, I'll update.


---
*Keep it real, space cowboys...*
---



[boot-loader-specs]: https://uapi-group.org/specifications/specs/boot_loader_specification/
