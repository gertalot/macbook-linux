# Linux on a MacBook Pro 2017

Instructions to install Linux on a MacBook Pro 2017 (MacBook14,2). These are just my personal notes; I don't
guarantee anything.

These instructions are fully based on the following sources:

- [install broadcom wifi drivers](https://askubuntu.com/questions/55868/installing-broadcom-wireless-drivers)
- [roadrunner2's gist](https://gist.github.com/roadrunner2/1289542a748d9a104e7baec6a92f9cd7)
- [nixaid's guide](https://nixaid.com/linux-on-macbookpro/)
- [geteltorito](https://github.com/rainer042/geteltorito/) to extract ISO bootimage information
- [epir.at's bootable iso article](https://epir.at/2013/03/10/modify-a-bootable-.iso-image-macos/)
- [using xorriso](https://askubuntu.com/questions/1110651/how-to-produce-an-iso-image-that-boots-only-on-uefi)
- [custom bootable iso with xorriso](https://www.0xf8.org/2020/03/recreating-isos-that-boot-from-both-dvd-and-mass-storage-such-as-usb-sticks-and-in-both-legacy-bios-and-uefi-environments/)

This is working as of December 2024.

## Prerequisites

- a MacBook Pro 2017 15" (MacBook14,2) — it may work on other models, but that's the one I've got
- a USB flash drive
- _maybe_ an ethernet adapter if I can't get it working offline

## Clean Install

I did this on a spare laptop, so started from a clean slate:

- boot the machine holding cmd-option-R for Internet Recovery
- connect to wifi and let the system boot
- open Disk Utility and erase the disk (for good measure I guess)
- Install the OS

After installing, follow the prompts and go to Settings -> General -> Software update and install any updates.

## Partition drive

- Open Disk Utility, click "Partition"
- Reduce the one partion's size to 32Gb. We'll install Linux on the freed up space.

(NOTE: is this actually necessary?)

## Create install media

Download [Ubuntu 24.0.1 ISO from NZ 2degrees mirror](https://mirror.2degrees.nz/ubuntu-releases/24.04.1/ubuntu-24.04.1-desktop-amd64.iso):

```sh
curl -O https://mirror.2degrees.nz/ubuntu-releases/24.04.1/ubuntu-24.04.1-desktop-amd64.iso
```

Download [Etcher](https://etcher.balena.io/).

If you just want to boot up Ubuntu and have an Ethernet adapter or don't care about Internet connectivity, plug in your USB drive, open Etcher, select the ISO and the USB drive, and flash the image to the drive.

If you don't have an Ethernet adapter, we need to do a bit more work, because the Ubuntu installer doesn't install the broadcom wifi drivers necessary for wifi connectivity. So instead of flashing the USB drive with the ISO installer as-is, we're going to create a new iso with one additional file. Read on!

### Install WiFi drivers offline

TL;DR:

```sh
# download the broadcom firmware that works for most hardware
curl -O http://www.lwfinger.com/b43-firmware/broadcom-wl-5.100.138.tar.bz2
```

#### Find out what hardware we have

Shut down your target laptop, plug in the USB drive, press the power button and hold option. You should see an option to boot from the USB drive. When you do, you'll see a menu giving you the option to try or install Ubuntu. Choose that and boot the system.

Once it's started up, open a Terminal, and find your network device:

```sh
lspci -nn -d 14e4:
```

That should show something like:

```txt
03:00.0 Network Controller [0280]: Broadcom Inc. and subsidiaries BCM43602 802.11ac Wireless LAN SoC [14e4:43ba] (rev 02)
```

Note the chip ID `BCM43602`, PCI ID `14e4:43ba`, and the revision `rev 02`. Looking at [the wireless driver table](https://askubuntu.com/questions/55868/installing-broadcom-wireless-drivers),
that corresponds to the driver `firmware-b43-installer/linux-firmware`.

#### Download the firmware

If you have an Internet connection, you can use the `firmware-b43-installer` package.
The installer in that package will download the broadcom firmware and install it. Since
we're installing offline, download the firmware from the same URL ahead of time. We will
add it to the ISO so we can install it later.

```sh
curl -O https://www.lwfinger.com/b43-firmware/broadcom-wl-5.100.138.tar.bz2
```

#### edit iso

We'll mount the installer ISO image, but we need to extract the boot image information
so we can add that when we create a new iso.

Download Ubuntu:

```sh
# Download ubuntu from a NZ mirror (change this if you're not in New Zealand)
curl -O https://mirror.2degrees.nz/ubuntu-releases/24.04.1/ubuntu-24.04.1-desktop-amd64.iso
```

Mount the iso:

```sh
mkdir mnt
hdiutil attach -nomount ubuntu-24.04.1-desktop-amd64.iso
# check what device the above command attaches to. For me it's /dev/disk4
mount -t cd9660 /dev/disk4 mnt
```

Copy the files to a new place because we're going to change a few things:

```sh
mkdir iso
rsync -a mnt/ iso/
chmod -R u+w iso
mkdir iso/pool/main/b/broadcom-wl
cp broadcom-wl-5.100.138.tar.bz2 iso/pool/main/b/broadcom-wl
chmod -R u-w iso
```

Unmount the image:

```sh
# replace with the device you got with the hdiutil attach command above
# If you're not sure, use `hdiutil info` to find out
umount /dev/disk4
hdiutil detach /dev/disk4
```

#### build a new iso

Install the tools we need:

```sh
brew install xorriso
```

Get the boot image info from the original iso:

```sh
xorriso -indev ubuntu-24.04.1-desktop-amd64.iso -report_el_torito as_mkisofs
```

Make a note of the options it shows. For me it's:

```txt
-V 'Ubuntu 24.04.1 LTS amd64'
--modification-date='2024082716232600'
--grub2-mbr --interval:local_fs:0s-15s:zero_mbrpt,zero_gpt:'ubuntu-24.04.1-desktop-amd64.iso'
--protective-msdos-label
-partition_cyl_align off
-partition_offset 16
--mbr-force-bootable
-append_partition 2 28732ac11ff8d211ba4b00a0c93ec93b --interval:local_fs:12105120d-12115263d::'ubuntu-24.04.1-desktop-amd64.iso'
-appended_part_as_gpt
-iso_mbr_part_type a2a0d0ebe5b9334487c068b6b72699c7
-c '/boot.catalog'
-b '/boot/grub/i386-pc/eltorito.img'
-no-emul-boot
-boot-load-size 4
-boot-info-table
--grub2-boot-info
-eltorito-alt-boot
-e '--interval:appended_partition_2_start_3026280s_size_10144d:all::'
-no-emul-boot
-boot-load-size 10144
```

Now create a new iso; use the options as reported above:

```sh
xorriso -as mkisofs \
    -o ubuntu-24.04.1-desktop-amd64-custom.iso \
    -V 'Ubuntu 24.04.1 LTS amd64' \
    --modification-date='2024082716232600' \
    --grub2-mbr --interval:local_fs:0s-15s:zero_mbrpt,zero_gpt:'ubuntu-24.04.1-desktop-amd64.iso' \
    --protective-msdos-label \
    -partition_cyl_align off \
    -partition_offset 16 \
    --mbr-force-bootable \
    -append_partition 2 28732ac11ff8d211ba4b00a0c93ec93b --interval:local_fs:12105120d-12115263d::'ubuntu-24.04.1-desktop-amd64.iso' \
    -appended_part_as_gpt \
    -iso_mbr_part_type a2a0d0ebe5b9334487c068b6b72699c7 \
    -c '/boot.catalog' \
    -b '/boot/grub/i386-pc/eltorito.img' \
    -no-emul-boot \
    -boot-load-size 4 \
    -boot-info-table \
    --grub2-boot-info \
    -eltorito-alt-boot \
    -e '--interval:appended_partition_2_start_3026280s_size_10144d:all::' \
    -no-emul-boot \
    -boot-load-size 10144 \
    iso/
```

#### flash the USB drive with the ISO

Open Etcher, select the newly created ISO and the USB drive, and flash.

Voila! That gives you a bootable Ubuntu installer with the broadcom firmware so
we can install it offline.

### Install Ubuntu

Plug the USB drive into the target laptop, press the power button and hold option until you see bootable drive icons.
Select the "EFI boot" image and wait for Ubuntu to load up.

Once loaded, let's get some wifi! Open a terminal and:

```sh
# install the tool that can install the firmware
sudo dpkg -i /cdrom/pool/main/b/b43-fwcutter/b43-fwcutter_1%3a019-11build1_amd64.deb
# install the firmware
tar xjvf /cdrom/pool/main/b/broadcom-wl/broadcom-wl-5.100.138.tar.bz2
sudo b43-fwcutter -w /lib/firmware broadcom-wl-5.100.138/linux/wl_apsta.o
sudo modprobe b43
```

Set the transmit power:

```sh
# for the actual interface name, do iwconfig. For me it's wlp3s0
sudo iwconfig wlp3s0 txpower 10dBm
```

Open settings, turn wifi off/on, and connect to the network you want. It should work now.

With WiFi operational, follow the steps to install Ubuntu.
