---
layout: post
title:  "Grub basics"
date:   2025-01-23 15:35:19 -0300
categories: jekyll update
---

The `efibootmgr` command in Linux allows you to view and manage your system's UEFI boot configuration. Let's look at an example output:

<figure>
  <img src="/assets/images/efibootmgrOut.png" alt="Output of efibootmgr command showing UEFI boot entries">
  <figcaption>Terminal output showing UEFI boot configuration</figcaption>
</figure>

Let's break down what this output tells us:

## Current Configuration
- **Current Boot Entry**: 0000 (Ubuntu)
- **Boot Timeout**: 2 seconds

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

All entries are located on the same EFI partition with these details:
- Partition Type: GPT
- UUID: 8f756691-3c00-4eef-99e7-d5b15cadf461
- Partition Location: 0x800,0x9b4000

This configuration shows a dual-boot system with both Ubuntu and Fedora installed, with Ubuntu being the default boot option.


I tend not to keep `/boot` itself on a separate partition even if in theory it would be more efficient, I just don't trust the different OS' not to trample eachother. Check out the [official specification][boot-loader-specs] for more (in truth probably waaaay too much) information on how the various boot loaders are supposed to play nice with eachother.

[boot-loader-specs]: https://uapi-group.org/specifications/specs/boot_loader_specification/
