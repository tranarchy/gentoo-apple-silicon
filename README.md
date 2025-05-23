# Gentoo Linux for Apple Sillicon

As of writing this guide there is no official documentation for installing Gentoo on Apple Sillicon

There is an older [guide](https://wiki.gentoo.org/wiki/User:Jared/Gentoo_On_An_M1_Mac) by Jared that is flagged as "no longer required in most cases" and now it points to a [new guide](https://asahilinux.org/docs/alt/installing-gentoo/) that doesn't even exists

So I have decided to document my Gentoo installation process that doesn't require installing Fedora or using [asahi-gentoosupport](https://github.com/chadmed/asahi-gentoosupport/tree/main)

> [!NOTE]
> If you are planning on using Steam read [this](#steam) before proceeding

## UEFI environment

From macOS run the following in the terminal

```
curl https://alx.sh | sh
```

Alternatively you can read the code before running it

 ```
curl https://alx.sh > alx.sh
cat alx.sh
sh ./alx.sh
 ```

During the install process choose "UEFI environment only"

> [!WARNING]
> Follow the Asahi installer instructions carefully

## LiveCD

As of writing this guide Gentoo does not offer an official aarch64 LiveCD that boots on Apple Sillicon\
We are going to use the official Void Linux apple sillicon LiveCD that you can download [here](https://voidlinux.org/download/#arm%20platforms)

After downloading the LiveCD write it to an USB with `dd`

```
dd if=void-live-asahi-XXXXXXXX-base.iso of=/dev/sdX
```

## Booting from USB

Boot from the USB in the Asahi UEFI environment

If it doesn't boot the USB by default you can try the following

```
run bootcmd_usb0
```

## Live environment

Once you booted into the LiveCD we can start the installation process

### Partitioning

The Asahi installer script previously allocated free space on your disk, we are going to use this free space for our root partition

```
cfdisk /dev/nvme0n1
```

Select the free space and create a new partition, write the changes then quit

Now you can format the new partition with whatever filesystem you want

```
mkfs.xfs /dev/nvme0n1pX
```

> [!CAUTION]
> Do **NOT** touch any other partition
>
> Deleting or formatting the wrong partition can render your system unusable

### Internet

Use `ip a` to get the name of your network interface

```
wpa_passphrase <SSID> <PASSWORD> > wpa.conf
wpa_supplicant -i <NETWORK_INTERFACE> -c wpa.conf &

dhcpcd
```

Confirm that internet works with `ping gentoo.org`

### Acquiring the necessary tools

We will need `xz` and `wget` to download and extract the stage3 tarball\
The Void live environment doesn't come with these tools by default so we are going to install them

```
xbps-install -Su
xbps-install xz wget
```

You could install `links` instead of `wget` if you want

### Stage3

Create the `/mnt/gentoo` directory and mount your root partition

```
mkdir /mnt/gentoo
mount /dev/nvme0n1pX /mnt/gentoo
cd /mnt/gentoo
```

Download the desired arm64 stage3 tarball

```
wget https://distfiles.gentoo.org/releases/...
```

Extract it to `/mnt/gentoo`

```
tar xpvf stage3-*.tar.xz --xattrs-include='*.*' --numeric-owner -C /mnt/gentoo
```

## make.conf

```
vi /mnt/gentoo/etc/portage/make.conf
```

Modify your make.conf to your liking but make sure you apply the following changes

```
COMMON_FLAGS="-march=armv8.5-a+fp16+simd+crypto+i8mm -mtune=native -O2 -pipe"
```

```
RUSTFLAGS="-C target-cpu=native"
```

```
VIDEO_CARDS="asahi"
```

If you use GRUB

```
GRUB_PLATFORMS="efi-64"
```

For M2 chips use `armv8.6-a`

## Chroot

Now chroot into the Gentoo install, Void comes with `xchroot` so we are going to use that

```
xchroot /mnt/gentoo /bin/bash
```

Now we are going to mount our boot partition, we are going to use the `vfat` partition that the Asahi installer created

```
mount /dev/nvme0n1pX /boot
```

> [!IMPORTANT]
> From now on you can follow the [handbook](https://wiki.gentoo.org/wiki/Handbook:AMD64) as you would on an amd64 machine
>
> However stop at the "Configuring the Linux kernel" section

## Asahi packages

As of writing this guide Gentoo offers [some](https://packages.gentoo.org/packages/search?q=asahi) pacakges that are required for Apple Sillicon to work, but not everything is included in the official repos

We are going to use the `asahi-overlay`, that is [endorsed](https://wiki.gentoo.org/wiki/Project:Asahi) by Gentoo to get everything we need

```
emerge eselect-repository
eselect repository enable asahi
emerge --sync

emerge asahi-meta
```

Huge thanks to chadmed for creating [asahi-overlay](https://github.com/chadmed/asahi-overlay)

## Kernel and bootloader

### Kernel

Apple Sillicon needs `linux-firwmare` to work so emerge it

```
emerge linux-firmware
```

We are going to use [dracut](https://wiki.gentoo.org/wiki/Dracut) for generating our initramfs

```
emerge dracut
```

We will also need [installkernel](https://wiki.gentoo.org/wiki/Installkernel)

Make sure you enable the `dracut` USE flag for it

```
emerge installkernel
```

The `asahi-meta` package we previously installed already pulled in `asahi-sources`

```
eselect kernel list
eselect kernel set x
```

```
cd /usr/src/linux
```

You can start configuring the kernel with 

```
make rustavailable
make menuconfig
``` 

You can also use the [.config](https://raw.githubusercontent.com/void-linux/void-packages/refs/heads/master/srcpkgs/linux-asahi/files/arm64-dotconfig) from Void Linux as a base

> [!WARNING]
> `make rustavailable` enables rust specific kernel options
>
> This is crucial on Apple Sillicon since multiple drivers for it are written in rust
>
> You _can_ boot without rust enabled but you won't get GPU acceleration or audio for an example

```
make -j$(nproc)
make modules_install
make install
```

### Apple firwmare and m1n1

Upgrade the firmware and m1n1

```
asahi-fwupdate
update-m1n1
```

> [!WARNING]
> Run `update-m1n1` whenever `m1n1`, `u-boot` or `asahi-sources` updates

### Bootloader

We will use GRUB as our bootloader

```
emerge grub
grub-install --boot-directory=/boot/ --efi-directory=/boot/ --removable
grub-mkconfig -o /boot/grub/grub.cfg
```

> [!NOTE]
> Now you can return to the [handbook](https://wiki.gentoo.org/wiki/Handbook:AMD64/Installation/System) and finish your install

## Post install

### Compiling amd64 only packages

You might want to emerge a package that is only available for amd64

You can try this by unmasking the `amd64` or `~amd64` keyword for it and for it's amd64 only dependencies

```
vi /etc/portage/package.accept_keywords
```

```
gui-wm/hyprland ~amd64
other-hyprland/dependency ~amd64
```

It may or may not not compile, but I got [hyprland](https://packages.gentoo.org/packages/gui-wm/hyprland) to work this way so it's worth a try

### Steam

You can install Steam from the `asahi-overlay`, it works by running in a micorVM via [muvm](https://github.com/AsahiLinux/muvm/) and [FEX](https://github.com/FEX-Emu/FEX)

```
emerge steam
```

Run `steam-aarch64` to launch Steam

Currently the `RootFS` for `FEX` is mounted via systemd files, I tried to manually mount them on an OpenRC install however it complained about `libc.so.6` missing, [someone else](https://github.com/chadmed/asahi-overlay/issues/138) experienced the same issue so right now it's probably best to use systemd if you are planning on using Steam. 






