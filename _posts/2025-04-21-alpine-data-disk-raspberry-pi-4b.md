---
layout: post
title: "Alpine data disk setup on a Raspberry PI 4B"
description: "A somewhat lengthy guide "
---

## Stuff used

- **Raspberry Pi 4B 4GB rev 1.2**
- A Seagate 4 TB USB 3.0 external HDD
- Alpine 3.21.3 (rpi)
- Kernel 6.12.13 (rpi)

Although the installation media was prepared on a Fedora 41 machine, the same process can be achieved by unzipping with 7-zip the tarball on windows and installing GPT/ext4 related packages (`apk add gptfdisk e2fsprogs`) within Alpine after running `setup-alpine` (see [docs](https://wiki.alpinelinux.org/wiki/Setting_up_disks_manually#Manual_partitioning))

> Consider creating a `setup.md` file with this guide (or part of it) on the bootable media so you can always `vi`/`cat` it on the fly from within Alpine. ([source code](https://github.com/DadiBit/blog.dadib.it/tree/main/_posts))

## Image download

1. Head over to [the official website](https://alpinelinux.org/downloads/) 
2. Click on the first `aarch64` label (should download a tar.gz)
3. Download the relative sha256 checksum and the GPG sign as well
4. Open a terminal and type the following:

```
curl https://alpinelinux.org/keys/ncopa.asc | gpg --import ;
gpg --verify alpine-*.tar.gz.asc
# Ignore warning of not being able to verify that the key belongs to the owner
sha256sum -c alpine-*.tar.gz.sha256
```

5. If verification went smoothly we're ready to create a bootable media

## Disk paritioning

Let's start with an empty, clean disk.

| Size | Filesystem | Mount point | Comment |
| --: | :-: | :-- | :-: |
| 100 MB | FAT32 | /media/boot | Boot (read-only) |
| 300 MB | ext4 | /media/setup | LBU storage + APK cache |
| - MB | ext4 | /var | Data |

> `lbu` can `commit` to a path, however the apkovl backup must be at the root of the partition so it can be discovered at boot.

Here is a list of option you have on how to identify the device (`/dev/sdg`) of the disk:
 - use `lsblk`
 - hot-plug the disk after boot and use `dmesg` to see what the kernel tells you
 - use `df`
 - use the Disks application in GNOME: hot-plug the device and it should pop up in the list of devices

> If you see multiple partitions, like `/dev/sdg2` or `/dev/sdg6` you should double check if it's the correct device

I am going to use `gdisk` to partition my disk, since I want to use the e`x`pert menu to `w`rite the partition layout. After typing `sudo gdisk /dev/sdg`:
```
o # Create new parition table
Y # Confirm

n # Create new partition
<enter> # First partiion
<enter> # Default is fine
+100M # Size
ef00 # EFI system partition

n # Create new partition
<enter> # Second partiion
<enter> # Default is fine
+300M # Size
8300 # Linux filesystem

n # Create new partition
<enter> # Third partiion
<enter> # Default is fine
<enter> # Utilize the whole disk
8310 # Linux /var

x # Expert menu
a # Attributes
1 # First partition
2 # Switch Legacy BIOS bootable attribute
<enter> # Exit from attributes menu

w # Write and exit from gdisk
Y # Confirm
```

You should now have three partitions. Check it by issuing: `sudo blkid /dev/sdg?` (`?` is a wildcard to pick any digit)

Now format the partitions with:
```
# Format the boot partition as FAT32, so we can modify cmdline.txt and usercfg.txt from windows
sudo mkfs.vfat -F 32 /dev/sdg1

# Disable journaling and 64 bit ext4 options "to reduce write operations and allow the disk to
# spin down after the .apkovl and the packages have been read from the partition during the boot"
# See: https://wiki.alpinelinux.org/wiki/Diskless_Mode
sudo mkfs.ext4 -O ^has_journal,^64bit /dev/sdg2

# Variable data needs journaling to make sure that even on power loss we can recover
sudo mkfs.ext4 /dev/sdg3
```
Please be patient with ext4, since it may take a bit of time on big and slow disks (ie, 4TB+ external HDDs)

## Bootable device

> Please note that the following commands use `sudo` since official docs use `--same-owner`, which extracts stuff as if owned by root (uid 0, gid 0)
> Being a FAT32 partition it's fine to omit this option and work as non root (use fuse to mount device).

Mount as superuser the first partition with `sudo mount /dev/sdg1 /mnt`.

Push the mount directory with `pushd /mnt` and issue (after replacing `~/Downloads` with the path where you downloaded the tarball):
```
sudo tar -p -s --atime-preserve --same-owner --one-top-level=$PWD -zxvf ~/Downloads/alpine-*.tar.gz
```

## Create answer file

> Trust me, you want this, you waste a lot less time when experimenting.
> Also, please note that if you want to touch as little as possible the boot partition, consider storing it in the second partition, or on a server!

Create a `setup-answers` file, with you favorite editor within the `/mnt` boot partition mountpoint with the following content:
```
# Use US layout with US-INTL variant
KEYMAPOPTS="us us-intl"

# Set hostname to coffee
HOSTNAMEOPTS="-n coffee"

# Contents of /etc/network/interfaces
INTERFACESOPTS="auto lo
iface lo inet loopback

auto eth0
iface eth0 inet dhcp
    hostname coffee
"

# Set timezone to Europe/Rome
TIMEZONEOPTS="-z Europe/Rome"

# Set http/ftp proxy to none
PROXYOPTS="none"

# Use "official" Alpine CDN, use "-f" to look for fastest mirror
APKREPOSOPTS="-1"

# Prompt non-root user creation

# Install Dropbear
SSHDOPTS="-c dropbear"

# Use chrony
NTPOPTS="-c chrony"

# Perform diskless setup
DISKOPTS="none"
LBUOPTS="none"
APKCACHEOPTS="none"
```

I couldn't get `setup-disk` to work in data mode on the `/dev/sda3` partition, so I had to manually edit fstab (which is good, since I get to customize my install). A working `fstab`:
```
# This is the standard Alpine boot image.
# You may wipe all other partitions, reboot and be able to start from scratch 
/dev/sda1       /media/boot     vfat    defaults,ro 0 0

# This is where LBU stores its backups (LBU_PATH variable)
# This is also where APK caches are stored (reinstalled at boot automatically)
/dev/sda2       /media/setup    ext4    defaults,rw 0 0

# This is user data, intended for containers, network shares etc.
/dev/sda3       /var            ext4    defaults,rw 0 0
```

> We use `0` in the last field, skipping fsck: it's not installed by default at system boot. We can enable it later on with the apk cache.

After saving and closing the file you may now pop the directory with `popd` and unmount the disk with `sudo umount /mnt`.

You may consider adding the `noatime` option on ext4 partitions to prevent access times writing (reducing writes to disk).

## First boot

1. Plug the usb disk in the Raspberry Pi USB 3.0 port
2. Connect the HDMI/micro HDMI cable
3. Plug in usb keyboard
4. Plug in the power plug

> If your keyboard is backlit turn it off or use a different one - the USB hub can't output enough watts to a spinning HDD **and** the rainbow keyboard

## Data disk setup

First of all we need to locate the answers file by using `ls`. It should be either in `/media/usb/` or `/media/sda1/`.

Now issue a `setup-alpine -f /media/sda1/setup-answers` and provide the root password and optionally create a non-root user.

You now have a configured environment with internet synced time and a working package manager.

### Manual disk setup

Override the fstab with:
```
cp /media/sda1/fstab /etc/fstab
# Make sure everyone can read it, but only root can write to it
chown root:root /etc/fstab
chmod 644 /etc/fstab
```

Make sure all media mountpoints exist:
```
mkdir /media/boot
mkdir /media/setup
```

Mount both media devices:
```
mount /dev/sda1
mount /dev/sda2
```

#### Variable Data setup

Add rsync package: `apk add rsync`

Remove apk cache before copying variable data: `rm -rf /var/cache/apk`

Mount and copy the `/var` directory over to the partition:
```
mount /dev/sda3 /mnt
rsync -Pa /var/ /mnt/
umount /mnt
```

Now mount the variable data partition too: `mount /dev/sda3`

#### `apk` setup

Open `/etc/apk/repositories` with `vi`:
1. Change the local repo to `/media/boot/apks` (should be `/media/sda1/apks` or `/media/usb/apks`)
2. Uncomment the community repo (should be the last line).

Create the apk cache directory: `mkdir /media/setup/cache`

Now issue: `setup-apkcache /media/setup/cache`.

#### fsck setup for ext4 partitions

After setting the apk cache you may `apk add e2fsprogs`, which provides `fsck.ext4`.

You may now set the fsck (last) field of the `/var` entry (`/dev/sda3` in my case) within `/etc/fstab` **to 2** (instead of 0, which would skip it)

> Feel free to fsck the `/media/setup` fstab entry too.

#### LBU seutp

Now set `LBU_PATH=/media/setup` within `/etc/lbu/lbu.conf`.

You can finally `lbu commit` and `reboot`.

### Final checks

- If you see your hostname at login then LBU was configured correctly
- If you see fsck stating that `/dev/sda3` is clean at boot the apk cache was configured correctly
- Alternatively, if you see dropbear being loaded at boot the apk cache was configured correctly

Upadte the apk indecies and fetch new package versions:
```
apk update
apk upgrade
```

Do not forget to `lbu commit`.

#### Check non-root admin

Run `doas whoami`, insert your normal user password and make sure it outputs `root`

#### Check non-root ssh login

You can already copy over your key with `ssh-copy-id <user>@<ip>`.

Insert the password and login with `ssh <user>@<ip>`. Do not forget to `lbu commit` after adding the key.

## Final tweaks on boot parition

`poweroff` your raspberry and plug the disk back in your desktop.

Make sure to mount the boot partition (`/dev/sdg1` previously) as **read-write** (default, unless you're working live from your alpine system)

### Update bootable read-only system

> This works for restoring the boot image to its original form too.
> This is useful after messing up big time while tweaking files in it.

Simply wipe the partition as FAT32 and extract the tar contents to the partition root.

### Remove setup helper files

You may either "update" the bootable partition to the same installed version, or simply `rm fstab setup-answers`.

### Tweak cmdline.txt

By default the root tmpfs uses half of your ram. In my case 2 GB is too much, I need just 200 MB: add `rootflags=size=200M` to `cmdline.txt`. 

Since we don't need usb/sd modules discovery we can remove them from the modules list. This also gets rid of `ln: usbdisk: File exists` when mounting to `/media/usb` manually.

Here's the final `cmdline.txt`:
```
modules=loop,squashfs quiet console=tty1 rootflags=size=200M
```

### Get rid of unused files

> A backup of the partition is recommended, although you may always wipe the partition and untar the original tarball.

Here's a list of files you don't need to boot a Raspberry Pi 4B:
- bootcode.bin
- start.elf
- fixup.dat

You may then also remove all `*.dtb` files, excluding `bcm2711-rpi-4-b.dtb`.

Please check the [official docs](https://www.raspberrypi.com/documentation/computers/configuration.html#boot-folder-contents) to know what can be removed.

### Update firmware

You may check which version you're using by installing vcgencmd with `apk add raspberrypi-utils-vcgencmd` and running `vcgencmd version`

Head over to the [official repo latest release](https://github.com/raspberrypi/firmware/releases/latest) and download the tarball.

Replace the following files:
 - `bootcode.bin` (not needed on Raspberry Pi 4/5 variants)
 - `start.elf` (or `start4.elf` on a Raspberry Pi 4 variant)
 - `fixup.dat` (or `fixup4.dat` on a Raspberry Pi 4 variant)

### Switch to cut-down firmware

From [official docs](https://www.raspberrypi.com/documentation/computers/configuration.html#start-elf):
> a cut-down version of the firmware that removes support for hardware blocks such as codecs and 3D as well as debug logging support; it also imposes initial frame buffer limitations. The cut-down firmware is automatically used when gpu_mem=16 is specified in config.txt.

Similarly to previous step copy the `start_cd.elf`+`fixup_cd.dat` (or `start4cd.elf`+`fixup4cd.dat` on a Raspberry Pi 4 variant) to boot directory.

Now set `gpu_mem=16` in config.txt (not usercfg.txt, as it's included from config.txt and it can't set this parameter)

You may remove non-cut-down binary (elf) and linker (dat) files.

# (Optional) Flash USB-favourable eeprom

With the aid of the Raspberry Pi imager utility you can flash the most up-to-date eeprom version, possibly with "USB-over-uSD preference".
 - Poweroff the device
 - Insert just the sd card
 - Power device on and wait for green flashing/green screen

