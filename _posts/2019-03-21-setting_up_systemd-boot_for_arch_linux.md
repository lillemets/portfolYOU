---
title: "Setting up systemd-boot for Arch Linux"
description: "Installing Arch Linux with systemd-boot requires several details to be considered during the installation process. This post gives an overview of necessary steps."
tags: [Linux]
category: lifestyle
---

The [Arch Linux Wiki](https://wiki.archlinux.org/) covers all the necessary details on how to install and maintain a moderately customized Linux system. Installing a bootable Arch Linux system requires that all these details (partition table, file system, kernel parameters, drivers etc) are mutually compatible. Unfortunately, no single Wiki article can cover all the custom configurations and use cases. In order not to mess up installations as occasionally happens, I'm going to outline my approach. The instructions assume that the boot will reside on disk `/dev/sda` and partition `/dev/sda1` and root is installed on `/dev/sda3`. Alter the respective bits of commands if necessary.

First off, it is necessary that the hardware supports UEFI and the installation medium is **written with UEFI support** and **booted in UEFI mode**. To make sure that all this is true, see if UEFI variables are accessible: 

``` {bash}
efivar --list
```

or

``` {bash}
ls /sys/firmware/efi/efivars
```

After sucessfully booting the installation medium, the disk needs to be prepared to host the system. I use `fdisk` for this:

``` {bash}
fdisk /dev/sda
```

The disk must contain a **GPT partition table**. In order to create this, press *g* and confirm. Next, the boot partition type must be set to **EFI System**. This can be done by pressing *t* and selecting the designated boot partition. 

Once the new partition table is written to disk by pressing *w*, a **FAT32 file system** must be created on the boot partition:

``` {bash}
mkfs.vfat -F32 /dev/sda1
```

Now the boot partition must be mounted to suitable location, `/mnt/boot` is fine:

``` {bash}
mkdir /mnt/boot
mount /dev/sda1 /mnt/boot
```

One of the reasons why I prefer `systemd-bood` to alternative bootloaders is that it does not need to be explicitly downloaded and is very easy to install. Just run the following while inside `arch-chroot`:

``` {bash}
bootctl --path=/boot install
```

Another great thing is the minimal configuration of `systemd-boot`. First, create a boot entry in `/boot/loader/entries/arch.conf` (choose whatever name you prefer instead of *arch*):

``` {bash}
title	Arch Linux
linux	/vmlinuz-linux
initrd	/initramfs-linux.img
options	root=/dev/sda3 rw
```

If you're not lazy like me, you'll change the root pointer to indicate partition UUID to be on the safe side.

Finally, set the boot parameters in `/boot/loader/loader.conf` as explained in the [Wiki](https://wiki.archlinux.org/index.php/systemd-boot#Configuration). I keep it simple:

``` {bash}
timeout 3
default arch
```

Any other systems installed alongside Arch Linux can be added to the bootloader by creating additional entries in the `/boot/loader/entries/` directory.

If all was done correctly, the system should now boot into the `systemd-boot`.
