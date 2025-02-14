# Install Linux on Asus Zephyrus G14 Along with Windows

## Background

<!-- I simply use Linux as my primary workstation for daily things, including work. Windows is for gaming _only_. I tried Debian, Ubuntu and ArchLinux and eventually found **Manjaro** can automatically detect and smoothly install on this hardware.  -->

I primarily use Linux for my everyday activities, including work-related tasks. My Windows system is reserved exclusively for gaming. After experimenting with various distributions such as Debian, Ubuntu, and ArchLinux, I discovered that **Manjaro** offers an effortless setup process, automatically detecting and installing seamlessly on my hardware.

## Hardware Spec

- 32G memory
- Nvidia RTX 4090 16G ram
- 1T SSD

## Secure Boot

- Power on with holding `ESC`, then enter BIOS
- BIOS > Boot > Secure Boot > disable, then power off
- BIOS > Advanced (F7) > Cloud Recovery

## EFI Partition

<!-- Microsoft Windows requires an EFI partition for booting, which should exist already. In case if such partition has been deleted, you _have to_ recreate one by using either Linux bootable USB or other utility. The partition _must be_ set to

- EFI
- bootable as flag
- 300MB+

As long as Cloud Recovery is completed successfully, Windows will create its own partition for booting. -->

Windows operating systems necessitate an `EFI` partition for booting, which should already be present. However, if this partition has been accidentally removed, it needs to be recreated using a Linux bootable USB or another suitable tool. This partition must be designated as `EFI bootable` and should have a size of at least 300MB. Once the Cloud Recovery process is successfully completed, Windows will autonomously generate its own partition for booting.

## Shrink the Disk

<!-- I leave **350GB** of disk space for Windows, including considering to install Red Dead Redemption 2. -->

I allocate 350GB of disk space for Windows, taking into account the potential installation of Red Dead Redemption 2.

## Installing Linux

<!-- Boot from a Linux Installer on USB. For modern Linux system, you need _only_ partitions that

- Create a **new** partition for `/etc/efi`, 300MB with boot flagged. 
- Leave all remaining space for `/`. I personally use `xfs` which is super-faster than `ext4`. -->

To install a modern Linux system, boot from a Linux installer on a USB drive. You'll need to ensure the following:

- Create a new partition for `/etc/efi`, sized at 300MB and marked as bootable.
- Allocate the rest of the space for `/`. Personally, I recommend using `xfs` due to its superior performance compared to `ext4`.


## Encryption

After completing the partitioning, consider encrypting your system. For me, encrypting just the `/` partition suffices. It's important to remember not to encrypt any Windows partitions, as doing so could prevent the system from booting correctly.

