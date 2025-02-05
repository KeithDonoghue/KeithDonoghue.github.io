---
layout: post
title:  "Grub basics"
date:   2025-01-23 15:35:19 -0300
categories: jekyll update
---

I don't know why I care about this, but the grub boot selection menu drives me insane. I don't want a list of 40 different OS configurations, with a recovery option and without, every differnt conmbination of kernel that OS has updated to etc. etc.

I want one option for each distro, with a nice simple name which loads the normal kernel with the normal settings. I do then want an extra screen with all the rest of the crap if I need it. 

I guess I'll also tolerate a tools menu. Right, two menues, one for other OS options, and one for tools -memtest and firmware updater God love them.


Let's first take a tour of the boot process, We have two steps, the EFI firmware phase, followed by the GRUB phase (this is the one that interests us), before finally passing control to the linux kernel.

The `efibootmgr` command in Linux allows you to view and manage your system's UEFI boot configuration. This is what it outputs on my machine:

### Boot Configuration

```html
BootCurrent: 0000
Timeout: 2 seconds
BootOrder: 0000, 0003, 0002
Boot0000* Ubuntu                          HD(1,GPT,8f756691-3c00-4eef-99e7-d5b15cadf461,0x800,0x9b4000)/\EFI\ubuntu\shimx64.efi
Boot0002* Linux Firmware Updater         HD(1,GPT,8f756691-3c00-4eef-99e7-d5b15cadf461,0x800,0x9b4000)/\EFI\fedora\fwupdx64.efi
Boot0003* Fedora                         HD(1,GPT,8f756691-3c00-4eef-99e7-d5b15cadf461,0x800,0x9b4000)/\EFI\fedora\shimx64.efi
```

This info is stored  NVRAM (Non-Volatile RAM) on the motherboard, part of the UEFI firmwareand can also be seen in the `/sys/firmware/efi/efivars/` directory after boot.

We can see `HD(1,GPT,8f756691-3c00-4eef-99e7-d5b15cadf461,0x800,0x9b4000)` is almost like a function call which returns the appropriate partition, to wich the path to the .efi file is appended.

In our case only the file specified to boot varies, all entries are located on the same EFI partition with these details:
- Partition number: 1
- Partition Type: GPT
- PART_UUID: 8f756691-3c00-4eef-99e7-d5b15cadf461
- Partition Location: 0x800,0x9b4000

## Current Configuration
- **BootCurrent**: 0000 (Ubuntu) - System that last booted, i.e. whatever booted the current system 
- **Timeout**: 2 seconds - How long to wait before running the highest priority boot option.

## Boot Order
The system will attempt to boot operating systems in this sequence:
1. 0000 (Ubuntu)
2. 0003 (Fedora)
3. 0002 (Linux Firmware Updater)

## Available Boot Entries

The following table shows all configured boot entries in the system:

| Boot Number | Operating System/Entry    | EFI Path                    |
|-------------|---------------------------|---------------------------- |
| Boot0000*   | Ubuntu                    | `\EFI\ubuntu\shimx64.efi`   |
| Boot0002*   | Linux Firmware Updater    | `\EFI\fedora\fwupdx64.efi`  |
| Boot0003*   | Fedora                    | `\EFI\fedora\shimx64.efi`   |



This specifies a partition to mount, this is shown in lsblk as the PARTUUID i.e. the partition ID independent of the filesystem the partition is formatted with, EFI firmware only understand FAT32.

It then executes the file specified. This is usually shimx64.efi in our case, i.e. an thin chainloader basically which just hands over to grubx64.efi. The shimx64.efi file however is signed by microsoft and thus can be used even with secure boot, unlike if we were to use grubx64.efi directly.


This initial grub runs a thin grub.cfg, as the main /boot partition has not yet been mounted. The UUID of the filesystem(not partition) is specified as / or /boot and from there the path to the proper grub.cfg file is used.




Let's break down what this output tells us:



This configuration shows a dual-boot system with both Ubuntu and Fedora installed, with Ubuntu being the default boot option.

### GRUB Configuration

### File: `/boot/efi/EFI/ubuntu/grub.cfg`

```
search.fs_uuid 1eb5e7ee-43ad-40f6-b862-7986c8464659 root 
set prefix=($root)'/boot/grub'
configfile $prefix/grub.cfg
```


### File: `/boot/efi/EFI/fedora/grub.cfg`

```
search --no-floppy --root-dev-only --fs-uuid --set=dev a7cdf71e-2219-423a-ac09-d6a020c22409
set prefix=($dev)/boot/grub2
export $prefix
configfile $prefix/grub.cfg
```

Aside from the different method used to indicate the appropriate filesystem(not necessarily partition) the two files are the same, they specify the FS and path to the full bootloader config file. This is possible from here, as now that grub is executing any FS type can be mounted.

I tend not to keep `/boot` itself on a separate partition even if in theory it would be more efficient, I just don't trust the different OS' not to trample eachother. Check out the [official specification][boot-loader-specs] for more (in truth probably waaaay too much) information on how the various boot loaders are supposed to play nice with eachother.

[boot-loader-specs]: https://uapi-group.org/specifications/specs/boot_loader_specification/
